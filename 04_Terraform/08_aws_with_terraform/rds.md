# RDS with Terraform — Managed Relational Databases

## The Restaurant Kitchen Analogy

Running your own database server is like building and managing your own restaurant kitchen: you buy the equipment, maintain the ovens, handle repairs, manage health inspections, and worry about electricity. It takes significant expertise and time.

Amazon RDS (Relational Database Service) is like renting a fully equipped, professionally managed kitchen. AWS handles hardware, OS patches, automated backups, failover, and monitoring. You just use the kitchen to cook.

With Terraform, you describe the kitchen you need, and AWS builds it to specification.

---

## RDS Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        VPC                                       │
│                                                                  │
│  ┌─────────────────┐   ┌─────────────────┐                       │
│  │  Private Subnet │   │  Private Subnet │                       │
│  │  us-east-1a     │   │  us-east-1b     │                       │
│  │                 │   │                 │                       │
│  │  ┌───────────┐  │   │  ┌───────────┐  │                       │
│  │  │ RDS       │  │   │  │ RDS       │  │                       │
│  │  │ Primary   │◄─┼───┼─►│ Standby   │  │                       │
│  │  │ (writer)  │  │   │  │ (Multi-AZ)│  │                       │
│  │  └─────┬─────┘  │   │  └───────────┘  │                       │
│  └────────┼────────┘   └─────────────────┘                       │
│           │                                                      │
│  ┌────────▼──────────────────────────────┐                       │
│  │  DB Subnet Group                      │                       │
│  │  (spans both private subnets)         │                       │
│  └───────────────────────────────────────┘                       │
│                                                                  │
│  ┌──────────────────┐                                            │
│  │  Security Group  │ ← only allows traffic from app servers     │
│  │  Port 5432 from  │                                            │
│  │  app-sg only     │                                            │
│  └──────────────────┘                                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## Complete RDS Configuration

### variables.tf

```hcl
variable "project"     { type = string }
variable "environment" { type = string }

variable "db_username" {
  type        = string
  description = "Database admin username"
  default     = "dbadmin"
}

variable "db_password" {
  type        = string
  description = "Database admin password"
  sensitive   = true   # never shown in logs or plan output
}

variable "db_instance_class" {
  type    = string
  default = "db.t3.micro"
}

variable "db_allocated_storage" {
  type    = number
  default = 20   # GB
}

variable "private_subnet_ids" {
  type = list(string)
}

variable "app_security_group_id" {
  type = string
}

variable "vpc_id" {
  type = string
}
```

### main.tf

```hcl
locals {
  name_prefix   = "${var.project}-${var.environment}"
  is_production = var.environment == "prod"
}

# ─────────────────────────────────────────────────────────
# DB Subnet Group — tells RDS which subnets to use
# Must span at least 2 AZs
# ─────────────────────────────────────────────────────────
resource "aws_db_subnet_group" "main" {
  name        = "${local.name_prefix}-db-subnet-group"
  subnet_ids  = var.private_subnet_ids   # must be in private subnets
  description = "Subnet group for ${local.name_prefix} RDS"

  tags = { Name = "${local.name_prefix}-db-subnet-group" }
}

# ─────────────────────────────────────────────────────────
# Security Group — controls who can talk to the database
# ─────────────────────────────────────────────────────────
resource "aws_security_group" "rds" {
  name        = "${local.name_prefix}-rds-sg"
  description = "Allow PostgreSQL from app servers only"
  vpc_id      = var.vpc_id

  ingress {
    description     = "PostgreSQL from app servers"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]   # only from app servers!
    # NOT from "0.0.0.0/0" — never expose databases to the internet
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${local.name_prefix}-rds-sg" }
}

# ─────────────────────────────────────────────────────────
# Parameter Group — tune database settings
# ─────────────────────────────────────────────────────────
resource "aws_db_parameter_group" "postgres14" {
  name        = "${local.name_prefix}-postgres14-params"
  family      = "postgres14"
  description = "Custom parameter group for ${local.name_prefix}"

  parameter {
    name  = "log_connections"
    value = "1"
  }

  parameter {
    name  = "log_disconnections"
    value = "1"
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"   # log queries taking longer than 1 second
  }

  tags = { Name = "${local.name_prefix}-postgres14-params" }
}

# ─────────────────────────────────────────────────────────
# RDS Instance
# ─────────────────────────────────────────────────────────
resource "aws_db_instance" "main" {
  identifier = "${local.name_prefix}-postgres"

  # Database engine
  engine         = "postgres"
  engine_version = "14"
  instance_class = var.db_instance_class   # "db.t3.micro" for dev

  # Storage
  allocated_storage     = var.db_allocated_storage   # 20 GB
  max_allocated_storage = 100    # allow autoscaling up to 100 GB
  storage_type          = "gp3"
  storage_encrypted     = true   # always encrypt database storage

  # Credentials — NEVER hardcode; use variables or Secrets Manager
  username = var.db_username
  password = var.db_password
  # In production, consider manage_master_user_password = true
  # to let AWS manage the password rotation in Secrets Manager

  # Database name created on first launch
  db_name = replace("${var.project}${var.environment}", "-", "")
  # e.g., "myappdev" or "myappprod"

  # Network
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false   # NEVER true in production

  # Availability and resilience
  multi_az               = local.is_production   # true in prod, false in dev
  availability_zone      = local.is_production ? null : "us-east-1a"

  # Backup
  backup_retention_period   = local.is_production ? 7 : 1   # days
  backup_window             = "03:00-04:00"   # UTC, low-traffic window
  maintenance_window        = "Mon:04:00-Mon:05:00"

  # Parameter group
  parameter_group_name = aws_db_parameter_group.postgres14.name

  # Deletion protection — prevents accidental delete in production
  deletion_protection = local.is_production ? true : false

  # When Terraform destroys this resource, take a final snapshot?
  skip_final_snapshot       = local.is_production ? false : true
  final_snapshot_identifier = local.is_production ? "${local.name_prefix}-final-snapshot-${formatdate("YYYY-MM-DD", timestamp())}" : null

  # Performance Insights (enhanced query monitoring)
  performance_insights_enabled = local.is_production
  performance_insights_retention_period = local.is_production ? 7 : null

  # Enhanced monitoring (OS-level metrics every 60 seconds)
  monitoring_interval = local.is_production ? 60 : 0
  monitoring_role_arn = local.is_production ? aws_iam_role.rds_monitoring[0].arn : null

  tags = { Name = "${local.name_prefix}-postgres" }
}

# Enhanced monitoring IAM role (production only)
resource "aws_iam_role" "rds_monitoring" {
  count = local.is_production ? 1 : 0
  name  = "${local.name_prefix}-rds-monitoring-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "monitoring.rds.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "rds_monitoring" {
  count      = local.is_production ? 1 : 0
  role       = aws_iam_role.rds_monitoring[0].name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}
```

