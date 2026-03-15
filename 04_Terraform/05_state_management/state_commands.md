# Terraform State Commands — Inspecting and Manipulating State

## The Warehouse Inventory Tool Analogy

A warehouse manager has a sophisticated inventory system. They can query it ("show me everything in aisle 5"), move items between locations ("transfer item X from shelf A to shelf B"), remove outdated records ("delete the record for item Y that we sold"), and add items that arrived without a receipt ("import this new shipment into the system").

Terraform's state commands are that inventory management toolkit. They let you inspect, move, rename, remove, and import resources in the state file — all without touching your `.tf` code or creating/destroying real infrastructure.

---

## State Command Overview

```
┌──────────────────────────────────────────────────────────────────┐
│              TERRAFORM STATE COMMANDS                            │
├──────────────────────┬───────────────────────────────────────────┤
│  terraform state list│ List all resources tracked in state       │
│  terraform state show│ Show all attributes of a resource         │
│  terraform state mv  │ Rename/move a resource in state           │
│  terraform state rm  │ Remove a resource from state              │
│  terraform import    │ Bring existing resource under management   │
│  terraform refresh   │ Update state to match real-world          │
└──────────────────────┴───────────────────────────────────────────┘
```

---

## terraform state list

Lists all resources tracked in the current state file.

```bash
terraform state list
```

Example output:
```
aws_instance.web
aws_security_group.web
aws_subnet.private[0]
aws_subnet.private[1]
aws_subnet.public[0]
aws_subnet.public[1]
aws_vpc.main
data.aws_ami.amazon_linux
module.rds.aws_db_instance.main
module.rds.aws_db_subnet_group.main
```

Filter by a pattern:
```bash
terraform state list 'aws_instance.*'
terraform state list 'module.rds.*'
```

---

## terraform state show

Shows all attributes of a specific resource in state. Equivalent to "read one row from the inventory ledger."

```bash
terraform state show aws_instance.web
```

Example output:
```
# aws_instance.web:
resource "aws_instance" "web" {
    ami                                  = "ami-0c55b159cbfafe1f0"
    arn                                  = "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"
    availability_zone                    = "us-east-1a"
    id                                   = "i-1234567890abcdef0"
    instance_state                       = "running"
    instance_type                        = "t3.micro"
    private_ip                           = "10.0.1.5"
    public_ip                            = "54.210.167.204"
    tags                                 = {
        "Environment" = "prod"
        "ManagedBy"   = "Terraform"
        "Name"        = "web-server"
    }
    ... (all attributes)
}
```

Useful for:
- Debugging: see the real values Terraform has stored
- Finding attribute names to use in references
- Verifying an import worked correctly

---

## terraform state mv

Renames a resource in the state file. The infrastructure in AWS is NOT changed — only the state record is renamed.

**Common use case:** You want to rename a resource in your `.tf` file without destroying and recreating it.

```bash
# Rename a resource (after updating your .tf code)
terraform state mv aws_instance.old_name aws_instance.new_name

# Move a resource into a module
terraform state mv aws_vpc.main module.networking.aws_vpc.main

# Move a resource out of a module
terraform state mv module.networking.aws_vpc.main aws_vpc.main

# Move a resource from count-indexed to for_each-indexed
terraform state mv 'aws_subnet.public[0]' 'aws_subnet.public["us-east-1a"]'
```

Example workflow when refactoring:
```hcl
# BEFORE: resource named "app_server"
resource "aws_instance" "app_server" { ... }

# AFTER: you rename it to "web_server" in your .tf file
resource "aws_instance" "web_server" { ... }
```

```bash
# Without state mv: Terraform destroys app_server, creates web_server
# With state mv: just renames it, no recreation
terraform state mv aws_instance.app_server aws_instance.web_server
terraform plan  # should show: no changes
```

---

## terraform state rm

Removes a resource from the state file. The infrastructure in AWS is NOT deleted — Terraform just forgets about it.

