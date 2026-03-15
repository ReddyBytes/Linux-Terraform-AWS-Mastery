# What is Terraform? — Infrastructure as Code Explained

## The Problem: Building Infrastructure by Hand

Imagine you are opening a restaurant. On day one, you set up the kitchen manually — you buy the stoves, arrange the tables, hire staff, and wire up the electricity yourself. It works! But then the restaurant becomes popular and you need to open five more locations.

Now you face a nightmare: you have to repeat every single step, from memory, at each new location. You will forget something. One location will have the tables in the wrong arrangement. Another will be missing an oven. Every location will be slightly different, and troubleshooting problems will be a mess because no two kitchens are the same.

This is exactly what happens when engineers build cloud infrastructure by clicking through the AWS console manually.

---

## What is Infrastructure as Code (IaC)?

**Infrastructure as Code (IaC)** means you describe your entire infrastructure in text files — just like a blueprint for a building. Instead of clicking buttons in a web console, you write code that says "I want a server with 4 CPUs, in this region, with these firewall rules." Then a tool reads that blueprint and builds it for you.

```
Traditional Approach (ClickOps)         Infrastructure as Code
─────────────────────────────           ──────────────────────────────
  Engineer logs into AWS console    →     Engineer writes a config file
  Clicks "Launch Instance"          →     Runs: terraform apply
  Fills in 20 form fields           →     Tool reads the file and builds it
  Repeats for every environment     →     Reuse the same file everywhere
  Forgets what was configured       →     File is the single source of truth
  Can't reproduce exact setup       →     100% reproducible every time
```

**The blueprint analogy:** An architect does not build a skyscraper by winging it. They produce detailed blueprints first. Every contractor reads the same blueprint and builds to specification. IaC is the blueprint for your infrastructure.

---

## What is Terraform?

**Terraform** is an open-source IaC tool created by HashiCorp in 2014. It lets you define cloud infrastructure using a simple, human-readable language called **HCL (HashiCorp Configuration Language)**, then provisions and manages that infrastructure across dozens of cloud providers.

```
┌─────────────────────────────────────────────────────────────────┐
│                        TERRAFORM                                │
│                                                                 │
│   You write HCL files     Terraform reads them                  │
│   ┌──────────────┐        ┌──────────────────────┐             │
│   │  main.tf     │  ───►  │  terraform plan      │             │
│   │  variables.tf│        │  (shows what changes)│             │
│   │  outputs.tf  │        └──────────┬───────────┘             │
│   └──────────────┘                   │ terraform apply          │
│                                      ▼                          │
│                          ┌───────────────────────┐             │
│                          │        AWS             │             │
│                          │  EC2  S3  VPC  RDS     │             │
│                          │  (real infra created)  │             │
│                          └───────────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

Terraform is **cloud-agnostic**: the same workflow applies whether you are building on AWS, Azure, Google Cloud, or even on-premises tools like VMware.

---

## The Core Terraform Workflow

Think of Terraform like ordering from a factory. You submit an order form (your HCL files), the factory previews the order back to you for confirmation, you approve it, and the factory builds your product. If you want changes, you update the order form and repeat.

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│             │    │             │    │             │    │             │
│  terraform  │    │  terraform  │    │  terraform  │    │  terraform  │
│    init     │───►│    plan     │───►│    apply    │───►│   destroy   │
│             │    │             │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
  Downloads          Shows WHAT         Actually           Tears down
  providers &        will change        builds the         everything
  sets up            before you         infrastructure     when done
  backend            approve it
```

### Step 1: `terraform init`

Initializes your working directory. Downloads the required provider plugins (e.g., the AWS provider) and sets up the backend for storing state.

```bash
terraform init
```

```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.31.0...
Terraform has been successfully initialized!
```

### Step 2: `terraform plan`

Reads your HCL files and compares them to the current real-world state. It shows you exactly what it will create, change, or destroy — **without actually doing anything**. This is your safety check.

```bash
terraform plan
```

```
Plan: 3 to add, 0 to change, 0 to destroy.
  + aws_vpc.main          (will be created)
  + aws_subnet.public     (will be created)
  + aws_instance.web      (will be created)
```

### Step 3: `terraform apply`

Executes the plan and makes the real changes in your cloud account. Prompts for confirmation unless you pass `-auto-approve`.

```bash
terraform apply
# or
terraform apply -auto-approve
```

### Step 4: `terraform destroy`

Removes all resources that Terraform manages in the current configuration. Used to clean up environments, avoid charges, or start fresh.

```bash
terraform destroy
```

---

## A Minimal Working Example

Here is the smallest complete Terraform configuration that creates an S3 bucket in AWS:

```hcl
# versions.tf — specify required Terraform and provider versions
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure the AWS provider
provider "aws" {
  region = "us-east-1"
}

# Create an S3 bucket
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-terraform-demo-bucket-2024"

  tags = {
    Name        = "My Demo Bucket"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

To run it:
```bash
terraform init    # download AWS provider
terraform plan    # preview changes
terraform apply   # create the bucket
terraform destroy # delete the bucket when done
```

---

## Why Use Terraform? The Real Benefits

| Problem Without IaC          | Solution With Terraform           |
|------------------------------|-----------------------------------|
| "It works on my setup"       | Same config runs everywhere       |
| Manual steps take hours      | `terraform apply` takes minutes   |
| Can't audit what changed     | All changes tracked in Git        |
| Hard to reproduce prod       | Identical envs from same config   |
| Snowflake servers            | Cattle not pets — rebuild easily  |
| No disaster recovery plan    | Rebuild entire infra from files   |

---

## When to Use Terraform

**Use Terraform when:**
- You need to create cloud resources (EC2, S3, VPC, RDS, etc.)
- You want reproducible environments (dev, staging, prod)
- You are working in a team and need change history in Git
- You need to manage infrastructure across multiple cloud providers
- You want to automate infra creation in CI/CD pipelines

**Terraform is NOT the right tool for:**
- Configuring software inside servers (use Ansible for that)
- Deploying application code (use Docker, Kubernetes, or CI/CD pipelines)
- One-off quick tasks where clicking is faster

---

## Key Vocabulary

| Term       | Plain English                                           |
|------------|---------------------------------------------------------|
| HCL        | The language you write Terraform configs in             |
| Provider   | A plugin that talks to a specific cloud (e.g., AWS)     |
| Resource   | A real piece of infra (e.g., an EC2 instance)           |
| State      | A file Terraform uses to track what it has built        |
| Plan       | A preview of changes before they happen                 |
| Module     | A reusable group of Terraform resources                 |
| Backend    | Where Terraform stores its state file                   |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  IaC = describe infra in code, not clicks                       │
│  Terraform = HashiCorp's IaC tool using HCL                     │
│  Core workflow: init → plan → apply → destroy                   │
│  Benefits: reproducible, auditable, team-friendly, fast         │
│  Use for: cloud resources (AWS/Azure/GCP)                       │
│  Not for: app config, software installs (use Ansible)           │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Bash Interview Questions](../../02_Bash-Scripting/99_interview_master/bash_questions.md) &nbsp;|&nbsp; **Next:** [Terraform vs Others →](./terraform_vs_others.md)

**Related Topics:** [Installation](./installation.md) · [HCL Syntax](../02_hcl_basics/syntax.md) · [Providers](../03_providers_resources/providers.md)