### outputs.tf

```hcl
output "db_endpoint" {
  description = "RDS connection endpoint (hostname:port)"
  value       = aws_db_instance.main.endpoint
}

output "db_address" {
  description = "RDS hostname (without port)"
  value       = aws_db_instance.main.address
}

output "db_port" {
  description = "RDS port"
  value       = aws_db_instance.main.port
}

output "db_name" {
  description = "Database name"
  value       = aws_db_instance.main.db_name
}

output "db_username" {
  description = "Database admin username"
  value       = aws_db_instance.main.username
}
```

---

## Connecting from EC2 to RDS

After applying, from your EC2 instance in the same VPC:

```bash
# Get RDS endpoint
terraform output db_address
# → myapp-dev-postgres.abcdefghijkl.us-east-1.rds.amazonaws.com

# Install PostgreSQL client on Amazon Linux 2023
sudo dnf install -y postgresql15

# Connect to the database
psql -h $(terraform output -raw db_address) \
     -U dbadmin \
     -d myappdev \
     -p 5432

# Or using environment variables
export PGHOST=$(terraform output -raw db_address)
export PGUSER=dbadmin
export PGDATABASE=myappdev
psql
```

---

## Using AWS Secrets Manager for Passwords (Production Best Practice)

```hcl
# Let AWS manage the password automatically
resource "aws_db_instance" "main" {
  # ...
  manage_master_user_password = true   # AWS generates and rotates password
  # Remove the 'password' argument when using this option
}

# Retrieve the secret ARN
output "db_secret_arn" {
  value = aws_db_instance.main.master_user_secret[0].secret_arn
}

# In your app, read the secret at runtime:
# aws secretsmanager get-secret-value --secret-id <arn>
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  aws_db_subnet_group → spans ≥2 private subnets                 │
│  aws_security_group  → allow DB port from app SG only           │
│  aws_db_parameter_group → tune DB engine settings               │
│  aws_db_instance     → the database itself                      │
│                                                                 │
│  Key settings:                                                  │
│  storage_encrypted = true          (always!)                    │
│  publicly_accessible = false       (always for production!)     │
│  multi_az = true                   (production HA)              │
│  deletion_protection = true        (production safety)          │
│  backup_retention_period = 7       (production)                 │
│                                                                 │
│  Passwords: use sensitive variables or manage_master_user_password│
│  Never: hardcode passwords in .tf files                         │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← IAM](./iam.md) &nbsp;|&nbsp; **Next:** [Code Organization →](../09_best_practices/code_organization.md)

**Related Topics:** [VPC](./vpc.md) · [EC2](./ec2.md) · [IAM](./iam.md) · [Security Best Practices](../09_best_practices/security.md)
