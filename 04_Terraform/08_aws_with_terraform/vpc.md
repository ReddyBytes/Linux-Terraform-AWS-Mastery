# Building a VPC with Terraform — Complete Working Example

## The City Infrastructure Analogy

Before building a city, you need infrastructure: roads connecting neighborhoods, electricity grids, water systems, and controlled entry/exit points. Every building in the city depends on this foundation.

A VPC (Virtual Private Cloud) is the infrastructure foundation for your AWS workloads. It defines your private network space, how traffic flows in and out, and which resources can talk to each other. Everything else — EC2 instances, RDS databases, Lambda functions — lives inside a VPC.

---

## VPC Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        VPC: 10.0.0.0/16                                 │
│                                                                         │
│   ┌─────────────────────────┐   ┌─────────────────────────┐             │
│   │  Public Subnet          │   │  Public Subnet           │             │
│   │  us-east-1a             │   │  us-east-1b              │             │
│   │  10.0.1.0/24            │   │  10.0.2.0/24             │             │
│   │                         │   │                          │             │
│   │  [Web Servers]          │   │  [Web Servers]           │             │
│   │  [Load Balancer]        │   │  [Load Balancer]         │             │
│   └──────────┬──────────────┘   └──────────┬───────────────┘             │
│              │                             │                             │
│   ┌──────────▼─────────────────────────────▼───────────────┐            │
│   │               Route Table (Public)                      │            │
│   │  0.0.0.0/0  →  Internet Gateway                        │            │
│   └──────────────────────────┬──────────────────────────────┘            │
│                              │                                           │
│                    ┌─────────▼──────────┐                               │
│                    │  Internet Gateway  │ ←── Internet                  │
│                    └────────────────────┘                               │
│                                                                         │
│   ┌─────────────────────────┐   ┌─────────────────────────┐             │
│   │  Private Subnet         │   │  Private Subnet          │             │
│   │  us-east-1a             │   │  us-east-1b              │             │
│   │  10.0.11.0/24           │   │  10.0.12.0/24            │             │
│   │                         │   │                          │             │
│   │  [App Servers]          │   │  [App Servers]           │             │
│   │  [RDS Databases]        │   │  [RDS Databases]         │             │
│   └─────────────────────────┘   └─────────────────────────┘             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Complete VPC Terraform Configuration

### versions.tf

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Project     = var.project
      Environment = var.environment
    }
  }
}
```

### variables.tf

```hcl
variable "project" {
  type        = string
  description = "Project name for naming and tagging"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  default     = "dev"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "region" {
  type    = string
  default = "us-east-1"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

variable "public_subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  type    = list(string)
  default = ["10.0.11.0/24", "10.0.12.0/24"]
}

variable "enable_nat_gateway" {
  type    = bool
  default = true
  description = "Create NAT gateway(s) for private subnet outbound access"
}
```

### main.tf

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
}

# ─────────────────────────────────────────────────────────
# VPC
# ─────────────────────────────────────────────────────────
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true   # required for public DNS names
  enable_dns_support   = true

  tags = { Name = "${local.name_prefix}-vpc" }
}

# ─────────────────────────────────────────────────────────
# Internet Gateway — allows public subnets to reach internet
# ─────────────────────────────────────────────────────────
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${local.name_prefix}-igw" }
}

# ─────────────────────────────────────────────────────────
# Public Subnets — directly routable from the internet
# ─────────────────────────────────────────────────────────
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true   # instances get public IPs by default

  tags = {
    Name = "${local.name_prefix}-public-${var.availability_zones[count.index]}"
    Type = "public"
  }
}

# ─────────────────────────────────────────────────────────
# Private Subnets — not directly reachable from internet
# ─────────────────────────────────────────────────────────
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${local.name_prefix}-private-${var.availability_zones[count.index]}"
    Type = "private"
  }
}

# ─────────────────────────────────────────────────────────
# NAT Gateway — allows private subnets to reach internet
# (for software updates, API calls) without being reachable
# from the internet
# ─────────────────────────────────────────────────────────
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? length(var.availability_zones) : 0
  domain = "vpc"

  tags = {
    Name = "${local.name_prefix}-nat-eip-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? length(var.availability_zones) : 0

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id   # NAT gateways go IN public subnets

  tags = {
    Name = "${local.name_prefix}-nat-${var.availability_zones[count.index]}"
  }

  depends_on = [aws_internet_gateway.main]
}

# ─────────────────────────────────────────────────────────
# Route Tables
# ─────────────────────────────────────────────────────────

# Public route table — routes internet traffic through IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${local.name_prefix}-rt-public" }
}

# Associate all public subnets with the public route table
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private route tables — one per AZ, routes internet traffic through NAT
resource "aws_route_table" "private" {
  count  = var.enable_nat_gateway ? length(var.availability_zones) : 1
  vpc_id = aws_vpc.main.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.main[count.index].id
    }
  }

  tags = { Name = "${local.name_prefix}-rt-private-${count.index + 1}" }
}

# Associate all private subnets with private route tables
resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[var.enable_nat_gateway ? count.index : 0].id
}

# ─────────────────────────────────────────────────────────
# Security Group — web-facing traffic
# ─────────────────────────────────────────────────────────
resource "aws_security_group" "web" {
  name        = "${local.name_prefix}-web-sg"
  description = "Allow HTTP/HTTPS inbound, all outbound"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${local.name_prefix}-web-sg" }
}
```

### outputs.tf

```hcl
output "vpc_id"              { value = aws_vpc.main.id }
output "vpc_cidr"            { value = aws_vpc.main.cidr_block }
output "public_subnet_ids"   { value = aws_subnet.public[*].id }
output "private_subnet_ids"  { value = aws_subnet.private[*].id }
output "internet_gateway_id" { value = aws_internet_gateway.main.id }
output "web_security_group_id" { value = aws_security_group.web.id }
```

---

## Running It

```bash
terraform init
terraform plan -var="project=myapp" -var="environment=dev"
terraform apply -var="project=myapp" -var="environment=dev"
terraform destroy -var="project=myapp" -var="environment=dev"
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  aws_vpc           → the private network container              │
│  aws_internet_gateway → connects VPC to the internet            │
│  aws_subnet (public)  → subnets with direct internet access     │
│  aws_subnet (private) → subnets without direct internet access  │
│  aws_nat_gateway      → lets private subnets reach internet     │
│  aws_route_table      → rules for where traffic goes            │
│  aws_security_group   → firewall rules for resources            │
│                                                                 │
│  Public subnet: has IGW route, instances get public IPs         │
│  Private subnet: has NAT route (if enabled), no public IPs      │
│  NAT gateway: always lives in a PUBLIC subnet                   │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Environments](../07_workspaces/environments.md) &nbsp;|&nbsp; **Next:** [EC2 →](./ec2.md)

**Related Topics:** [EC2](./ec2.md) · [Security Groups](./iam.md) · [Module Registry VPC](../06_modules/module_registry.md)