**Common use cases:**
- You want to stop managing a resource with Terraform (hand it off to another team/tool)
- A resource was already deleted outside of Terraform and you want to clean up state
- You want to move a resource to a different Terraform config/workspace

```bash
# Remove a single resource
terraform state rm aws_instance.web

# Remove a resource instance (count)
terraform state rm 'aws_instance.web[0]'

# Remove a resource instance (for_each)
terraform state rm 'aws_instance.web["frontend"]'

# Remove all resources in a module
terraform state rm module.old_networking
```

After `terraform state rm`, if you run `terraform plan` and that resource is still in your `.tf` code, Terraform will show it as a resource to create (since it's no longer in state). Remove it from your code too, or use `terraform import` to re-import it.

---

## terraform import

Brings an existing AWS resource that was created outside of Terraform (manually, via CLI, by another tool) under Terraform management.

**Workflow:**
1. Write the resource block in your `.tf` file
2. Run `terraform import`
3. Run `terraform plan` to verify the imported state matches your code
4. Adjust your code until plan shows no changes

```bash
# Syntax:
terraform import <resource_address> <real_world_id>

# Import an EC2 instance
terraform import aws_instance.web i-1234567890abcdef0

# Import an S3 bucket
terraform import aws_s3_bucket.my_bucket my-existing-bucket-name

# Import a VPC
terraform import aws_vpc.main vpc-0123456789abcdef0

# Import a security group
terraform import aws_security_group.web sg-12345678

# Import an IAM role
terraform import aws_iam_role.app my-existing-role-name

# Import an RDS instance
terraform import aws_db_instance.main mydb-instance-identifier

# Import a resource with a count index
terraform import 'aws_instance.web[0]' i-1234567890abcdef0

# Import a resource with a for_each key
terraform import 'aws_instance.web["frontend"]' i-1234567890abcdef0
```

### Modern Import Block (Terraform 1.5+)

```hcl
# import.tf — import as code (no CLI flag needed)
import {
  id = "i-1234567890abcdef0"
  to = aws_instance.web
}

import {
  id = "my-existing-bucket"
  to = aws_s3_bucket.assets
}
```

With `terraform plan`, Terraform will generate the import and show you what it will import.

---

## terraform refresh

Updates the state file to match the current real-world state of resources — without making any changes to resources.

```bash
terraform refresh
```

Use with caution: if someone has manually changed resources, `refresh` will update state to reflect those manual changes. The next `plan` will then show your `.tf` code wants to revert them.

Modern alternative: `terraform apply -refresh-only` (safer, shows what changed before updating state):

```bash
terraform apply -refresh-only
# Shows: state will be updated to match reality
# Prompts for confirmation
```

---

## Useful State Debugging Workflow

```bash
# 1. See everything in state
terraform state list

# 2. Inspect a specific resource
terraform state show aws_instance.web

# 3. Check for drift (state vs real world)
terraform plan -refresh-only

# 4. If resources were manually renamed in AWS, refresh
terraform apply -refresh-only

# 5. List and grep for specific resource types
terraform state list | grep aws_security_group
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  terraform state list           → list all tracked resources    │
│  terraform state show <address> → show all attrs of a resource  │
│  terraform state mv A B         → rename in state (no AWS change│
│  terraform state rm <address>   → remove from state (no delete) │
│  terraform import <addr> <id>   → add existing resource to state│
│  terraform refresh              → sync state with real-world    │
│  terraform apply -refresh-only  → safer version of refresh      │
│                                                                 │
│  state mv: use when refactoring resource names (avoid recreate) │
│  state rm: use when handing off a resource to another config    │
│  import: use when adopting manually-created infrastructure      │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Remote State](./remote_state.md) &nbsp;|&nbsp; **Next:** [Creating Modules →](../06_modules/creating_modules.md)

**Related Topics:** [State File](./state_file.md) · [Remote State](./remote_state.md) · [Resources](../03_providers_resources/resources.md)
