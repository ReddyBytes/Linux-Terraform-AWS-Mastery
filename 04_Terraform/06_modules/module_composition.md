# Module Composition — Building Large Infrastructure from Modules

## The City Blueprint Analogy

A city is not planned as one giant drawing. Urban planners divide the city into zones: residential, commercial, industrial, parks. Each zone has its own detailed blueprint, designed by specialists. The overall city plan references these zone blueprints and shows how they connect — streets link zones, utilities flow between them.

Terraform module composition is the same. Large infrastructure is split into domain-specific modules (networking, compute, database, security). A root module orchestrates them all, passing outputs from one module as inputs to another.

---

## Root Module vs Child Modules

```
┌──────────────────────────────────────────────────────────────────┐
│                 MODULE HIERARCHY                                 │
│                                                                  │
│  ROOT MODULE (where you run terraform apply)                     │
│  main.tf                                                         │
│  ├── module "networking" {                                       │
│  │       source = "./modules/networking"    ─► Child Module 1    │
│  │   }                           ┌──────────── vpc_id output     │
│  │   vpc_id = module.networking.vpc_id       ◄───────────────── │
│  │                                                               │
│  ├── module "security_groups" {                                  │
│  │       source = "./modules/security_groups"  ─► Child Module 2 │
│  │       vpc_id = module.networking.vpc_id  ← uses networking    │
│  │   }                                                           │
│  │                                                               │
│  └── module "compute" {                                          │
│          source = "./modules/compute"        ─► Child Module 3   │
│          subnet_ids = module.networking.private_subnet_ids       │
│          sg_id      = module.security_groups.web_sg_id           │
│      }                                  ↑ chains outputs         │
└──────────────────────────────────────────────────────────────────┘
```

---

## Passing Outputs Between Modules

The key pattern: **a child module's output becomes another module's input**.

```hcl
# main.tf — Root module orchestration

# Layer 1: Networking (no dependencies)
module "networking" {
  source = "./modules/networking"

  project     = var.project
  environment = var.environment
  vpc_cidr    = "10.0.0.0/16"
  azs         = ["us-east-1a", "us-east-1b"]
}

# Layer 2: Security Groups (depends on networking VPC ID)
module "security_groups" {
  source = "./modules/security_groups"

  project     = var.project
  environment = var.environment
  vpc_id      = module.networking.vpc_id   # ← from networking module
}

# Layer 3: Compute (depends on networking subnets + security groups)
module "webservers" {
  source = "./modules/webservers"

  project            = var.project
  environment        = var.environment
  subnet_ids         = module.networking.public_subnet_ids    # ← from networking
  security_group_ids = [module.security_groups.web_sg_id]    # ← from security_groups
  instance_type      = var.instance_type
}

# Layer 4: Database (depends on networking private subnets + security groups)
module "database" {
  source = "./modules/database"

  project            = var.project
  environment        = var.environment
  subnet_ids         = module.networking.private_subnet_ids   # ← from networking
  security_group_ids = [module.security_groups.db_sg_id]     # ← from security_groups
  instance_class     = var.db_instance_class
}

# Root outputs aggregate module outputs
output "app_server_ips"  { value = module.webservers.public_ips }
output "db_endpoint"     { value = module.database.endpoint }
output "vpc_id"          { value = module.networking.vpc_id }
```

---

## A Multi-Module Project Structure

```
my-infrastructure/
├── main.tf            ← root: calls modules, wires them together
├── variables.tf       ← root: environment-level inputs
├── outputs.tf         ← root: expose final outputs
├── versions.tf        ← Terraform and provider versions
├── terraform.tfvars   ← variable values
│
└── modules/
    ├── networking/
    │   ├── main.tf        → VPC, subnets, IGW, NAT, route tables
    │   ├── variables.tf
    │   └── outputs.tf     → vpc_id, public_subnet_ids, private_subnet_ids
    │
    ├── security_groups/
    │   ├── main.tf        → web SG, app SG, db SG, bastion SG
    │   ├── variables.tf   → vpc_id
    │   └── outputs.tf     → web_sg_id, app_sg_id, db_sg_id
    │
    ├── webservers/
    │   ├── main.tf        → ASG, launch template, ALB, target group
    │   ├── variables.tf   → subnet_ids, sg_ids, instance_type
    │   └── outputs.tf     → alb_dns_name, alb_arn
    │
    ├── database/
    │   ├── main.tf        → RDS instance, subnet group, parameter group
    │   ├── variables.tf   → subnet_ids, sg_ids, instance_class
    │   └── outputs.tf     → endpoint, address, port
    │
    └── iam/
        ├── main.tf        → IAM roles, policies, instance profiles
        ├── variables.tf
        └── outputs.tf     → role_arn, instance_profile_name
```

