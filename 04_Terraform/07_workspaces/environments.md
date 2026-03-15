# Environment Management Strategies — Dev, Staging, and Prod

## The Branch Office Analogy

A company might run its New York, London, and Tokyo offices identically (same policies, same tools, same org chart) — or they might need different structures due to local law, headcount, or business focus. How you manage the offices depends on how similar they need to be.

Terraform environment management is the same choice. If dev and prod are nearly identical, workspaces work fine. If they have meaningfully different configurations, different AWS accounts, or different teams owning them — separate directories or repos are the right answer.

There is no single correct answer. This guide explains the three main strategies and when to use each.

---

## The Three Strategies

```
┌──────────────────────────────────────────────────────────────────┐
│            ENVIRONMENT MANAGEMENT STRATEGIES                     │
│                                                                  │
│  Strategy 1: Workspaces                                          │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  one codebase → workspace: dev / staging / prod            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Strategy 2: Separate Directories                               │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  environments/                                             │  │
│  │  ├── dev/       (dev-specific main.tf + tfvars)            │  │
│  │  ├── staging/   (staging-specific)                         │  │
│  │  └── prod/      (prod-specific)                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Strategy 3: Separate Repositories                              │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  github.com/myorg/infra-dev    (dev)                       │  │
│  │  github.com/myorg/infra-prod   (prod)                      │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Strategy 1: Workspaces

Covered in depth in [workspaces.md](./workspaces.md). Summary here:

```bash
terraform workspace new dev
terraform workspace select dev
terraform apply -var-file="envs/dev.tfvars"

terraform workspace select prod
terraform apply -var-file="envs/prod.tfvars"
```

**Pros:**
- Single codebase — changes automatically apply to all environments
- Easy to create ephemeral environments (feature branches)
- Simple CI/CD: select workspace, apply

**Cons:**
- All environments in one AWS account (same credentials, same isolation level)
- Easy to accidentally apply to wrong workspace
- If envs diverge in architecture, conditionals get messy

**Best for:** Small teams, single AWS account, structurally identical environments.

---

## Strategy 2: Separate Directories (Recommended for Most Teams)

Each environment gets its own directory. They share modules from a common `modules/` directory.

```
my-infrastructure/
├── modules/               ← shared, reusable modules
│   ├── networking/
│   ├── compute/
│   └── database/
│
├── environments/
│   ├── dev/
│   │   ├── main.tf        ← calls modules, dev-specific
│   │   ├── variables.tf
│   │   ├── terraform.tfvars  ← dev values
│   │   ├── outputs.tf
│   │   └── backend.tf     ← dev state bucket + key
│   │
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars  ← staging values
│   │   ├── outputs.tf
│   │   └── backend.tf
│   │
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars  ← prod values
│       ├── outputs.tf
│       └── backend.tf     ← prod state bucket + key (different account!)
```

### Example: environments/dev/main.tf

```hcl
# Calls the same modules as prod, but with different variable values

module "networking" {
  source      = "../../modules/networking"
  project     = var.project
  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
  azs         = ["us-east-1a", "us-east-1b"]
}

