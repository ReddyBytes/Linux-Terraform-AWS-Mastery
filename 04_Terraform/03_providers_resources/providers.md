# Terraform Providers — Plugins That Talk to the World

## The Translator Analogy

Imagine you want to order food from a restaurant in Japan, but you only speak English. You need a translator — someone who understands both languages and can relay your request to the kitchen and bring back the response.

A Terraform **provider** is exactly that translator. Terraform speaks HCL. AWS speaks its own API. The AWS provider is the translator between them. When you write `resource "aws_instance" "web"`, the AWS provider knows how to call the AWS EC2 API, handle the authentication, interpret the response, and tell Terraform what happened.

---

## What is a Provider?

A provider is a plugin that:
1. Knows how to talk to a specific API (AWS, Azure, GCP, GitHub, Datadog, etc.)
2. Exposes a set of **resources** you can create/manage
3. Exposes a set of **data sources** you can read from
4. Handles authentication to that API

```
┌──────────────────────────────────────────────────────────────────┐
│                   HOW PROVIDERS WORK                             │
│                                                                  │
│   Your HCL Code                                                  │
│   ┌──────────────────┐                                           │
│   │ resource         │                                           │
│   │ "aws_instance"   │─────► AWS Provider Plugin                 │
│   │ "web" { ... }    │       ┌──────────────────────┐            │
│   └──────────────────┘       │ Translates HCL to    │            │
│                              │ AWS API calls        │            │
│                              └──────────┬───────────┘            │
│                                         │ HTTPS API calls        │
│                                         ▼                        │
│                              ┌──────────────────────┐            │
│                              │     AWS Cloud         │            │
│                              │  EC2, S3, VPC, RDS   │            │
│                              └──────────────────────┘            │
└──────────────────────────────────────────────────────────────────┘
```

Terraform has providers for over 1,000 services. Browse them at https://registry.terraform.io/browse/providers

---

## Declaring Required Providers

Best practice is to declare all required providers in a `versions.tf` (or within your `terraform {}` block in `main.tf`):

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

The `source` is the provider's registry address: `<namespace>/<type>`. For HashiCorp-maintained providers, the namespace is `hashicorp`. Community providers use the author's namespace.

---

## Version Constraints Explained

Version constraints tell Terraform which provider versions are acceptable:

```
┌──────────────────────────────────────────────────────────────────┐
│                  VERSION CONSTRAINT OPERATORS                    │
│                                                                  │
│  = 5.0.0     Exact version only                                  │
│  != 5.0.0    Exclude this version                                │
│  > 5.0.0     Greater than                                        │
│  >= 5.0.0    Greater than or equal to                            │
│  < 6.0.0     Less than                                           │
│  <= 6.0.0    Less than or equal to                               │
│                                                                  │
│  ~> 5.0      Allow 5.x.x but NOT 6.x.x  (most common)           │
│              Same as >= 5.0.0, < 6.0.0                           │
│                                                                  │
│  ~> 5.0.0    Allow 5.0.x but NOT 5.1.0                           │
│              Same as >= 5.0.0, < 5.1.0                           │
└──────────────────────────────────────────────────────────────────┘
```

Best practices:
```hcl
# Use ~> for minor/patch flexibility
version = "~> 5.0"   # allows 5.1, 5.2, etc., blocks 6.0

# Pin exact version in production if you need total stability
version = "= 5.31.0"
```

---

## Configuring the AWS Provider

```hcl
# Basic configuration — uses credentials from ~/.aws/credentials or env vars
provider "aws" {
  region = "us-east-1"
}

# With a specific profile
provider "aws" {
  region  = "us-east-1"
  profile = "mycompany-prod"
}

# With explicit credentials (AVOID — never commit keys to Git!)
provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"        # NEVER do this
  secret_key = "wJalrXUtnFEMI/K7MDENG..."   # NEVER do this
}

# Better: use environment variables or instance profiles
# export AWS_ACCESS_KEY_ID="..."
# export AWS_SECRET_ACCESS_KEY="..."
# export AWS_DEFAULT_REGION="us-east-1"
```

### Commonly Used AWS Provider Configuration Options

```hcl
provider "aws" {
  region = "us-east-1"

  # Default tags applied to ALL resources created by this provider
  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Project     = "MyApp"
      Environment = var.environment
    }
  }

  # Skip validation of credentials (useful in testing)
  # skip_credentials_validation = true

  # Use a specific endpoint (useful for LocalStack testing)
  # endpoints {
  #   s3 = "http://localhost:4566"
  # }
}
```

---

## Provider Aliases — Multiple Regions or Accounts

Sometimes you need to create resources in multiple AWS regions or multiple AWS accounts in the same Terraform configuration. Use **provider aliases** for this.

```hcl
# Default provider (us-east-1)
provider "aws" {
  region = "us-east-1"
}

# Aliased provider for us-west-2
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Aliased provider for a different AWS account
provider "aws" {
  alias   = "shared_services"
  region  = "us-east-1"
  profile = "shared-services-account"
}

# Use the default provider (no alias needed)
resource "aws_s3_bucket" "east_bucket" {
  bucket = "my-east-bucket"
}

# Use the aliased provider
resource "aws_s3_bucket" "west_bucket" {
  provider = aws.west   # explicitly specify which provider to use
  bucket   = "my-west-bucket"
}

# Replicate data to both regions
resource "aws_s3_bucket_replication_configuration" "replication" {
  bucket = aws_s3_bucket.east_bucket.id
  # ...
}
```

---

## Using Multiple Providers Together

Real-world infrastructure often spans multiple providers:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
    github = {
      source  = "integrations/github"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}

provider "github" {
  token = var.github_token
}

# Create AWS resources
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# Create DNS record in Cloudflare pointing to AWS instance
resource "cloudflare_record" "web" {
  zone_id = var.cloudflare_zone_id
  name    = "app"
  value   = aws_instance.web.public_ip
  type    = "A"
}
```

---

## How terraform init Downloads Providers

When you run `terraform init`, Terraform reads your `required_providers` block and downloads the plugins to `.terraform/providers/`:

```bash
terraform init
```

```
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.31.0...
- Installed hashicorp/aws v5.31.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

The downloaded providers are stored in:
```
.terraform/
└── providers/
    └── registry.terraform.io/
        └── hashicorp/
            └── aws/
                └── 5.31.0/
                    └── linux_amd64/
                        └── terraform-provider-aws_v5.31.0
```

A `.terraform.lock.hcl` file is created (and should be committed to Git) to pin exact provider versions:

```hcl
# .terraform.lock.hcl — commit this to Git!
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abcdef...",   # checksum for verification
  ]
}
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  Provider = plugin that translates HCL to cloud API calls       │
│  Declare in: terraform { required_providers { ... } }           │
│  ~> 5.0 = allow 5.x but not 6.x (recommended constraint)        │
│  provider "aws" { region = "..." } configures the provider      │
│  default_tags = apply tags to all resources automatically        │
│  alias = use multiple regions or accounts simultaneously         │
│  terraform init = downloads declared providers                  │
│  .terraform.lock.hcl = pin exact versions, commit to Git        │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Expressions](../02_hcl_basics/expressions.md) &nbsp;|&nbsp; **Next:** [Resources →](./resources.md)

**Related Topics:** [Resources](./resources.md) · [Data Sources](./data_sources.md) · [Installation](../01_introduction/installation.md)
