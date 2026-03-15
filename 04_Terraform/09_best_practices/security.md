# Security Best Practices — Keeping Terraform Safe

## The Safe and the Sticky Note Analogy

Imagine an employee who keeps their safe combination written on a sticky note attached to the front of the safe. The entire purpose of the safe is defeated. In software, hardcoding secrets (passwords, API keys, credentials) directly in code files that get committed to Git is exactly the same mistake — and one that real companies have made with costly consequences.

This guide covers the most important security practices for Terraform projects.

---

## Rule 1: Never Hardcode Credentials

The most critical rule: do not put credentials in your `.tf` files.

```hcl
# DANGEROUS — DO NOT DO THIS
provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"          # ← this ends up in Git history
  secret_key = "wJalrXUtnFEMI/K7MDENG..."     # ← even if you delete it later
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = "SuperSecret123!"  # ← visible to anyone who sees this file
}

# SAFE — use variables
provider "aws" {
  region = "us-east-1"
  # Credentials come from environment, ~/.aws/credentials, or instance profile
}

resource "aws_db_instance" "main" {
  username = var.db_username
  password = var.db_password   # value provided via TF_VAR_db_password or tfvars
}
```

**How credentials should be provided:**
1. **Local development**: `aws configure` (stored in `~/.aws/credentials`, never in your project)
2. **CI/CD pipelines**: Environment variables (`TF_VAR_db_password`, `AWS_ACCESS_KEY_ID`) set as pipeline secrets
3. **EC2 / Lambda / ECS**: IAM instance/execution roles — no credentials needed at all

---

## Rule 2: Mark Sensitive Variables and Outputs

```hcl
# Mark sensitive variables — Terraform hides value in plan/apply output
variable "db_password" {
  type      = string
  sensitive = true
}

variable "api_key" {
  type      = string
  sensitive = true
}

# Mark sensitive outputs
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}

output "connection_string" {
  value     = "postgres://${var.db_username}:${var.db_password}@${aws_db_instance.main.endpoint}/${var.db_name}"
  sensitive = true   # mark sensitive if value contains a sensitive variable
}
```

**Important:** `sensitive = true` hides values from terminal output, but they are still stored in the state file in plaintext. You must also secure the state file.

---

## Rule 3: Protect Your State File

The state file can contain passwords, private keys, and connection strings in plaintext. Treat it like a secret:

```hcl
# backend.tf — use S3 with encryption + access controls
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "myapp/prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true    # ← encrypt at rest with S3 SSE
    dynamodb_table = "terraform-state-locks"
  }
}
```

```hcl
# Restrict S3 bucket access — only Terraform CI/CD role should access it
resource "aws_s3_bucket_policy" "state" {
  bucket = aws_s3_bucket.terraform_state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource  = ["${aws_s3_bucket.terraform_state.arn}/*"]
        Condition = {
          StringNotEquals = {
            "aws:PrincipalArn" = [
              "arn:aws:iam::${local.account_id}:role/terraform-ci-role",
              "arn:aws:iam::${local.account_id}:role/platform-team-role"
            ]
          }
        }
      }
    ]
  })
}
```

---

## Rule 4: .gitignore — What to Never Commit

Create a `.gitignore` at the root of your Terraform project:

```gitignore
# .gitignore for Terraform projects

# State files — NEVER commit
*.tfstate
*.tfstate.*
*.tfstate.backup

# Terraform working directory
.terraform/

# Variable files that might contain secrets
terraform.tfvars
*.auto.tfvars
# Exception: commit non-secret tfvars like:
# environments/dev/terraform.tfvars (if it only has non-secret values)

# Override files — local developer customizations, never commit
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Generated private keys
*.pem
*.key
keys/

# Crash logs
crash.log
crash.*.log

# Terraform plan files (can contain sensitive data)
*.tfplan
plan.out

# OS files
.DS_Store
Thumbs.db
```

---

## Rule 5: Store Secrets in AWS Secrets Manager or SSM

Never store production passwords in `.tfvars` files. Instead, use AWS managed secrets:

