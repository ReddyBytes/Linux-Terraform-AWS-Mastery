# Terraform Variables — Making Your Code Reusable

## The Recipe Book Analogy

A great cookbook does not say "buy exactly 3 chicken breasts from the Farm Fresh store on Main Street." It says "use N chicken breasts." You fill in N based on how many people you are cooking for. The recipe is the same — the inputs change.

Terraform variables work the same way. Instead of hardcoding every value in your configuration (region, instance type, bucket name), you define variables. When you apply the configuration, you provide the values. The same Terraform code can deploy to dev with small instances or production with large instances — you just change the variable values.

---

## The Variable Block

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment: dev, staging, or prod"
  default     = "dev"
}
```

The four keys in a variable block:

```
┌──────────────────────────────────────────────────────────────────┐
│                  VARIABLE BLOCK COMPONENTS                       │
│                                                                  │
│  variable "name" {                                               │
│    type        = string       ← what kind of value to accept    │
│    description = "..."        ← documentation for your team     │
│    default     = "dev"        ← value if none provided          │
│    sensitive   = false        ← hide value from logs/output     │
│    validation { ... }         ← custom validation rules         │
│  }                                                               │
│                                                                  │
│  Only the variable name is required.                             │
│  Everything else is optional but recommended.                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Variable Types

```hcl
# Primitive types
variable "region"     { type = string }
variable "port"       { type = number }
variable "encrypted"  { type = bool   }

# Collection types
variable "availability_zones" { type = list(string) }
variable "allowed_ports"      { type = set(number)  }
variable "instance_types"     { type = map(string)  }

# Complex types
variable "database" {
  type = object({
    engine   = string
    version  = string
    port     = number
    multi_az = bool
  })
}
```

Using variables:
```hcl
resource "aws_instance" "web" {
  instance_type = var.instance_type    # reference with var.<name>
  ami           = var.ami_id
}
```

---

## Variable Defaults and Required Variables

```hcl
# Variable WITH a default — optional to provide
variable "instance_type" {
  type    = string
  default = "t3.micro"   # used if no value provided
}

# Variable WITHOUT a default — REQUIRED — Terraform will error if not provided
variable "db_password" {
  type        = string
  description = "Database admin password"
  sensitive   = true
  # No default = must always be provided
}
```

---

## Input Validation

You can validate variable values before Terraform does anything:

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type = string

  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Only t3 instance types are allowed (cost control)."
  }
}

variable "vpc_cidr" {
  type = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block, e.g. 10.0.0.0/16."
  }
}
```

---

## How to Provide Variable Values

There are five ways to provide values — in order of precedence (later overrides earlier):

```
┌─────────────────────────────────────────────────────────────────────┐
│            VARIABLE PRECEDENCE (lowest to highest)                  │
│                                                                     │
│  1. Default values in variable blocks                               │
│  2. terraform.tfvars (auto-loaded if present)                       │
│  3. *.auto.tfvars files (auto-loaded alphabetically)                │
│  4. TF_VAR_name environment variables                               │
│  5. -var and -var-file flags on the CLI  ← highest priority         │
└─────────────────────────────────────────────────────────────────────┘
```

### Method 1: Default Values (Already Shown Above)

### Method 2: terraform.tfvars File

Create a file named `terraform.tfvars` in your working directory — Terraform loads it automatically:

```hcl
# terraform.tfvars
environment   = "production"
instance_type = "t3.large"
region        = "us-east-1"

availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

instance_types = {
  web     = "t3.large"
  worker  = "t3.medium"
  bastion = "t3.micro"
}
```

### Method 3: Custom .tfvars Files

Use named var files for different environments:

```hcl
# environments/dev.tfvars
environment   = "dev"
instance_type = "t3.micro"
db_size       = "db.t3.micro"

# environments/prod.tfvars
environment   = "prod"
instance_type = "t3.large"
db_size       = "db.r5.large"
```

Apply with:
```bash
terraform apply -var-file="environments/dev.tfvars"
terraform apply -var-file="environments/prod.tfvars"
```

### Method 4: Environment Variables

Prefix any variable name with `TF_VAR_`:

```bash
export TF_VAR_environment="staging"
export TF_VAR_db_password="SuperSecretPassword123"
export TF_VAR_instance_type="t3.small"

terraform apply   # picks up TF_VAR_* automatically
```

This is great for CI/CD pipelines where you set secrets as environment variables.

### Method 5: CLI Flags

Pass values directly on the command line:

```bash
terraform apply -var="environment=prod" -var="instance_type=t3.large"
terraform plan  -var-file="prod.tfvars" -var="region=us-west-2"
```

---

## Sensitive Variables

Mark a variable as sensitive to prevent its value from appearing in logs, plan output, or state file diff output:

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

variable "api_key" {
  type      = string
  sensitive = true
}
```

When `sensitive = true`:
```bash
terraform plan
# Outputs: db_password = (sensitive value)
# Terraform will not show the actual value
```

**Important:** Sensitive values ARE still stored in the state file in plaintext. Always use encrypted remote state (S3 with encryption) when handling sensitive variables.

---

## Practical Example: Fully Parameterized EC2

```hcl
# variables.tf
variable "project" {
  type        = string
  description = "Project name, used in resource naming"
}

variable "environment" {
  type        = string
  description = "dev, staging, or prod"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "key_pair_name" {
  type        = string
  description = "Name of an existing EC2 Key Pair for SSH access"
}

# main.tf
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  key_name      = var.key_pair_name

  tags = {
    Name        = "${var.project}-${var.environment}-app"
    Project     = var.project
    Environment = var.environment
  }
}

# terraform.tfvars (development)
project       = "myapp"
environment   = "dev"
instance_type = "t3.micro"
key_pair_name = "my-dev-key"
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  variable "name" { type, description, default, sensitive }      │
│  Reference with: var.variable_name                              │
│  No default = required; must be provided at runtime             │
│  validation { condition, error_message } = custom rules         │
│                                                                 │
│  Ways to provide values (highest wins):                         │
│  1. Default in variable block                                   │
│  2. terraform.tfvars (auto-loaded)                              │
│  3. *.auto.tfvars (auto-loaded)                                 │
│  4. TF_VAR_name environment variables                           │
│  5. -var="name=value" or -var-file=file.tfvars (CLI flags)      │
│                                                                 │
│  sensitive = true hides values from output but NOT state file   │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Data Sources](../03_providers_resources/data_sources.md) &nbsp;|&nbsp; **Next:** [Outputs →](./outputs.md)

**Related Topics:** [Outputs](./outputs.md) · [Locals](./locals.md) · [Security Best Practices](../09_best_practices/security.md)
