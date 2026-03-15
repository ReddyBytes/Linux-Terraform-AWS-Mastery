# Code Organization — Structuring Terraform Projects for Maintainability

## The Library Analogy

A well-organized library has a clear system: fiction is separate from non-fiction, books are alphabetized within sections, reference books are in a special area. A first-time visitor can find any book without asking for help because the organization is consistent and predictable.

Your Terraform project should work the same way. A new team member should be able to navigate your codebase, understand what each file does, and find what they need — without a guided tour.

---

## The Standard File Structure

Every Terraform module (including your root module) should follow this file structure:

```
project/
├── main.tf          ← resource blocks go here
├── variables.tf     ← all variable declarations
├── outputs.tf       ← all output declarations
├── versions.tf      ← terraform{} block, required_providers
├── data.tf          ← data source blocks (optional, can be in main.tf)
├── locals.tf        ← locals block (optional, can be in main.tf)
└── terraform.tfvars ← variable values (do NOT commit if it has secrets!)
```

**Why separate files instead of one big `main.tf`?**

Because when a new engineer asks "what variables does this module accept?", they look in `variables.tf`. When they ask "what does this module output?", they look in `outputs.tf`. The file name is documentation.

---

## What Goes in Each File

### versions.tf — Provider and Terraform version constraints

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "myapp/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.region

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Project     = var.project
      Environment = var.environment
    }
  }
}
```

### variables.tf — All input variables

```hcl
# variables.tf
variable "project" {
  type        = string
  description = "Project name used in all resource names and tags"
}

variable "environment" {
  type        = string
  description = "Deployment environment: dev, staging, prod"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "region" {
  type    = string
  default = "us-east-1"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}
```

### main.tf — Resource blocks and module calls

```hcl
# main.tf
locals {
  name_prefix = "${var.project}-${var.environment}"
}

module "networking" {
  source      = "./modules/networking"
  project     = var.project
  environment = var.environment
  vpc_cidr    = var.vpc_cidr
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.al2023.id
  instance_type = var.instance_type
  subnet_id     = module.networking.public_subnet_ids[0]
  tags          = { Name = "${local.name_prefix}-web" }
}
```

---

## Naming Conventions

Consistent naming makes your code readable and your AWS console navigable:

```
┌──────────────────────────────────────────────────────────────────┐
│                    NAMING CONVENTIONS                            │
│                                                                  │
│  Resources:  use snake_case, descriptive names                   │
│  ✓  aws_instance.web_server                                      │
│  ✓  aws_security_group.web_lb_sg                                 │
│  ✗  aws_instance.Instance1                                       │
│  ✗  aws_security_group.sg1                                       │
│                                                                  │
│  Variables:  snake_case, describe what they hold                 │
│  ✓  var.instance_type                                            │
│  ✓  var.vpc_cidr_block                                           │
│  ✗  var.instanceType  (no camelCase)                             │
│  ✗  var.x  (no abbreviations that aren't universally known)      │
│                                                                  │
│  AWS resource Names (the Name tag):                              │
│  ✓  "${var.project}-${var.environment}-web-server"               │
│  ✓  "myapp-prod-web-server"  (readable in AWS console)          │
│  ✗  "web-server-1-new-2"  (meaningless context)                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## Tagging Strategy

Tags are your most important operational tool. They enable cost allocation, security auditing, automation, and incident response:

```hcl
# Minimum required tags for every resource
locals {
  mandatory_tags = {
    Project     = var.project       # which project owns this?
    Environment = var.environment   # dev / staging / prod
    ManagedBy   = "Terraform"       # don't touch manually!
    Owner       = var.team          # which team owns this?
  }

  # Use default_tags in the provider block instead of repeating in every resource
}

provider "aws" {
  region = var.region

  default_tags {
    tags = local.mandatory_tags
  }
}

# Then add resource-specific tags where needed
resource "aws_instance" "web" {
  # ...
  tags = {
    Name        = "${var.project}-${var.environment}-web"
    # default_tags adds: Project, Environment, ManagedBy, Owner
    # Additional resource-specific tags:
    Role        = "web-server"
    CostCenter  = "eng-web-team"
  }
}
```

---

## Folder Structure for Large Projects

```
organization-infrastructure/
│
├── modules/                         ← reusable internal modules
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── web-cluster/
│   └── database/
│
├── environments/
│   ├── dev/
│   │   ├── main.tf                  ← calls modules with dev vars
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   ├── outputs.tf
│   │   └── backend.tf
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
│
├── global/                          ← resources shared across envs
│   ├── iam/                         ← org-wide IAM roles/policies
│   └── dns/                         ← Route53 hosted zones
│
└── bootstrap/                       ← one-time setup: state bucket, etc.
    ├── main.tf
    └── README.md
```

---

## File Organization Rules

```
┌──────────────────────────────────────────────────────────────────┐
│                 PRACTICAL RULES                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Rule 1: One resource TYPE per file for large resource blocks    │
│  networking.tf, compute.tf, database.tf, monitoring.tf           │
│                                                                  │
│  Rule 2: Never put variable values in main.tf                    │
│  All var.* values go in terraform.tfvars or -var-file            │
│                                                                  │
│  Rule 3: Keep main.tf declarative, not procedural               │
│  It should read like a list of "I want X, Y, Z" not instructions│
│                                                                  │
│  Rule 4: Every module must have a README.md                      │
│  Document: purpose, inputs, outputs, examples, requirements      │
│                                                                  │
│  Rule 5: Run terraform fmt on every commit                       │
│  Consistent formatting = easier diffs, easier reviews            │
│                                                                  │
│  Rule 6: Run terraform validate in CI                            │
│  Catches syntax errors before plan/apply                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## Useful Commands for Code Quality

```bash
# Format all .tf files in current directory and subdirectories
terraform fmt -recursive

# Check formatting without changing files (for CI)
terraform fmt -check -recursive

# Validate configuration syntax
terraform validate

# Generate dependency graph (requires graphviz)
terraform graph | dot -Tpng > graph.png

# Show current configuration as JSON
terraform show -json | jq .
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  Standard files: main.tf, variables.tf, outputs.tf, versions.tf │
│  Each file has a clear purpose — follow it consistently          │
│                                                                 │
│  Naming: snake_case, descriptive, project-env-type pattern       │
│  Tags: mandatory set via default_tags in provider block          │
│  Tag at minimum: Project, Environment, ManagedBy, Owner          │
│                                                                 │
│  Folder structure for teams:                                    │
│    modules/ → reusable code                                     │
│    environments/dev|staging|prod/ → per-env root modules        │
│    global/ → shared resources                                   │
│    bootstrap/ → one-time setup                                  │
│                                                                 │
│  terraform fmt -recursive → enforce consistent style             │
│  terraform validate       → catch syntax errors early           │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← RDS](../08_aws_with_terraform/rds.md) &nbsp;|&nbsp; **Next:** [Security →](./security.md)

**Related Topics:** [Security](./security.md) · [CI/CD Integration](./ci_cd_integration.md) · [Creating Modules](../06_modules/creating_modules.md)
