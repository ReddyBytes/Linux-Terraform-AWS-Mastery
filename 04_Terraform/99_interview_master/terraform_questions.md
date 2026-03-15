# Terraform Interview Questions — From Beginner to Advanced

## How to Use This Guide

These questions are organized by level. Work through beginner questions first — they are the foundation. Then advance to intermediate and expert levels. For each question, try to answer it yourself before reading the answer.

---

## Beginner Questions

### Q1: What is Terraform and what problem does it solve?

**Answer:** Terraform is an open-source Infrastructure as Code (IaC) tool by HashiCorp. It solves the problem of manual, inconsistent, and hard-to-reproduce cloud infrastructure management. Instead of clicking through the AWS console and forgetting what you configured, you write HCL files that describe your desired infrastructure. Terraform then provisions it consistently, every time.

---

### Q2: What is the difference between terraform plan and terraform apply?

**Answer:**
- `terraform plan`: A **dry run**. Reads your code and state, compares to real-world, and shows exactly what changes will be made — without making any changes. Think of it as a preview.
- `terraform apply`: Actually **executes** the changes. By default, it shows the same plan output and asks for confirmation before proceeding.

Rule of thumb: always review the plan before applying, especially in production.

---

### Q3: What is the terraform state file and why does it exist?

**Answer:** `terraform.tfstate` is a JSON file that maps your HCL resource blocks to real-world infrastructure IDs and attributes. Terraform needs it to:
1. Track what it has already created (so it doesn't create duplicates)
2. Detect differences between your code and existing infrastructure (for planning)
3. Store computed attributes (like EC2 public IP) that you can reference and output

Without state, Terraform cannot know what infrastructure it manages.

---

### Q4: What is a Terraform provider?

**Answer:** A provider is a plugin that allows Terraform to interact with a specific API or cloud platform. For example, the AWS provider knows how to call the AWS EC2 API to create instances, the S3 API to create buckets, etc. Providers are declared with `required_providers` in the `terraform {}` block and downloaded by `terraform init`.

---

### Q5: What is a resource in Terraform?

**Answer:** A resource block declares a piece of real infrastructure that Terraform should create and manage. Example:
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```
The first label (`aws_instance`) is the resource type (defined by the provider), the second (`web`) is a local name you use to reference it in your code.

---

### Q6: What does terraform init do?

**Answer:** `terraform init` initializes a Terraform working directory. It:
1. Downloads required provider plugins (e.g., `hashicorp/aws`)
2. Downloads any remote modules referenced in `source`
3. Configures the backend (where state is stored)
4. Creates `.terraform.lock.hcl` to pin provider versions

Must be run before any other Terraform command, and again when you add new providers or modules.

---

### Q7: What is the difference between a variable and a local in Terraform?

**Answer:**
- `variable`: An **input** — value comes from outside (`.tfvars` file, CLI flag, environment variable, or user prompt). Can change between runs.
- `local`: An **internal computation** — the value is defined in your code as an expression. Cannot be set by users. Used to avoid repeating complex expressions throughout your config.

Rule: if someone calling or using your module needs to control the value, use a variable. If it's an internal derived value, use a local.

---

### Q8: How do you reference another resource's attribute in Terraform?

**Answer:** Using the resource address: `<resource_type>.<local_name>.<attribute>`. Example:
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id   # references the VPC's ID
}
```
Terraform automatically detects this dependency and creates the VPC before the subnet.

---

### Q9: How do you output a value from Terraform?

**Answer:**
```hcl
output "web_server_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of the web server"
}
```
After `terraform apply`, outputs are displayed in the terminal. Access them later with `terraform output web_server_ip` or `terraform output -raw web_server_ip` (without quotes, for scripting).

---

### Q10: What is HCL?

**Answer:** HCL (HashiCorp Configuration Language) is the declarative language used to write Terraform configurations. It is designed to be human-readable while also being machine-parseable. It uses a block structure (`resource "type" "name" { key = value }`) and supports expressions, functions, references, and type constraints.

---

## Intermediate Questions

### Q11: What is a Terraform module?

**Answer:** A module is a reusable collection of Terraform resources grouped in a directory. You call it with a `module` block, passing input variables, and it returns outputs. The directory where you run `terraform apply` is the root module. All others are child modules. Modules prevent code duplication — write once, use in dev/staging/prod with different variable values.

---

### Q12: What is the difference between count and for_each?

**Answer:**
- `count = N`: Creates N copies of a resource, identified by numeric index `[0]`, `[1]`, etc. Problem: removing the middle element renumbers subsequent ones, causing Terraform to destroy and recreate them.
- `for_each = map_or_set`: Creates one copy per map key or set element, identified by the key (string). Removing one element only removes that one instance — other instances are unaffected.

**Rule:** Use `for_each` when resources have meaningful names. Use `count` only for truly identical copies where the index is sufficient.

---

### Q13: What is remote state and why is it important?

**Answer:** Remote state stores `terraform.tfstate` in a shared location (typically S3) instead of the local filesystem. It enables:
1. **Team collaboration**: everyone operates on the same state
2. **State locking**: only one person applies at a time (via DynamoDB)
3. **Security**: state can be encrypted and access-controlled
4. **CI/CD**: pipelines can access state without a developer's laptop

Without remote state, two engineers could apply simultaneously and corrupt the state file.

---

### Q14: How does state locking work?

**Answer:** When using the S3 backend with DynamoDB, Terraform writes a lock record to DynamoDB before modifying state. If another process tries to modify state at the same time, it reads the existing lock and errors out, showing who has the lock and when they acquired it. After apply completes, the lock is released. This prevents concurrent modifications that would corrupt state.

---

### Q15: What is a data source and how is it different from a resource?

**Answer:** A data source (`data "type" "name" {}`) reads information about **existing** infrastructure — it never creates, changes, or deletes anything. A resource (`resource "type" "name" {}`) creates and manages infrastructure that Terraform owns.

Example: `data "aws_ami" "ubuntu"` finds the latest Ubuntu AMI ID (reads existing). `resource "aws_instance" "web"` creates a new EC2 instance (manages).

---

### Q16: What are Terraform workspaces?

**Answer:** Workspaces let you use the same Terraform configuration with separate state files. Each workspace has its own state, so `terraform workspace select dev` and then `apply` only affects dev resources. The current workspace name is available as `terraform.workspace`. Common uses: dev/staging/prod environments with identical architecture. Limitation: all workspaces share the same backend config and credentials — not suitable for multi-account isolation.

---

### Q17: How would you import an existing resource into Terraform?

**Answer:**
1. Write the resource block in your `.tf` file to match the existing resource
2. Run `terraform import aws_instance.web i-1234567890abcdef0`
3. Run `terraform plan` — it should show no changes if your code matches reality
4. If there are changes, adjust your code until plan shows nothing to change

Modern Terraform (1.5+) also supports an `import {}` block that makes the import part of your code.

---

### Q18: What is the lifecycle meta-argument?

**Answer:** `lifecycle` controls how Terraform handles create, update, and destroy operations for a resource:
- `create_before_destroy = true`: Create replacement before destroying old (zero-downtime)
- `prevent_destroy = true`: Error if Terraform tries to destroy this resource
- `ignore_changes = [field]`: Don't trigger updates when these attributes change externally
- `replace_triggered_by = [resource]`: Force replacement when a referenced resource changes

---

## Advanced Questions

### Q19: How does Terraform detect drift?

**Answer:** Terraform compares three things during `plan`:
1. Your `.tf` code (desired state)
2. The state file (last known state)
3. The real infrastructure (by calling cloud APIs)

When real infrastructure differs from state (someone changed something manually), Terraform plans to revert the change back to what the code specifies. `terraform apply -refresh-only` updates state to match reality without applying code changes.

---

### Q20: What is the difference between taint and replace?

**Answer:**
- **Old way — taint**: `terraform taint aws_instance.web` marks a resource for forced recreation on next apply. Deprecated since Terraform v0.15.2.
- **New way — replace**: `terraform apply -replace=aws_instance.web` replaces a specific resource in a single command. This is the preferred modern approach.

Both cause Terraform to destroy and recreate the resource, useful when a resource is in a bad state and needs to be rebuilt.

---

### Q21: How would you handle a situation where two engineers applied Terraform simultaneously and the state file was corrupted?

**Answer:**
1. If using remote state with DynamoDB locking, this should not be possible — the second apply would fail with a lock error.
2. If state was corrupted despite locking (rare), restore from the S3 versioned state history: download a previous version of the state file and use it.
3. If no backup exists, the path to recovery is painstaking: `terraform import` each resource back into a fresh state file.
4. Prevention: always use remote state + DynamoDB locking. Enable S3 versioning on the state bucket.

---

### Q22: How do you pass outputs from one Terraform configuration to another?

**Answer:** Use `terraform_remote_state` data source:
```hcl
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-state-bucket"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
}
```
The networking configuration must have declared `output "private_subnet_ids" { ... }`.

---

### Q23: Scenario — Your team wants to roll out a change to 100 EC2 instances created with count. How do you minimize disruption?

**Answer:**
Options:
1. **lifecycle create_before_destroy**: Ensures new instance is healthy before old one is destroyed — requires load balancer or equivalent.
2. **-target**: Apply to a subset first: `terraform apply -target='aws_instance.web[0]'`, validate, then proceed.
3. **Blue/green approach**: Create a parallel set of instances (a new resource), switch traffic, then destroy old set.
4. **Consider for_each**: If using count, changing an item in the middle causes cascade. For_each avoids this — each instance is independent.
5. **Auto Scaling Groups**: Better pattern for 100 instances — manage through ASG desired/min/max, Terraform triggers rolling update via launch template.

---

### Q24: What is the difference between terraform refresh and terraform apply -refresh-only?

**Answer:**
- `terraform refresh`: Updates state to match real-world, directly modifies state file, no confirmation prompt. Deprecated behavior.
- `terraform apply -refresh-only`: Safer modern equivalent. Shows you what state changes would be made (just like a plan), asks for confirmation before updating state. Recommended approach.

---

### Q25: How do you debug a Terraform error?

**Answer:**
1. Read the error message carefully — it usually tells you exactly what failed
2. Check the resource's documentation on registry.terraform.io
3. Enable debug logging: `TF_LOG=DEBUG terraform apply 2>&1 | tee debug.log`
4. Use `terraform state show` to inspect the current state of the failing resource
5. Use `terraform console` to interactively test expressions:
   ```bash
   terraform console
   > var.environment
   > aws_vpc.main.id
   ```
6. Check AWS CloudTrail for the actual API error (the real reason often hides behind Terraform's error message)

---

## Scenario Questions

### Q26: You need to create the same infrastructure in 3 AWS regions. How would you approach this?

**Answer:** Use provider aliases:
```hcl
provider "aws" { alias = "us_east"; region = "us-east-1" }
provider "aws" { alias = "us_west"; region = "us-west-2" }
provider "aws" { alias = "eu_west"; region = "eu-west-1" }

module "infra_us_east" { source = "./modules/infra"; providers = { aws = aws.us_east } }
module "infra_us_west" { source = "./modules/infra"; providers = { aws = aws.us_west } }
module "infra_eu_west" { source = "./modules/infra"; providers = { aws = aws.eu_west } }
```

---

### Q27: Your `terraform plan` shows a resource needs to be destroyed and recreated. But you cannot have downtime. What do you do?

**Answer:**
1. Check `lifecycle { create_before_destroy = true }` on the resource — this creates the new resource before destroying the old one.
2. If it is an EC2 instance behind a load balancer, the load balancer health check ensures traffic only goes to the new instance after it passes health checks.
3. For databases: avoid in-place recreation. Use snapshots and restore. Or blue/green database approach.
4. Consider whether the change actually needs a replacement (immutable attribute changing) vs an in-place update (mutable attribute).

---

## Quick Reference Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                     INTERVIEW CHEAT SHEET                       │
├─────────────────────────────────────────────────────────────────┤
│  IaC: infrastructure described as code, not clicks              │
│  init: download providers + modules, set up backend             │
│  plan: dry run — shows changes, does nothing                    │
│  apply: execute the plan, creates/modifies real infra           │
│  destroy: remove all managed resources                          │
│                                                                 │
│  state: JSON file mapping HCL to real resource IDs              │
│  remote state: S3 + DynamoDB (shared, locked, encrypted)        │
│  drift: real world differs from state (manual changes)          │
│                                                                 │
│  count vs for_each: numeric index vs named key                  │
│  count remove: renumbers all → mass destroy/recreate            │
│  for_each remove: only that one key is removed                  │
│                                                                 │
│  module: reusable package. inputs=variables, outputs=outputs    │
│  workspace: same code, separate state per environment           │
│                                                                 │
│  sensitive=true: hide from terminal, NOT from state             │
│  prevent_destroy: protect production resources                  │
│  create_before_destroy: zero-downtime replacement               │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← CI/CD Integration](../09_best_practices/ci_cd_integration.md) &nbsp;|&nbsp; **Next:** [AWS Cloud Foundations →](../../03_AWS/01_cloud_foundations/theory.md)

**Related Topics:** [What is Terraform](../01_introduction/what_is_terraform.md) · [State File](../05_state_management/state_file.md) · [Modules](../06_modules/creating_modules.md) · [Best Practices](../09_best_practices/code_organization.md)
