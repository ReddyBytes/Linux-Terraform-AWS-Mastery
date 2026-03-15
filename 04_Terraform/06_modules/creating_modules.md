# Creating Terraform Modules — Writing Reusable Infrastructure Code

## The LEGO Kit Analogy

LEGO sells specialized kits: a castle kit, a space kit, a city kit. Each kit contains specific pieces assembled into a pattern that can be reused. You do not rebuild from scratch every time — you use the kit.

Terraform modules are those kits. A module is a reusable package of Terraform resources. Once you write a module for a VPC, you can call that module in dev, staging, and prod — passing different variables each time. Same code, different configurations.

---

## What is a Module?

In Terraform's terminology:
- **Root module**: The directory where you run `terraform apply`. Your `main.tf`, `variables.tf`, etc.
- **Child module**: A subdirectory called by the root module with a `module` block. Gets its own set of resources, variables, and outputs.

```
┌──────────────────────────────────────────────────────────────────┐
│                    MODULE STRUCTURE                              │
│                                                                  │
│  my-project/                    ← root module                   │
│  ├── main.tf                    ← calls child modules            │
│  ├── variables.tf               ← root variables                 │
│  ├── outputs.tf                 ← root outputs                   │
│  ├── versions.tf                ← provider/terraform versions    │
│  └── modules/                                                   │
│      ├── networking/            ← child module: VPC/subnets      │
│      │   ├── main.tf                                            │
│      │   ├── variables.tf                                       │
│      │   └── outputs.tf                                         │
│      └── webserver/             ← child module: EC2 + SG         │
│          ├── main.tf                                            │
│          ├── variables.tf                                       │
│          └── outputs.tf                                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## Standard Module File Structure

Every well-structured module has exactly these three files:

```
modules/networking/
├── main.tf        ← the resources
├── variables.tf   ← input variables (what the caller provides)
└── outputs.tf     ← output values (what the caller can use)
```

Optional files:
```
├── versions.tf    ← provider requirements (for standalone modules)
├── README.md      ← documentation
└── examples/      ← usage examples
```

---

## Writing a Module: The networking Example

### modules/networking/variables.tf

```hcl
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "environment" {
  type        = string
  description = "Environment name (dev, staging, prod)"
}

variable "project" {
  type        = string
  description = "Project name for resource naming and tagging"
}

variable "availability_zones" {
  type        = list(string)
  description = "List of AZs to create subnets in"
}

variable "public_subnet_cidrs" {
  type        = list(string)
  description = "CIDR blocks for public subnets (one per AZ)"
}

variable "private_subnet_cidrs" {
  type        = list(string)
  description = "CIDR blocks for private subnets (one per AZ)"
}
```

### modules/networking/main.tf

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = merge(local.common_tags, { Name = "${local.name_prefix}-igw" })
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${var.availability_zones[count.index]}"
    Type = "public"
  })
}

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-${var.availability_zones[count.index]}"
    Type = "private"
  })
}
```

### modules/networking/outputs.tf

```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "The CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "internet_gateway_id" {
  description = "ID of the Internet Gateway"
  value       = aws_internet_gateway.main.id
}
```

---

## Calling a Module from the Root

### main.tf (root module)

```hcl
module "networking" {
  source = "./modules/networking"   # path to the module directory

  # Pass values to the module's variables
  project     = var.project
  environment = var.environment
  vpc_cidr    = "10.0.0.0/16"

  availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
}

# Now use the module's outputs
resource "aws_instance" "app" {
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.micro"
  subnet_id     = module.networking.public_subnet_ids[0]

  vpc_security_group_ids = [aws_security_group.app.id]
}

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = module.networking.vpc_id   # from module output
}
```

---

## Module Syntax Reference

```hcl
module "module_name" {
  source = "./path/to/module"  # local path
  # OR
  source = "git::https://github.com/org/repo.git//modules/networking"
  # OR
  source = "hashicorp/consul/aws"  # Terraform Registry

  version = "~> 2.0"  # only for registry or git-tagged modules

  # Variables = the module's input variables
  variable_name = value
  another_var   = another_value

  # Meta-arguments work in module blocks too
  count    = 2
  for_each = var.environments
  depends_on = [resource.something.else]

  # Providers (if module needs aliased providers)
  providers = {
    aws = aws.west
  }
}
```

---

## Calling the Same Module Multiple Times

```hcl
# Call the same networking module for dev and prod
module "networking_dev" {
  source      = "./modules/networking"
  project     = "myapp"
  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
  # ...
}

module "networking_prod" {
  source      = "./modules/networking"
  project     = "myapp"
  environment = "prod"
  vpc_cidr    = "10.1.0.0/16"
  # ...
}
```

---

## After Adding/Changing Modules: terraform get

When you add a new module source or change the source path, run:

```bash
terraform init   # always works (re-downloads everything)
# or
terraform get    # only updates modules (faster)
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  Module = reusable package of Terraform resources               │
│  Root module = where you run terraform apply                    │
│  Child module = subdirectory called with module block           │
│                                                                 │
│  Standard module structure:                                     │
│    main.tf      → the resources                                 │
│    variables.tf → inputs (what callers provide)                 │
│    outputs.tf   → outputs (what callers can use)                │
│                                                                 │
│  module "name" { source = "./path", var = value }               │
│  Access outputs: module.name.output_name                        │
│  terraform init or terraform get to download modules            │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← State Commands](../05_state_management/state_commands.md) &nbsp;|&nbsp; **Next:** [Module Registry →](./module_registry.md)

**Related Topics:** [Module Registry](./module_registry.md) · [Module Composition](./module_composition.md) · [Outputs](../04_variables_outputs/outputs.md)
