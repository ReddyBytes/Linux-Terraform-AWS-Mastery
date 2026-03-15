# Terraform Locals — Eliminating Repetition in Your Code

## The Nickname Analogy

Imagine your company's full legal name is "Northeastern Advanced Computing and Cloud Infrastructure Services LLC." In every email, document, and conversation, you refer to it as "NACCS." You defined that abbreviation once, and everyone uses the short form. If the company name changes, you update the abbreviation in one place.

Terraform `locals` work exactly this way. You define a value (or a computed expression) once, give it a short name, and use that name throughout your configuration. When the value needs to change, you update it in one place.

---

## The locals Block

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = "platform-team"
  }
}
```

Reference locals with `local.<name>` (singular, not plural):
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}
```

---

## Why Use Locals?

### Problem: Repeated Expressions

Without locals — notice the repetition:

```hcl
resource "aws_s3_bucket" "app_data" {
  bucket = "${var.project}-${var.environment}-data"
  tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket" "app_logs" {
  bucket = "${var.project}-${var.environment}-logs"
  tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_instance" "web" {
  # ...
  tags = {
    Name        = "${var.project}-${var.environment}-web"
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

With locals — define once, use everywhere:

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket" "app_data" {
  bucket = "${local.name_prefix}-data"
  tags   = local.common_tags
}

resource "aws_s3_bucket" "app_logs" {
  bucket = "${local.name_prefix}-logs"
  tags   = local.common_tags
}

resource "aws_instance" "web" {
  # ...
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-web" })
}
```

---

## Complex Local Computations

Locals can contain complex expressions, not just simple strings:

```hcl
locals {
  # Boolean computed from variable
  is_production = var.environment == "prod"

  # Conditional values
  instance_type = local.is_production ? "t3.large" : "t3.micro"
  db_size       = local.is_production ? "db.r5.large" : "db.t3.micro"
  multi_az      = local.is_production ? true : false

  # Map derived from variable list
  az_to_cidr = {
    for idx, az in var.availability_zones :
    az => cidrsubnet(var.vpc_cidr, 8, idx)
  }
  # Input:  ["us-east-1a", "us-east-1b"]  cidr: "10.0.0.0/16"
  # Output: {"us-east-1a" = "10.0.0.0/24", "us-east-1b" = "10.0.1.0/24"}

  # List transformation
  public_subnet_cidrs = [
    for idx in range(length(var.availability_zones)) :
    cidrsubnet(var.vpc_cidr, 8, idx)
  ]

  private_subnet_cidrs = [
    for idx in range(length(var.availability_zones)) :
    cidrsubnet(var.vpc_cidr, 8, idx + 100)
  ]

  # Computed ARN prefix
  iam_arn_prefix = "arn:aws:iam::${data.aws_caller_identity.current.account_id}"
}
```

---

## Locals vs Variables — When to Use Each

```
┌──────────────────────────────────────────────────────────────────┐
│              locals vs variables                                 │
├───────────────────────────┬──────────────────────────────────────┤
│  variables                │  locals                              │
├───────────────────────────┼──────────────────────────────────────┤
│  Input from outside       │  Computed inside your config         │
│  (passed in at runtime)   │  (never set by user)                 │
│                           │                                      │
│  User provides value via  │  You define the value in your code   │
│  .tfvars, env, CLI        │                                      │
│                           │                                      │
│  Can have a default       │  Always has an expression            │
│                           │                                      │
│  Used when value changes  │  Used to avoid repeating complex     │
│  per environment/user     │  expressions throughout your code    │
│                           │                                      │
│  Example: region          │  Example: name_prefix derived        │
│  Example: instance_type   │  from project + environment vars     │
│  Example: team_name       │  Example: is_production bool         │
└───────────────────────────┴──────────────────────────────────────┘
```

---

## Locals Referencing Other Locals

Locals can reference each other — Terraform resolves them in dependency order:

```hcl
locals {
  # Build up values step by step
  account_id  = data.aws_caller_identity.current.account_id
  region      = data.aws_region.current.name

  name_prefix = "${var.project}-${var.environment}"
  full_prefix = "${local.name_prefix}-${local.region}"

  # S3 bucket names must be globally unique — include account ID
  bucket_name = "${local.full_prefix}-${local.account_id}"

  # Centralized tag set
  base_tags = {
    Project     = var.project
    Environment = var.environment
    Region      = local.region
    ManagedBy   = "Terraform"
  }
}
```

---

## Naming Conventions for Locals

```hcl
locals {
  # Use snake_case for local names
  name_prefix  = "good_name"
  namePrEfIx   = "bad_name"     # avoid camelCase

  # Group related locals together
  # Tags
  common_tags  = { ... }
  ec2_tags     = merge(local.common_tags, { ... })

  # Names
  vpc_name     = "${local.name_prefix}-vpc"
  subnet_name  = "${local.name_prefix}-subnet"

  # Computed booleans: prefix with is_ or enable_
  is_production   = var.environment == "prod"
  enable_backups  = local.is_production

  # Computed sizes/counts: be descriptive
  nat_gateway_count = local.is_production ? 3 : 1
}
```

---

## Real-World Complete Example

```hcl
# locals.tf

locals {
  # Identity
  name_prefix = "${var.project}-${var.environment}"

  # Environment flags
  is_production = var.environment == "prod"

  # Sizing
  ec2_instance_type = local.is_production ? "t3.large" : "t3.micro"
  rds_instance_class = local.is_production ? "db.r5.large" : "db.t3.micro"
  rds_multi_az       = local.is_production

  # Tagging
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
    CostCenter  = var.cost_center
  }

  # Network
  public_subnet_cidrs  = [for i in range(3) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i in range(3) : cidrsubnet(var.vpc_cidr, 8, i + 10)]
}

# main.tf — clean resource definitions using locals
resource "aws_instance" "app" {
  ami           = data.aws_ami.al2023.id
  instance_type = local.ec2_instance_type   # t3.large or t3.micro

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-app"
  })
}

resource "aws_db_instance" "main" {
  engine         = "postgres"
  instance_class = local.rds_instance_class
  multi_az       = local.rds_multi_az

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-db"
  })
}
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  locals { name = expression }  — define reusable computed values│
│  Reference with: local.name  (singular "local")                 │
│                                                                 │
│  Use locals to:                                                 │
│  - Avoid repeating the same expression multiple times           │
│  - Build complex derived values from variables                  │
│  - Create a central tagging strategy                            │
│  - Compute conditional sizes/settings                           │
│                                                                 │
│  locals vs variables:                                           │
│  variables = input from outside (user/pipeline provides value)  │
│  locals    = computed internally (you define the expression)    │
│                                                                 │
│  locals can reference other locals                              │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Outputs](./outputs.md) &nbsp;|&nbsp; **Next:** [State File →](../05_state_management/state_file.md)

**Related Topics:** [Variables](./variables.md) · [Outputs](./outputs.md) · [Expressions](../02_hcl_basics/expressions.md)
