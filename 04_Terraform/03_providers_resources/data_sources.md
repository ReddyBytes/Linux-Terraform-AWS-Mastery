# Data Sources — Reading Existing Infrastructure

## The Research Librarian Analogy

Imagine you are writing a report and need to know the population of Tokyo. You don't create a new population — you look up the existing data in a library. The librarian (data source) retrieves the information for you, and you use it in your report.

In Terraform, a **data source** lets you look up information about existing infrastructure — things that already exist in AWS and that Terraform did not create. You read that data and use it to configure your resources.

Data sources are **read-only**. They never create, change, or delete anything.

---

## Resource vs Data Source

```
┌─────────────────────────────────────────────────────────────────┐
│            RESOURCE vs DATA SOURCE                              │
│                                                                 │
│  resource "aws_vpc" "main" { ... }                              │
│  ────────                                                       │
│  Creates a NEW VPC in AWS                                       │
│  Terraform OWNS this — manages its lifecycle                    │
│                                                                 │
│  data "aws_vpc" "existing" { ... }                              │
│  ────                                                           │
│  READS an EXISTING VPC from AWS                                 │
│  Terraform does NOT own this — just queries it                  │
│                                                                 │
│  When to use a data source:                                     │
│  - VPC created by another team/Terraform config                 │
│  - AMI IDs that change (always use latest Amazon Linux)         │
│  - Account ID, region, availability zones                       │
│  - Secrets from AWS Secrets Manager                             │
│  - IAM policy documents                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Source Syntax

```hcl
data  "<data_source_type>"  "<local_name>"  {
  # filter/lookup criteria
}

# Then reference it:
data.<data_source_type>.<local_name>.<attribute>
```

---

## Most Common Data Sources

### 1. aws_ami — Find the Latest AMI

AMI IDs are region-specific and change when AWS releases updates. Hard-coding an AMI ID means you are pinned to an old version. Use a data source to always get the latest:

```hcl
# Find the latest Amazon Linux 2023 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]   # official Amazon images

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Find the latest Ubuntu 22.04 LTS AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical's AWS account ID

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Use the AMI in a resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id   # always latest
  instance_type = "t3.micro"
}
```

### 2. aws_vpc — Look Up an Existing VPC

```hcl
# Find a VPC by tag
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["main-vpc"]
  }
}

# Find the default VPC
data "aws_vpc" "default" {
  default = true
}

# Use VPC ID in your resources
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = data.aws_vpc.main.id   # from data source
}

# Access other VPC attributes
output "vpc_cidr" {
  value = data.aws_vpc.main.cidr_block
}
```

### 3. aws_subnets — Find Existing Subnets

```hcl
# Find all private subnets in the VPC
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  filter {
    name   = "tag:Type"
    values = ["private"]
  }
}

# Use subnet IDs in a load balancer
resource "aws_lb" "app" {
  name               = "app-lb"
  internal           = true
  load_balancer_type = "application"
  subnets            = data.aws_subnets.private.ids   # list of subnet IDs
}
```

### 4. aws_availability_zones — Get AZs in Current Region

```hcl
# Get all available AZs in the current region
data "aws_availability_zones" "available" {
  state = "available"
}

# Create a subnet in each AZ
resource "aws_subnet" "public" {
  count = length(data.aws_availability_zones.available.names)

  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "public-${data.aws_availability_zones.available.names[count.index]}"
  }
}
```

### 5. aws_caller_identity — Get Your AWS Account ID

```hcl
# Find out who you are (account ID, user ARN, etc.)
data "aws_caller_identity" "current" {}

data "aws_region" "current" {}

# Use in IAM policies, bucket policies, etc.
resource "aws_s3_bucket_policy" "app" {
  bucket = aws_s3_bucket.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
      Action    = "s3:*"
      Resource  = "${aws_s3_bucket.app.arn}/*"
    }]
  })
}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "current_region" {
  value = data.aws_region.current.name
}
```

### 6. aws_iam_policy_document — Build IAM Policies

```hcl
# Build an IAM policy as a data source (cleaner than raw JSON)
data "aws_iam_policy_document" "ec2_s3_access" {
  statement {
    sid    = "AllowS3Access"
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject"
    ]
    resources = [
      "${aws_s3_bucket.app.arn}",
      "${aws_s3_bucket.app.arn}/*"
    ]
  }

  statement {
    sid       = "AllowListBucket"
    effect    = "Allow"
    actions   = ["s3:ListBucket"]
    resources = [aws_s3_bucket.app.arn]
  }
}

# Use the policy document in an IAM policy resource
resource "aws_iam_role_policy" "ec2_s3" {
  name   = "ec2-s3-access"
  role   = aws_iam_role.ec2.id
  policy = data.aws_iam_policy_document.ec2_s3_access.json
}
```

### 7. aws_secretsmanager_secret_version — Read Secrets Safely

```hcl
# Read a database password from Secrets Manager (not hardcoded!)
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "myapp/production/db-password"
}

resource "aws_db_instance" "main" {
  engine   = "postgres"
  username = "dbadmin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  # password is never in your code — only pulled at runtime
}
```

---

## Full Example: EC2 Using Data Sources

```hcl
# Look up latest Amazon Linux AMI
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# Look up existing VPC
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["main-vpc"]
  }
}

# Look up existing public subnets
data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Type"
    values = ["public"]
  }
}

# Now create EC2 using the looked-up values
resource "aws_instance" "app" {
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.micro"
  subnet_id     = data.aws_subnets.public.ids[0]

  tags = {
    Name = "app-server"
  }
}
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  data "type" "name" { } = read existing infra, never creates    │
│  Reference: data.type.name.attribute                            │
│                                                                 │
│  Common data sources:                                           │
│  aws_ami              → find latest AMI image                   │
│  aws_vpc              → look up existing VPC by filter/tag      │
│  aws_subnets          → find subnets in a VPC                   │
│  aws_availability_zones → list AZs in current region            │
│  aws_caller_identity  → get current AWS account ID             │
│  aws_iam_policy_document → build IAM policy JSON cleanly        │
│  aws_secretsmanager_secret_version → read secrets safely        │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Resources](./resources.md) &nbsp;|&nbsp; **Next:** [Variables →](../04_variables_outputs/variables.md)

**Related Topics:** [Resources](./resources.md) · [Variables](../04_variables_outputs/variables.md) · [EC2 with Terraform](../08_aws_with_terraform/ec2.md)