module "compute" {
  source        = "../../modules/compute"
  project       = var.project
  environment   = "dev"
  vpc_id        = module.networking.vpc_id
  subnet_ids    = module.networking.public_subnet_ids
  instance_type = "t3.micro"
  instance_count = 1
}
```

### Example: environments/prod/main.tf

```hcl
# Same modules, different values
module "networking" {
  source      = "../../modules/networking"
  project     = var.project
  environment = "prod"
  vpc_cidr    = "10.1.0.0/16"
  azs         = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

module "compute" {
  source         = "../../modules/compute"
  project        = var.project
  environment    = "prod"
  vpc_id         = module.networking.vpc_id
  subnet_ids     = module.networking.private_subnet_ids
  instance_type  = "t3.large"
  instance_count = 3
}
```

### Workflow

```bash
# Work on dev
cd environments/dev/
terraform init
terraform plan
terraform apply

# Promote to prod (separately, intentionally)
cd environments/prod/
terraform init
terraform plan    # carefully review
terraform apply
```

**Pros:**
- Environments can have different AWS accounts (different backend configs)
- Changes to one env do not automatically affect others — promotion is deliberate
- Clear blast radius — working in `prod/` vs `dev/` is obvious
- Teams can own specific environment directories

**Cons:**
- Risk of copy-paste drift if environments diverge in ways not captured in modules
- More directories to navigate

**Best for:** Most teams. The sweet spot of isolation + code reuse via shared modules.

---

## Strategy 3: Separate Repositories

Each environment (or sometimes each service) has its own Git repository.

```
github.com/myorg/infra-shared-modules   ← shared modules (published)
github.com/myorg/infra-dev              ← dev environment
github.com/myorg/infra-staging          ← staging environment
github.com/myorg/infra-prod             ← prod environment
```

```hcl
# In infra-prod/main.tf — references tagged versions of shared modules
module "networking" {
  source  = "git::https://github.com/myorg/infra-shared-modules.git//networking?ref=v2.1.0"
}
```

**Pros:**
- Maximum isolation — each repo has its own access controls, CI/CD, history
- Modules versioned independently with Git tags
- Team A owns prod repo, Team B owns dev repo

**Cons:**
- Most complex — cross-environment changes require coordinating multiple repos
- Module updates require PRs to each consuming repo
- Overkill for most small/medium teams

**Best for:** Large organizations with separate teams owning different environments, strict security requirements, or compliance requirements separating prod access.

---

## Comparison Matrix

```
┌───────────────────────┬───────────────┬───────────────┬───────────────┐
│  Factor               │  Workspaces   │  Directories  │  Repos        │
├───────────────────────┼───────────────┼───────────────┼───────────────┤
│  Multi-AWS account    │      NO       │     YES       │     YES       │
│  Code reuse           │    HIGH       │    HIGH       │   MEDIUM      │
│  Isolation            │     LOW       │    HIGH       │   HIGHEST     │
│  Accidental apply risk│    HIGH       │     LOW       │   LOWEST      │
│  Setup complexity     │     LOW       │    MEDIUM     │    HIGH       │
│  CI/CD complexity     │    MEDIUM     │    MEDIUM     │    HIGH       │
│  Team size fit        │   1-5 devs    │  5-50 devs    │   50+ devs    │
│  Recommended?         │  Sometimes    │   Usually     │  Large orgs   │
└───────────────────────┴───────────────┴───────────────┴───────────────┘
```

---

## Recommended Pattern for Small Teams (1-10 Engineers)

```
my-infrastructure/
├── modules/          ← reusable, environment-agnostic
├── environments/
│   ├── dev/
│   └── prod/
└── .github/
    └── workflows/
        ├── dev.yml   ← auto-apply to dev on merge to main
        └── prod.yml  ← manual approve for prod
```

## Recommended Pattern for Large Teams (10+ Engineers)

```
github.com/myorg/
├── terraform-modules         ← shared modules, versioned with tags
├── infra-networking           ← VPC/networking (platform team owns)
├── infra-compute              ← compute layer (app team owns)
└── infra-databases            ← databases (data team owns)
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  3 strategies: workspaces / directories / separate repos        │
│                                                                 │
│  Workspaces:                                                    │
│    + simple, one codebase                                       │
│    - all in one account, easy to apply to wrong env             │
│    Best: tiny teams, identical envs, single AWS account         │
│                                                                 │
│  Separate directories (RECOMMENDED for most):                   │
│    + clear isolation, different accounts possible               │
│    + shared modules prevent drift                               │
│    Best: most teams with 2-50 engineers                         │
│                                                                 │
│  Separate repos:                                                │
│    + maximum isolation, team ownership                          │
│    - complex coordination                                        │
│    Best: large orgs, strict compliance requirements             │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Workspaces](./workspaces.md) &nbsp;|&nbsp; **Next:** [VPC with Terraform →](../08_aws_with_terraform/vpc.md)

**Related Topics:** [Workspaces](./workspaces.md) · [Module Composition](../06_modules/module_composition.md) · [CI/CD Integration](../09_best_practices/ci_cd_integration.md)