```hcl
# Put a secret into Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "${var.project}/${var.environment}/db-password"
  recovery_window_in_days = 7   # 7 day safety window before permanent deletion
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = var.db_password   # value provided at apply time, never hardcoded
}

# Read the secret back in another resource
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}

# Or use AWS SSM Parameter Store (simpler, cheaper for non-rotating secrets)
resource "aws_ssm_parameter" "db_password" {
  name  = "/${var.project}/${var.environment}/db-password"
  type  = "SecureString"   # encrypted with KMS
  value = var.db_password
}

data "aws_ssm_parameter" "db_password" {
  name            = "/${var.project}/${var.environment}/db-password"
  with_decryption = true
}
```

For RDS, use the managed password feature:
```hcl
resource "aws_db_instance" "main" {
  # Let AWS generate AND rotate the password automatically
  manage_master_user_password = true
  # The password is stored and rotated in Secrets Manager automatically
  # No 'password' argument needed!
}
```

---

## Rule 6: Least-Privilege IAM

Give every role only exactly the permissions it needs:

```hcl
# BAD: over-privileged
resource "aws_iam_role_policy_attachment" "too_permissive" {
  role       = aws_iam_role.app.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"  # NEVER
}

# GOOD: least-privilege custom policy
data "aws_iam_policy_document" "app_specific" {
  statement {
    effect  = "Allow"
    actions = [
      "s3:GetObject",
      "s3:PutObject"
    ]
    resources = ["${aws_s3_bucket.app_data.arn}/*"]  # specific bucket only
  }

  statement {
    effect  = "Allow"
    actions = ["ssm:GetParameter"]
    resources = ["arn:aws:ssm:${var.region}:${local.account_id}:parameter/${var.project}/${var.environment}/*"]
  }
}
```

---

## Rule 7: Use lifecycle prevent_destroy for Critical Resources

```hcl
resource "aws_db_instance" "production" {
  # ...

  lifecycle {
    prevent_destroy = true   # terraform destroy will fail if this exists
  }
}

resource "aws_s3_bucket" "terraform_state" {
  lifecycle {
    prevent_destroy = true
  }
}
```

---

## Security Scanning with Checkov

[Checkov](https://www.checkov.io/) is a free static analysis tool that scans Terraform code for security misconfigurations:

```bash
# Install checkov
pip install checkov

# Scan your Terraform directory
checkov -d .

# Example output:
# PASSED checks: 45
# FAILED checks: 3
#   Check: CKV_AWS_18: "Ensure the S3 bucket has access logging enabled"
#   Check: CKV_AWS_21: "Ensure the S3 bucket has versioning enabled"
#   Check: CKV_AWS_86: "Ensure S3 bucket has a lifecycle configuration"
```

---

## Security Checklist

```
┌──────────────────────────────────────────────────────────────────┐
│                  TERRAFORM SECURITY CHECKLIST                    │
├──────────────────────────────────────────────────────────────────┤
│  [ ] No credentials hardcoded in .tf files                       │
│  [ ] All passwords/keys are sensitive = true                     │
│  [ ] .gitignore includes *.tfstate, .terraform/, *.tfvars        │
│  [ ] Remote state uses S3 with encryption enabled                │
│  [ ] State S3 bucket has restricted bucket policy                │
│  [ ] Secrets stored in Secrets Manager or SSM, not tfvars        │
│  [ ] All IAM roles follow least-privilege                        │
│  [ ] No wildcards (*) in IAM actions where avoidable             │
│  [ ] RDS not publicly accessible                                 │
│  [ ] S3 buckets have public access block (unless intentional)    │
│  [ ] Security groups restrict SSH to specific IPs (not 0.0.0.0) │
│  [ ] prevent_destroy on production databases and critical buckets│
│  [ ] Checkov or tfsec runs in CI on every PR                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  Never hardcode credentials in .tf files                        │
│  sensitive = true hides values from terminal (not from state!)  │
│  Remote state: S3 + encryption + restrict bucket policy         │
│  .gitignore: *.tfstate, .terraform/, *.tfvars (with secrets)    │
│  Secrets: use AWS Secrets Manager or SSM Parameter Store        │
│  IAM: least-privilege, no AdministratorAccess on services       │
│  prevent_destroy: protect production databases and state buckets│
│  Checkov: static analysis security scanner for Terraform code   │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Code Organization](./code_organization.md) &nbsp;|&nbsp; **Next:** [CI/CD Integration →](./ci_cd_integration.md)

**Related Topics:** [Code Organization](./code_organization.md) · [Remote State](../05_state_management/remote_state.md) · [IAM](../08_aws_with_terraform/iam.md) · [Variables](../04_variables_outputs/variables.md)
