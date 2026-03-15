# Terraform Workspaces — One Codebase, Multiple Environments

## The Office Template Analogy

A company has one standard office layout blueprint. When they open offices in New York, London, and Tokyo, they do not draw three different blueprints — they use the same blueprint three times, adjusting only for local requirements (number of desks, local fire code, available floor space).

Terraform workspaces work similarly. You have one set of `.tf` files. Each workspace has its own separate state file. You switch between workspaces to operate on different environments (dev, staging, prod) using the same code.

---

## What Problem Do Workspaces Solve?

Without workspaces, managing multiple environments requires either:
- Duplicating your entire Terraform directory for each environment (risky — drift between copies)
- Using one directory with complex conditionals based on a variable

Workspaces provide a third option: the same code, different state, toggled by a workspace name.

```
┌──────────────────────────────────────────────────────────────────┐
│                  WORKSPACES AND STATE                            │
│                                                                  │
│  same .tf files  ─────────►  workspace: default                 │
│                             state: terraform.tfstate            │
│                                                                  │
│                  ─────────►  workspace: dev                     │
│                             state: terraform.tfstate.d/dev/     │
│                                                                  │
│                  ─────────►  workspace: staging                 │
│                             state: terraform.tfstate.d/staging/ │
│                                                                  │
│                  ─────────►  workspace: prod                    │
│                             state: terraform.tfstate.d/prod/    │
└──────────────────────────────────────────────────────────────────┘
```

Each workspace has its own independent state. Destroying resources in `dev` has zero effect on `prod`.

---

## Workspace Commands

```bash
# List all workspaces (* = current active workspace)
terraform workspace list
# * default
#   dev
#   staging
#   prod

# Create a new workspace
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch to a workspace
terraform workspace select dev
terraform workspace select prod

# Show the current workspace
terraform workspace show
# → dev

# Delete a workspace (must not be current; state must be empty)
terraform workspace delete dev
```

---

## Using ${terraform.workspace} in Code

The current workspace name is available as the expression `terraform.workspace`. Use it to differentiate resource configurations per environment:

```hcl
# Vary instance type by workspace
resource "aws_instance" "web" {
  ami           = data.aws_ami.al2023.id
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"

  tags = {
    Name        = "web-${terraform.workspace}"
    Environment = terraform.workspace
  }
}

# Vary resource count by workspace
resource "aws_instance" "web" {
  count         = terraform.workspace == "prod" ? 3 : 1
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${terraform.workspace}-${count.index + 1}"
  }
}

# Use a lookup map for cleaner workspace-specific config
locals {
  workspace_config = {
    default = {
      instance_type = "t3.micro"
      db_class      = "db.t3.micro"
      count         = 1
    }
    dev = {
      instance_type = "t3.micro"
      db_class      = "db.t3.micro"
      count         = 1
    }
    staging = {
      instance_type = "t3.small"
      db_class      = "db.t3.small"
      count         = 1
    }
    prod = {
      instance_type = "t3.large"
      db_class      = "db.r5.large"
      count         = 3
    }
  }

  config = local.workspace_config[terraform.workspace]
}

resource "aws_instance" "app" {
  count         = local.config.count
  ami           = data.aws_ami.al2023.id
  instance_type = local.config.instance_type
}
```

---

## Workspace Workflow

```bash
# Create and apply a dev environment
terraform workspace new dev
terraform workspace select dev
terraform apply -var-file="environments/dev.tfvars"

# Create and apply a prod environment
terraform workspace new prod
terraform workspace select prod
terraform apply -var-file="environments/prod.tfvars"

# Make a change to dev
terraform workspace select dev
# edit your .tf files
terraform plan
terraform apply

# Promote to prod (same code, different workspace)
terraform workspace select prod
terraform plan    # IMPORTANT: review carefully before applying to prod!
terraform apply
```

---

## Remote State + Workspaces

With the S3 backend, each workspace automatically gets its own state path:

```
# With backend key = "myapp/terraform.tfstate"
S3 bucket structure:
├── myapp/terraform.tfstate          ← "default" workspace
├── env:/dev/myapp/terraform.tfstate ← "dev" workspace
├── env:/staging/myapp/terraform.tfstate ← "staging" workspace
└── env:/prod/myapp/terraform.tfstate    ← "prod" workspace
```

---

## When NOT to Use Workspaces

Workspaces are often misunderstood. Here are cases where they are the WRONG solution:

```
┌──────────────────────────────────────────────────────────────────┐
│                 WORKSPACE LIMITATIONS                            │
│                                                                  │
│  1. Same backend, same permissions                               │
│     All workspaces share the same AWS credentials.               │
│     You cannot isolate prod in a different AWS account           │
│     using workspaces alone.                                      │
│                                                                  │
│  2. Same code = same structure                                   │
│     If prod needs fundamentally different architecture           │
│     than dev, workspaces become messy conditionals.              │
│     Separate directories are cleaner.                            │
│                                                                  │
│  3. Easy to accidentally apply to wrong environment              │
│     You must remember to select the right workspace.             │
│     A mistake in prod is a bad day.                              │
│                                                                  │
│  GOOD FIT:                                                       │
│  - Feature branches (ephemeral environments)                     │
│  - dev/staging that are structurally identical                   │
│  - Small teams, single AWS account                               │
│                                                                  │
│  BAD FIT:                                                        │
│  - Multiple AWS accounts per environment                         │
│  - Prod needs very different architecture                        │
│  - Large teams needing strict isolation                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  Workspaces = one codebase, multiple independent state files     │
│  terraform workspace new <name>    → create workspace           │
│  terraform workspace select <name> → switch workspace           │
│  terraform workspace list          → list all workspaces        │
│  terraform.workspace               → current workspace name     │
│                                                                 │
│  Use terraform.workspace in code for environment-specific config│
│  Better: lookup map { dev = {...}, prod = {...} } over ternaries│
│                                                                 │
│  Not for: separate AWS accounts, very different architectures   │
│  Good for: identical-structure envs, ephemeral feature envs     │
│  Alternative: separate directories (see environments.md)        │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Module Composition](../06_modules/module_composition.md) &nbsp;|&nbsp; **Next:** [Environments →](./environments.md)

**Related Topics:** [Environments](./environments.md) · [Remote State](../05_state_management/remote_state.md) · [Variables](../04_variables_outputs/variables.md)
