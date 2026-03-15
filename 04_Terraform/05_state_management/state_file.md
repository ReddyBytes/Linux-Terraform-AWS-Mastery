# The Terraform State File — Terraform's Memory

## The Inventory Ledger Analogy

Imagine a warehouse manager who is responsible for 10,000 items. They do not memorize every item's location — they maintain a ledger. The ledger records what exists, where it is, and what condition it is in. When someone wants to add or remove an item, the manager checks the ledger first to understand the current state, plans the change, and then updates both the warehouse and the ledger.

Terraform's state file (`terraform.tfstate`) is that ledger. It is Terraform's memory of what infrastructure it has created and how it is configured. Without the state file, Terraform would have no idea what it has built.

---

## What is terraform.tfstate?

`terraform.tfstate` is a JSON file created in your working directory after the first `terraform apply`. It maps your HCL code to real-world resources.

```
┌──────────────────────────────────────────────────────────────────┐
│                 WHAT STATE DOES                                  │
│                                                                  │
│  Your HCL code says:                 State records:             │
│  ─────────────────────               ────────────────────────── │
│  resource "aws_instance" "web"  ───► id = "i-1234567890abcdef0" │
│  { instance_type = "t3.micro" } ───► public_ip = "54.1.2.3"    │
│                                ───► private_ip = "10.0.1.5"    │
│                                ───► all 50+ attributes          │
│                                                                  │
│  When you run terraform plan:                                    │
│  Terraform reads your .tf files                                  │
│  Terraform reads the state file                                  │
│  Terraform calls AWS API to check current reality                │
│  Compares all three → generates a diff                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## What the State File Contains

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 5,
  "lineage": "3d2a6b8c-1234-5678-abcd-ef0123456789",
  "outputs": {
    "web_server_ip": {
      "value": "54.210.167.204",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "i-1234567890abcdef0",
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t3.micro",
            "public_ip": "54.210.167.204",
            "private_ip": "10.0.1.5",
            "availability_zone": "us-east-1a",
            "...": "all 50+ attributes stored here"
          }
        }
      ]
    }
  ]
}
```

The state contains **every attribute** of every resource Terraform manages — including sensitive ones like passwords, keys, and connection strings.

---

## The Three-Way Comparison

During `terraform plan`, Terraform does a three-way comparison:

```
┌─────────────────────────────────────────────────────────────────┐
│               THREE-WAY COMPARISON                              │
│                                                                 │
│   1. Your .tf code     2. State file       3. Real AWS          │
│   (desired state)      (last known)        (actual current)     │
│   ────────────────      ────────────        ─────────────────── │
│   t3.large              t3.micro            t3.micro            │
│                                                                 │
│   Terraform sees: code wants t3.large, state says t3.micro,    │
│   AWS also has t3.micro → CHANGE: update instance type          │
│                                                                 │
│   Another scenario: manual change in AWS console               │
│   Code: t3.micro  State: t3.micro  AWS: t3.large               │
│   Terraform sees DRIFT → plan shows: revert to t3.micro         │
└─────────────────────────────────────────────────────────────────┘
```

---

## State Drift

State drift happens when someone manually changes infrastructure outside of Terraform (clicking in the AWS console, running AWS CLI, another tool modifying resources).

```
Code:  instance_type = "t3.micro"    ← what you wrote
State: instance_type = "t3.micro"    ← what Terraform last knew
AWS:   instance_type = "t3.large"    ← someone changed this manually!

terraform plan:
  ~ aws_instance.web
      instance_type: "t3.large" → "t3.micro"  ← Terraform will revert it!
```

To detect drift without planning changes:
```bash
terraform refresh    # updates state to match real-world (use carefully)
terraform plan       # shows diff between code and current reality
```

---

## State File Security — Handle With Care

The state file often contains sensitive data in plaintext:

```
┌──────────────────────────────────────────────────────────────────┐
│               WHAT CAN BE IN YOUR STATE FILE                     │
│                                                                  │
│  RDS database passwords                                          │
│  IAM access keys                                                 │
│  Private key material                                            │
│  Database connection strings                                     │
│  API keys from providers                                         │
│  Any value from resources, even sensitive ones                   │
│                                                                  │
│  NEVER commit terraform.tfstate to Git!                          │
│  NEVER share state files via email or Slack!                     │
│  ALWAYS use encrypted remote state (S3 + encryption)            │
└──────────────────────────────────────────────────────────────────┘
```

Add to your `.gitignore`:
```
# .gitignore
*.tfstate
*.tfstate.backup
*.tfstate.*
.terraform/
.terraform.lock.hcl  # optional — some teams commit this
terraform.tfvars     # if it contains secrets
*.auto.tfvars        # if it contains secrets
```

---

## Never Edit State Manually

The state file is a machine-managed file. Editing it by hand is like editing a database's binary log file — easy to corrupt, hard to fix.

```
NEVER DO THIS:
  nano terraform.tfstate
  vim terraform.tfstate

USE THIS INSTEAD:
  terraform state mv   ← move/rename resources in state
  terraform state rm   ← remove a resource from state
  terraform import     ← add existing resources to state
  terraform state show ← read state for a resource
```

---

## The .terraform.tfstate.backup File

Every time state is updated, Terraform creates a backup of the previous state as `terraform.tfstate.backup`. This gives you one level of undo if something goes wrong.

```
terraform.tfstate         ← current state
terraform.tfstate.backup  ← previous state (one apply ago)
```

For more robust backups, use remote state (covered in the next section).

---

## Local State vs Remote State

By default, state is stored locally. This causes problems for teams:

```
┌──────────────────────────────────────────────────────────────────┐
│          LOCAL STATE PROBLEMS FOR TEAMS                          │
│                                                                  │
│  Developer A runs terraform apply → state updated on A's laptop  │
│  Developer B runs terraform plan  → uses OLD state, doesn't know │
│  about A's changes → plan is wrong → potentially dangerous apply │
│                                                                  │
│  Solution: Store state remotely (S3) so everyone uses same state │
│  → Covered in the next file: remote_state.md                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  terraform.tfstate = JSON file, Terraform's memory              │
│  Contains: every resource attribute, outputs, metadata          │
│  Used in plan: code vs state vs real-world = what changes        │
│  Drift = real world differs from state (manual changes)          │
│                                                                  │
│  SECURITY:                                                       │
│  - Contains secrets in plaintext                                 │
│  - NEVER commit to Git                                           │
│  - Add *.tfstate to .gitignore                                   │
│  - Use remote state with encryption for teams                   │
│                                                                  │
│  NEVER edit manually — use terraform state commands              │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Locals](../04_variables_outputs/locals.md) &nbsp;|&nbsp; **Next:** [Remote State →](./remote_state.md)

**Related Topics:** [Remote State](./remote_state.md) · [State Commands](./state_commands.md) · [Security Best Practices](../09_best_practices/security.md)
