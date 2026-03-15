# Installing Terraform — Getting Your Workstation Ready

## Before You Write a Single Line of Code

Before a chef can cook, they need a kitchen. Before a carpenter can build, they need tools. Before you can use Terraform, you need to install it and connect it to your AWS account.

This guide walks you through three things:
1. Installing Terraform on Linux, Mac, or Windows
2. Managing multiple Terraform versions with `tfenv`
3. Connecting Terraform to your AWS account

---

## Installing Terraform

### On Linux (Ubuntu/Debian)

The official HashiCorp method using their package repository:

```bash
# Add HashiCorp's GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add the repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install Terraform
sudo apt update && sudo apt install terraform -y
```

### On Linux (RHEL/CentOS/Amazon Linux)

```bash
# Add HashiCorp's YUM repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo \
  https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Install Terraform
sudo yum install terraform -y
```

### On macOS (Homebrew)

```bash
# Install Homebrew first if you don't have it:
# /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

### On Windows

Option 1 — Winget (Windows Package Manager):
```powershell
winget install HashiCorp.Terraform
```

Option 2 — Chocolatey:
```powershell
choco install terraform
```

Option 3 — Manual:
1. Download the zip from https://developer.hashicorp.com/terraform/downloads
2. Extract `terraform.exe` to a folder like `C:\terraform`
3. Add that folder to your `PATH` environment variable

---

## Verifying Your Installation

After installing, confirm it works:

```bash
terraform version
```

Expected output:
```
Terraform v1.7.0
on linux_amd64
```

Also useful:
```bash
terraform -help              # list all commands
terraform -help plan         # help for a specific command
```

---

## Managing Versions with tfenv

Think of `tfenv` as a wardrobe for Terraform versions. Different projects may require different versions of Terraform. `tfenv` lets you install multiple versions and switch between them instantly — like changing outfits for different occasions.

### Install tfenv on Linux/macOS

```bash
# Clone tfenv into ~/.tfenv
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv

# Add to your PATH (add to ~/.bashrc or ~/.zshrc)
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Using tfenv

```bash
# List all available Terraform versions
tfenv list-remote

# Install a specific version
tfenv install 1.7.0
tfenv install 1.6.6

# Use a version globally
tfenv use 1.7.0

# List installed versions (* = currently active)
tfenv list
# * 1.7.0
#   1.6.6

# Pin a version for a specific project
# Create a .terraform-version file in the project directory
echo "1.6.6" > .terraform-version
# tfenv will automatically use 1.6.6 when you cd into this directory
```

---

## Setting Up AWS Credentials

Terraform needs permission to talk to your AWS account. Think of it like giving a builder the keys to the construction site — without the keys, they cannot get in.

### Method 1: AWS CLI Configuration (Recommended for Local Dev)

First, install the AWS CLI:
```bash
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# macOS
brew install awscli

# Verify
aws --version
```

Then configure credentials:
```bash
aws configure
```

It will prompt you for:
```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: json
```

This creates two files:
```
~/.aws/credentials    ← stores your keys
~/.aws/config         ← stores region and output format
```

### Method 2: Environment Variables

Useful in CI/CD pipelines or when you need to switch accounts quickly:

```bash
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"

# Now run Terraform — it picks these up automatically
terraform plan
```

### Method 3: AWS Profiles (For Multiple Accounts)

If you work with multiple AWS accounts (dev, staging, prod):

```bash
# Configure a named profile
aws configure --profile mycompany-dev
aws configure --profile mycompany-prod

# Use a specific profile with Terraform
export AWS_PROFILE=mycompany-dev
terraform apply

# Or set it in your provider block
provider "aws" {
  region  = "us-east-1"
  profile = "mycompany-dev"
}
```

### Method 4: IAM Roles (Best for EC2/CI/CD)

When Terraform runs on an EC2 instance or in CI/CD, attach an IAM role to the instance/runner. Terraform automatically uses the role's temporary credentials — no keys needed.

```
┌──────────────────────────────────────────────────────────┐
│              CREDENTIAL PRIORITY ORDER                   │
│                                                          │
│  1. Static credentials in provider block (avoid!)        │
│  2. Environment variables (AWS_ACCESS_KEY_ID, etc.)      │
│  3. AWS credentials file (~/.aws/credentials)            │
│  4. AWS config file (~/.aws/config)                      │
│  5. EC2 instance profile / ECS task role  ← best for CI  │
│                                                          │
│  Terraform checks these in order, top to bottom          │
└──────────────────────────────────────────────────────────┘
```

---

## Your First Terraform Project

Let us verify everything works end-to-end. Create a test directory:

```bash
mkdir ~/terraform-test && cd ~/terraform-test
```

Create `main.tf`:

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Just look up your AWS account ID — no resources created
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```

Run it:
```bash
terraform init     # downloads AWS provider (~100MB)
terraform plan     # shows it will output your account ID
terraform apply    # runs it
```

Expected output:
```
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:
account_id = "123456789012"
```

If you see your AWS account ID, Terraform is installed and connected correctly.

---

## Common Installation Issues

```
┌──────────────────────────────────────────────────────────┐
│                TROUBLESHOOTING                           │
├──────────────────────────────────────────────────────────┤
│ Error: terraform command not found                       │
│ Fix: Check PATH, re-run: source ~/.bashrc                │
│                                                          │
│ Error: No valid credentials found                        │
│ Fix: Run `aws configure` or set AWS_ACCESS_KEY_ID env var│
│                                                          │
│ Error: Error configuring the backend                     │
│ Fix: Run `terraform init` first                          │
│                                                          │
│ Error: Required plugins are not installed                │
│ Fix: Delete .terraform/ folder and run `terraform init`  │
└──────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  Linux install: apt/yum package manager via HashiCorp repo      │
│  macOS install: brew install hashicorp/tap/terraform            │
│  Windows install: winget install HashiCorp.Terraform            │
│  tfenv: manage multiple Terraform versions per project          │
│  AWS credentials: aws configure (local), env vars (CI/CD),      │
│                   IAM role (EC2/runner — best practice)         │
│  Verify: terraform version && terraform apply (data source)     │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Terraform vs Others](./terraform_vs_others.md) &nbsp;|&nbsp; **Next:** [HCL Syntax →](../02_hcl_basics/syntax.md)

**Related Topics:** [What is Terraform](./what_is_terraform.md) · [Providers](../03_providers_resources/providers.md) · [Security Best Practices](../09_best_practices/security.md)