---

## Using for_each with Modules

Create multiple instances of a module:

```hcl
variable "environments" {
  default = {
    dev = {
      instance_type  = "t3.micro"
      db_class       = "db.t3.micro"
      vpc_cidr       = "10.0.0.0/16"
    }
    prod = {
      instance_type  = "t3.large"
      db_class       = "db.r5.large"
      vpc_cidr       = "10.1.0.0/16"
    }
  }
}

module "environment" {
  source   = "./modules/full-stack"
  for_each = var.environments

  environment   = each.key                   # "dev" or "prod"
  instance_type = each.value.instance_type
  db_class      = each.value.db_class
  vpc_cidr      = each.value.vpc_cidr
  project       = var.project
}
```

---

## Monorepo vs Separate Repos

```
┌──────────────────────────────────────────────────────────────────┐
│           MONOREPO vs SEPARATE REPOS                             │
├─────────────────────────┬────────────────────────────────────────┤
│  Monorepo               │  Separate Repos                        │
│  (all modules + roots   │  (modules in their own repos)          │
│   in one Git repo)      │                                        │
├─────────────────────────┼────────────────────────────────────────┤
│  + Easy to refactor      │ + Each module versioned independently  │
│  + Single PR for         │ + Teams can own their modules         │
│    cross-module changes  │ + Cleaner release process             │
│  + Simpler CI/CD         │ + Used by published registry modules  │
│                          │                                        │
│  - Module versions not   │ - Cross-module changes need           │
│    independently tracked │   coordinated PRs                     │
│  - One team may break    │ - More complex CI/CD                  │
│    another's module      │ - Module consumers must pin versions  │
│                          │                                        │
│  RECOMMENDED FOR:        │ RECOMMENDED FOR:                      │
│  Small/medium teams      │ Large orgs, platform teams            │
│  Single product          │ Multiple product teams                │
└─────────────────────────┴────────────────────────────────────────┘
```

---

## Module Nesting — Keep It Shallow

Deeply nested modules are hard to follow and debug:

```
# AVOID: deep nesting is confusing
module "infra" {
  module "layer1" {
    module "layer2" {
      module "layer3" { ... }   ← avoid 3+ levels deep
    }
  }
}

# PREFER: flat structure with clear wiring in root
module "networking"      { source = "..." }
module "security"        { source = "...", vpc_id = module.networking.vpc_id }
module "compute"         { source = "...", subnet_ids = module.networking.private_subnet_ids }
module "database"        { source = "...", subnet_ids = module.networking.private_subnet_ids }
```

**Rule of thumb:** Maximum 2 levels of nesting in most cases (root → module, no deeper).

---

## Complete Root Module Example

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "myapp/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = { ManagedBy = "Terraform", Project = var.project }
  }
}

# main.tf
module "networking" {
  source      = "./modules/networking"
  project     = var.project
  environment = var.environment
  vpc_cidr    = var.vpc_cidr
  azs         = var.availability_zones
}

module "compute" {
  source      = "./modules/compute"
  project     = var.project
  environment = var.environment
  vpc_id      = module.networking.vpc_id
  subnet_ids  = module.networking.public_subnet_ids
}

# outputs.tf
output "app_url"    { value = "http://${module.compute.alb_dns_name}" }
output "vpc_id"     { value = module.networking.vpc_id }
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  Module composition = chain outputs from one module into        │
│  inputs of another module                                       │
│                                                                 │
│  Root module = orchestrator, wires modules together             │
│  Child modules = domain experts (networking, compute, db)       │
│                                                                 │
│  Pattern:                                                       │
│  module "networking" → output vpc_id                            │
│  module "compute" { vpc_id = module.networking.vpc_id }         │
│                                                                 │
│  Use for_each on modules to create per-environment stacks       │
│  Monorepo: good for small teams                                 │
│  Separate repos: good for large orgs with module ownership      │
│  Keep nesting shallow: root → module (avoid 3+ levels)          │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Module Registry](./module_registry.md) &nbsp;|&nbsp; **Next:** [Workspaces →](../07_workspaces/workspaces.md)

**Related Topics:** [Creating Modules](./creating_modules.md) · [Module Registry](./module_registry.md) · [Remote State](../05_state_management/remote_state.md)
