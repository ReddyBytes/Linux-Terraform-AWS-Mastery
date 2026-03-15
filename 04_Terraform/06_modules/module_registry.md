# The Terraform Module Registry — Pre-Built Infrastructure Components

## The App Store Analogy

When you need a flashlight app on your phone, you do not write one from scratch. You go to the App Store, search "flashlight," pick a well-reviewed one, and install it. Thousands of apps exist for common needs — you only build from scratch when what you need does not exist.

The **Terraform Registry** is the App Store for infrastructure modules. Thousands of pre-built, community-tested modules exist for common AWS patterns: VPCs, EKS clusters, RDS databases, S3 buckets, IAM roles, and much more. You reference them with a short source string and Terraform downloads them automatically.

---

## What is the Terraform Registry?

The public Terraform Registry lives at https://registry.terraform.io. It hosts:

1. **Providers** — plugins for AWS, Azure, GCP, GitHub, Datadog, etc.
2. **Modules** — pre-built collections of resources for common patterns

```
┌──────────────────────────────────────────────────────────────────┐
│                 TERRAFORM REGISTRY                               │
│                                                                  │
│  registry.terraform.io                                           │
│  ├── Providers/                                                  │
│  │   ├── hashicorp/aws        ← official, 1000+ resources        │
│  │   ├── hashicorp/azurerm                                       │
│  │   └── hashicorp/google                                        │
│  └── Modules/                                                    │
│      ├── terraform-aws-modules/vpc/aws      ← Official AWS VPC   │
│      ├── terraform-aws-modules/eks/aws      ← EKS cluster        │
│      ├── terraform-aws-modules/rds/aws      ← RDS database       │
│      ├── terraform-aws-modules/s3-bucket/aws ← S3 bucket         │
│      └── thousands more...                                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## Registry Module Source Format

```hcl
module "vpc" {
  source  = "<namespace>/<module_name>/<provider>"
  version = "<version_constraint>"
  # ...
}

# Examples:
source = "terraform-aws-modules/vpc/aws"      # AWS VPC
source = "terraform-aws-modules/eks/aws"      # EKS
source = "terraform-aws-modules/rds/aws"      # RDS
source = "terraform-aws-modules/s3-bucket/aws" # S3
```

---

## The Official terraform-aws-modules Collection

The `terraform-aws-modules` namespace is the most trusted AWS module collection, maintained by Anton Babenko and the community. It is battle-tested and used by thousands of companies.

### VPC Module

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false   # one per AZ for HA
  enable_dns_hostnames   = true
  enable_dns_support     = true

  # Tags for subnets (required for EKS)
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }

  tags = {
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}

# Use the module's outputs
output "vpc_id"             { value = module.vpc.vpc_id }
output "private_subnet_ids" { value = module.vpc.private_subnets }
output "public_subnet_ids"  { value = module.vpc.public_subnets }
```

### RDS Module

```hcl
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "myapp-postgres"

  engine            = "postgres"
  engine_version    = "14"
  instance_class    = "db.t3.medium"
  allocated_storage = 20

  db_name  = "myappdb"
  username = "dbadmin"
  # Password is auto-generated and stored in state
  manage_master_user_password = true

  vpc_security_group_ids = [module.rds_sg.security_group_id]
  db_subnet_group_name   = module.rds.db_subnet_group_name

  # Subnet group
  subnet_ids = module.vpc.private_subnets

  # Parameter group
  family = "postgres14"

  # Enhanced monitoring
  monitoring_interval    = 60
  monitoring_role_name   = "MyRDSMonitoringRole"
  create_monitoring_role = true

  tags = {
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}
```

### S3 Bucket Module

```hcl
module "s3_bucket" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "~> 3.0"

  bucket = "my-app-assets-prod"
  acl    = "private"

  versioning = {
    enabled = true
  }

  server_side_encryption_configuration = {
    rule = {
      apply_server_side_encryption_by_default = {
        sse_algorithm = "AES256"
      }
    }
  }

  # Block all public access
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true

  tags = {
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}
```

---

## version Pinning — Critically Important

Always pin a version when using registry modules. Without pinning, `terraform init` fetches the latest version, which may have breaking changes:

```hcl
# DANGEROUS: no version = gets latest on each terraform init
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # ...
}

# CORRECT: pinned to minor/patch range
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"   # allows 5.x but not 6.x
  # ...
}

# MOST LOCKED DOWN: exact version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "= 5.5.0"
  # ...
}
```

---

## Using Git Sources

You can source modules from a Git repository — useful for private internal modules:

```hcl
# From GitHub (public)
module "networking" {
  source = "git::https://github.com/myorg/terraform-modules.git//networking"
}

# With a specific tag
module "networking" {
  source = "git::https://github.com/myorg/terraform-modules.git//networking?ref=v1.2.0"
}

# From a private repo using SSH
module "networking" {
  source = "git::ssh://git@github.com/myorg/terraform-private-modules.git//networking?ref=v1.0.0"
}

# From S3 (useful in air-gapped environments)
module "networking" {
  source = "s3::https://s3.amazonaws.com/mybucket/networking-module.zip"
}
```

---

## Official vs Community Modules — How to Choose

```
┌──────────────────────────────────────────────────────────────────┐
│           EVALUATING A REGISTRY MODULE                           │
│                                                                  │
│  1. Downloads / week — high numbers = widely used                │
│  2. Last updated — recent = actively maintained                  │
│  3. GitHub stars — community trust signal                        │
│  4. Issues / PRs — signs of health                               │
│  5. README quality — good docs = thoughtful authors              │
│  6. Examples directory — shows intended usage                    │
│                                                                  │
│  Prefer: terraform-aws-modules/* for AWS — gold standard         │
│  Prefer: hashicorp/* for multi-cloud patterns                    │
│  Caution: unmaintained modules (no updates in 2+ years)          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Registry Module vs Custom Module — When to Write Your Own

```
Use a Registry Module when:                Write Your Own when:
───────────────────────────────────        ──────────────────────────────
A well-maintained module exists            No suitable module exists
The module's opinionated defaults          You need highly specific logic
  match your use case                      Module's interface is too complex
You want community bug fixes               Company security/compliance
  automatically (via version bumps)          requires custom behavior
Speed matters (faster to implement)        You are building a module for
                                             internal reuse by your team
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  registry.terraform.io = the "App Store" for Terraform modules  │
│  source = "namespace/module/provider" for registry modules      │
│  Always pin: version = "~> 5.0"                                 │
│                                                                 │
│  Key AWS registry modules (terraform-aws-modules):              │
│    vpc/aws           → VPC with subnets, NAT, IGW               │
│    rds/aws           → RDS instances with best practices         │
│    eks/aws           → EKS cluster                               │
│    s3-bucket/aws     → S3 with encryption + access blocking      │
│                                                                 │
│  Git source: git::https://github.com/org/repo.git//path?ref=tag │
│  Evaluate modules: downloads, last update, README, examples     │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Creating Modules](./creating_modules.md) &nbsp;|&nbsp; **Next:** [Module Composition →](./module_composition.md)

**Related Topics:** [Creating Modules](./creating_modules.md) · [Module Composition](./module_composition.md) · [VPC with Terraform](../08_aws_with_terraform/vpc.md)
