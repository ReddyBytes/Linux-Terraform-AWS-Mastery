# Terraform Resources — Creating and Managing Infrastructure

## The Factory Order Form Analogy

When a factory receives an order, it needs specific details: what to build, what materials to use, how many, and any special configurations. The factory does not guess — it follows the order form precisely.

In Terraform, a **resource block** is that order form. It tells Terraform exactly what piece of infrastructure to create, what type it is, and how to configure it. Once you submit the order (run `terraform apply`), the provider factory builds it.

---

## The Resource Block Anatomy

```
resource  "aws_instance"    "web_server"   {
  ────────  ──────────────   ────────────   │
  keyword   Resource type    Local name     │
            (provider_type)  (unique ID     │
                             in your code)  │
                                            │
    ami           = "ami-0c55b159cbfafe1f0" │
    instance_type = "t3.micro"              │
                                           }
```

The resource type `aws_instance` breaks down as:
- `aws` = the provider
- `instance` = the resource type within that provider

The local name `web_server` is how YOU refer to this resource elsewhere in your code. It has no effect on the actual resource name in AWS (use tags for that).

---

## A Complete Annotated Resource Example

```hcl
resource "aws_instance" "web_server" {
  # Required: which OS image to use
  ami = data.aws_ami.amazon_linux.id

  # Required: how powerful the machine should be
  instance_type = var.instance_type  # e.g., "t3.micro"

  # Which network subnet to place it in
  subnet_id = aws_subnet.public.id

  # Attach security groups (firewall rules)
  vpc_security_group_ids = [aws_security_group.web.id]

  # SSH key for access
  key_name = aws_key_pair.deployer.key_name

  # Startup script — runs when instance first boots
  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html
  EOF

  # Nested block for root disk configuration
  root_block_device {
    volume_size           = 20
    volume_type           = "gp3"
    encrypted             = true
    delete_on_termination = true
  }

  # Tags — metadata attached to the resource in AWS
  tags = {
    Name        = "web-server-prod-01"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

---

## Meta-Arguments

Meta-arguments are special arguments that work on ANY resource, regardless of provider. They control resource behavior at the Terraform level.

### 1. count — Create Multiple Copies

```hcl
resource "aws_instance" "web" {
  count = 3   # creates 3 identical instances

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server-${count.index + 1}"
    # → "web-server-1", "web-server-2", "web-server-3"
  }
}

# Reference: aws_instance.web[0], aws_instance.web[1], aws_instance.web[2]
```

### 2. for_each — Create Copies from a Map or Set

```hcl
variable "web_servers" {
  default = {
    frontend = "t3.micro"
    backend  = "t3.small"
    worker   = "t3.medium"
  }
}

resource "aws_instance" "app" {
  for_each = var.web_servers

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value   # t3.micro, t3.small, t3.medium

  tags = {
    Name = "app-${each.key}"
    # → "app-frontend", "app-backend", "app-worker"
  }
}

# Reference: aws_instance.app["frontend"], aws_instance.app["backend"]
```

### 3. depends_on — Explicit Dependencies

Terraform automatically detects dependencies through references. But sometimes a dependency is not captured in a reference. Use `depends_on` to declare it explicitly.

```hcl
resource "aws_iam_role_policy" "app_policy" {
  name   = "app-policy"
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.app.json
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # This instance needs the IAM policy to exist before it boots
  # There's no direct reference, so use depends_on
  depends_on = [aws_iam_role_policy.app_policy]
}
```

### 4. lifecycle — Control Resource Replacement Behavior

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  lifecycle {
    # Create new resource before destroying old one
    # Useful for zero-downtime replacements
    create_before_destroy = true

    # Prevent Terraform from destroying this resource
    # (even with terraform destroy)
    prevent_destroy = true

    # Ignore changes to these attributes after creation
    # Useful when something outside Terraform updates the resource
    ignore_changes = [
      tags,
      user_data,   # don't replace instance if user_data changes
    ]

    # Validate conditions after apply
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP"
    }
  }
}
```

---

## Resource Addressing

Every resource in Terraform has a unique address. You use this address to reference the resource in other resources, in CLI commands, and in plan output.

```
┌──────────────────────────────────────────────────────────────────┐
│                  RESOURCE ADDRESSING                             │
│                                                                  │
│  Single resource:                                                │
│  aws_instance.web_server                                         │
│  ─────────────.────────────                                      │
│  type          local name                                        │
│                                                                  │
│  With count:                                                     │
│  aws_instance.web[0]                                             │
│  aws_instance.web[1]                                             │
│                                                                  │
│  With for_each:                                                  │
│  aws_instance.app["frontend"]                                    │
│  aws_instance.app["backend"]                                     │
│                                                                  │
│  In a module:                                                    │
│  module.networking.aws_vpc.main                                  │
└──────────────────────────────────────────────────────────────────┘
```

Use these addresses in CLI commands:
```bash
# Show a specific resource
terraform state show aws_instance.web_server

# Target a specific resource in plan/apply
terraform plan -target=aws_instance.web_server
terraform apply -target=aws_instance.web_server

# Destroy a specific resource
terraform destroy -target=aws_instance.web[0]
```

---

## Importing Existing Resources

If you have resources that were created manually (before you had Terraform), you can bring them under Terraform management with `terraform import`.

```bash
# Import an existing EC2 instance
terraform import aws_instance.web_server i-1234567890abcdef0

# Import an existing S3 bucket
terraform import aws_s3_bucket.my_bucket my-existing-bucket-name

# Import an existing security group
terraform import aws_security_group.web sg-12345678
```

**Workflow for importing:**

1. Write the resource block in your `.tf` file first
2. Run `terraform import` to link the existing resource to your code
3. Run `terraform plan` — it should show no changes if your code matches reality
4. Adjust your code until `plan` shows no changes

Modern Terraform (v1.5+) also supports an `import` block for importing as code:

```hcl
# Import block (Terraform 1.5+)
import {
  id = "i-1234567890abcdef0"
  to = aws_instance.web_server
}
```

---

## Practical Example: VPC with Resources

```hcl
# 1. VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "main-vpc" }
}

# 2. Internet Gateway (depends on VPC)
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id   # reference creates implicit dependency
  tags   = { Name = "main-igw" }
}

# 3. Subnet (depends on VPC)
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1a"
  tags                    = { Name = "public-subnet-1a" }
}

# 4. Security Group (depends on VPC)
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# 5. EC2 Instance (depends on subnet + security group)
resource "aws_instance" "web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  tags                   = { Name = "web-server" }
}
```

Terraform builds this in the correct order automatically.

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  resource "type" "name" { ... } = declares infrastructure       │
│  Meta-arguments (work on any resource):                         │
│    count      = create N copies, indexed [0], [1], [2]          │
│    for_each   = create from map/set, indexed ["key"]            │
│    depends_on = explicit dependency declaration                 │
│    lifecycle  = control create/destroy/ignore behavior          │
│  Resource address: aws_instance.name or aws_instance.name[0]   │
│  terraform import = bring existing resources under management   │
│  Implicit deps: Terraform detects from references automatically │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Providers](./providers.md) &nbsp;|&nbsp; **Next:** [Data Sources →](./data_sources.md)

**Related Topics:** [Providers](./providers.md) · [Data Sources](./data_sources.md) · [Expressions](../02_hcl_basics/expressions.md)
