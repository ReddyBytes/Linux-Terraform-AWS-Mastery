# Remote State — Sharing Terraform State Across Teams

## The Shared Office Filing Cabinet Analogy

Imagine a law firm where paralegals work on the same cases. If each paralegal keeps their case notes in a personal desk drawer, chaos ensues: two people might edit the same document simultaneously, one person's updates overwrite another's, and nobody has the full picture.

The solution is a shared, locked filing cabinet. Only one person can access a folder at a time. Everyone sees the same up-to-date documents. Changes are logged.

Remote state is Terraform's shared, locked filing cabinet. Instead of storing state on a developer's laptop, it lives in a shared location (typically an S3 bucket), protected by a lock (DynamoDB), accessible to every team member and CI/CD pipeline.

---

## Why Remote State?

```
┌──────────────────────────────────────────────────────────────────┐
│             LOCAL STATE vs REMOTE STATE                          │
│                                                                  │
│  LOCAL STATE (the problem):                                      │
│  ┌──────────┐   ┌──────────┐                                     │
│  │ Dev A    │   │ Dev B    │                                     │
│  │ laptop   │   │ laptop   │                                     │
│  │tfstate ✓ │   │tfstate ✗ │  ← B has stale/missing state       │
│  └──────────┘   └──────────┘                                     │
│  No locking → A and B apply simultaneously → state corruption    │
│                                                                  │
│  REMOTE STATE (the solution):                                    │
│  ┌──────────┐   ┌──────────┐                                     │
│  │ Dev A    │   │ Dev B    │                                     │
│  │ laptop   │   │ laptop   │                                     │
│  └────┬─────┘   └────┬─────┘                                     │
│       └──────┬────────┘                                          │
│              ▼                                                   │
│     ┌─────────────────────┐                                      │
│     │  S3 Bucket          │ ← single source of truth            │
│     │  terraform.tfstate  │                                      │
│     └─────────────────────┘                                      │
│     ┌─────────────────────┐                                      │
│     │  DynamoDB Table     │ ← state lock                        │
│     │  Prevents conflicts │                                      │
│     └─────────────────────┘                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## Setting Up the S3 Backend

### Step 1: Create the S3 Bucket and DynamoDB Table

You need to create these resources BEFORE configuring the backend. You can do this manually or with a separate Terraform config (a "bootstrap" config):

```hcl
# bootstrap/main.tf — run this once to set up the backend infrastructure

provider "aws" {
  region = "us-east-1"
}

# S3 bucket for state storage
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state-prod"

  # Prevent accidental deletion of this bucket
  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name    = "Terraform State"
    Purpose = "terraform-backend"
  }
}

# Enable versioning — recover from accidental state deletion
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Encrypt state at rest
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block all public access to state
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"   # this exact name is required by Terraform

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name    = "Terraform State Locks"
    Purpose = "terraform-backend"
  }
}

output "state_bucket_name" {
  value = aws_s3_bucket.terraform_state.id
}

output "lock_table_name" {
  value = aws_dynamodb_table.terraform_locks.name
}
```

Run it:
```bash
cd bootstrap/
terraform init
terraform apply
```

### Step 2: Configure the Backend

Now configure your actual project to use the remote backend:

```hcl
# backend.tf (or in your terraform {} block in versions.tf)
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-prod"
    key            = "myapp/production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

The `key` is the path within the S3 bucket. Use a structured naming convention:
```
# Naming convention for state keys:
<project>/<environment>/terraform.tfstate

# Examples:
myapp/production/terraform.tfstate
myapp/staging/terraform.tfstate
networking/shared/terraform.tfstate
```

### Step 3: Initialize the New Backend

```bash
terraform init
```

If you had local state, Terraform will ask if you want to copy it to the remote backend:
```
Do you want to copy existing state to the new backend?
  Enter a value: yes
```

---

## State Locking

When someone runs `terraform apply`, Terraform writes a lock record to DynamoDB. Anyone else trying to apply at the same time will see:

```
Error: Error acquiring the state lock

Error message: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        3d2a6b8c-1234-5678-abcd-ef0123456789
  Path:      myapp/production/terraform.tfstate
  Operation: OperationTypeApply
  Who:       alice@company.com
  Version:   1.7.0
  Created:   2024-01-15 14:23:01.123456789 +0000 UTC
```

This prevents two people from applying simultaneously and corrupting state.

If a lock gets stuck (network failure mid-apply):
```bash
# Force-unlock — use the lock ID from the error message
terraform force-unlock 3d2a6b8c-1234-5678-abcd-ef0123456789
```

---

## Migrating Local State to Remote

If you started with local state and want to switch to remote:

```bash
# 1. Add the backend configuration to your .tf files
# 2. Run terraform init — it detects the backend change
terraform init

# Terraform will ask:
# "Do you want to copy existing state to the new backend?"
# Type: yes

# 3. Verify state was migrated
terraform state list

# 4. Delete local state files (they are now in S3)
rm terraform.tfstate terraform.tfstate.backup
```

---

## Backend Configuration with Variables

The `backend` block cannot use variables or locals. But you can use partial configuration:

```hcl
# backend.tf — minimal config
terraform {
  backend "s3" {}   # empty — values provided at init time
}
```

```bash
# Provide backend values at init time
terraform init \
  -backend-config="bucket=mycompany-terraform-state" \
  -backend-config="key=myapp/prod/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=terraform-state-locks"

# Or use a backend config file
terraform init -backend-config=backends/prod.hcl
```

```hcl
# backends/prod.hcl
bucket         = "mycompany-terraform-state"
key            = "myapp/prod/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-state-locks"
encrypt        = true
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  Remote state = store tfstate in S3 instead of local disk       │
│  Benefits: team sharing, locking, versioning, encryption        │
│                                                                 │
│  Setup requires:                                                │
│  1. S3 bucket (versioned + encrypted + private)                 │
│  2. DynamoDB table (hash_key = "LockID")                        │
│  3. backend "s3" { ... } block in your tf config                │
│  4. terraform init to migrate/initialize                        │
│                                                                 │
│  State locking: DynamoDB prevents concurrent applies            │
│  force-unlock: use only if lock is genuinely stuck              │
│  Key naming: project/environment/terraform.tfstate              │
│  Backend cannot use variables — use partial config files        │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← State File](./state_file.md) &nbsp;|&nbsp; **Next:** [State Commands →](./state_commands.md)

**Related Topics:** [State File](./state_file.md) · [State Commands](./state_commands.md) · [S3 with Terraform](../08_aws_with_terraform/s3.md) · [CI/CD Integration](../09_best_practices/ci_cd_integration.md)
