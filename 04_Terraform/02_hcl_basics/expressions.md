# HCL Expressions — Making Your Code Dynamic and Smart

## Static vs Dynamic Configuration

Imagine you are writing a menu for a restaurant. You could print a fixed menu with the exact price of every item. But prices change, seasonal items come and go, and different locations have different offerings. A smarter approach is a template with placeholders that get filled in based on context.

HCL expressions are that template system. They let you write configuration that adapts based on variables, conditions, and computed values — instead of hardcoding every detail.

---

## String Interpolation

You have already seen this, but it is worth a thorough look. Embed any expression inside a string using `${}`:

```hcl
variable "app_name"    { default = "myapp" }
variable "environment" { default = "prod" }
variable "region"      { default = "us-east-1" }

locals {
  # Combine strings
  bucket_name = "${var.app_name}-${var.environment}-data"
  # → "myapp-prod-data"

  # Use functions inside interpolation
  upper_env = "${upper(var.environment)}"
  # → "PROD"

  # Use resource attributes
  instance_url = "https://${aws_instance.web.public_ip}"
}
```

---

## Conditional Expressions

Like a ternary operator in most languages: `condition ? true_value : false_value`

```hcl
variable "environment" { default = "dev" }
variable "is_production" { default = false }

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"

  # Pick instance type based on environment
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
  #               ─────────────────────────  ─────────────  ──────────
  #               condition                  if true        if false

  # Enable monitoring only in production
  monitoring = var.is_production ? true : false

  # Conditional disk size
  root_block_device {
    volume_size = var.is_production ? 100 : 20
    encrypted   = var.is_production ? true : false
  }
}

# Conditional resource creation using count
resource "aws_eip" "web" {
  # Create elastic IP only in production
  count    = var.is_production ? 1 : 0
  instance = aws_instance.web.id
}
```

---

## For Expressions

`for` expressions transform one collection into another. Like a `map()` or `filter()` function in other languages.

### Basic For Expression (List → List)

```hcl
variable "names" {
  default = ["alice", "bob", "charlie"]
}

locals {
  # Transform: make all names uppercase
  upper_names = [for name in var.names : upper(name)]
  # → ["ALICE", "BOB", "CHARLIE"]

  # Transform: create greeting strings
  greetings = [for name in var.names : "Hello, ${name}!"]
  # → ["Hello, alice!", "Hello, bob!", "Hello, charlie!"]
}
```

### For Expression with Filter

```hcl
variable "users" {
  default = ["alice", "bob", "admin_user", "admin_bob"]
}

locals {
  # Filter: keep only items where condition is true
  admin_users = [for u in var.users : u if startswith(u, "admin")]
  # → ["admin_user", "admin_bob"]

  regular_users = [for u in var.users : u if !startswith(u, "admin")]
  # → ["alice", "bob"]
}
```

### For Expression (List → Map)

Use `{}` instead of `[]` to produce a map:

```hcl
variable "environments" {
  default = ["dev", "staging", "prod"]
}

locals {
  # Create a map of {env = bucket_name}
  bucket_names = {for env in var.environments : env => "${var.app_name}-${env}-data"}
  # → {dev = "myapp-dev-data", staging = "myapp-staging-data", ...}
}
```

### For Expression Over a Map

```hcl
variable "tags" {
  default = {
    Environment = "prod"
    Team        = "platform"
    CostCenter  = "eng-001"
  }
}

locals {
  # Iterate over map: k = key, v = value
  tag_strings = [for k, v in var.tags : "${k}=${v}"]
  # → ["Environment=prod", "Team=platform", "CostCenter=eng-001"]
}
```

---

## count — Create Multiple Resources with a Number

`count` is a meta-argument that tells Terraform to create N copies of a resource. Access the current index with `count.index`.

```hcl
variable "subnet_count" {
  default = 3
}

resource "aws_subnet" "public" {
  count = var.subnet_count   # creates 3 subnets: [0], [1], [2]

  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  # 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24

  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
    # "public-subnet-1", "public-subnet-2", "public-subnet-3"
  }
}

# Reference a specific instance
output "first_subnet_id" {
  value = aws_subnet.public[0].id
}

# Reference all instances
output "all_subnet_ids" {
  value = aws_subnet.public[*].id   # splat expression
}
```

**count vs for_each:** `count` uses numeric indices. If you remove item `[1]` from a count=3 list, Terraform renumbers all subsequent items and destroys/recreates them. This is the main weakness of `count`.

---

## for_each — Create Multiple Resources from a Map or Set

`for_each` creates one resource instance per map key or set element. Resources are identified by key, not index — so removing one item does not affect others.

```hcl
variable "buckets" {
  default = {
    logs    = "us-east-1"
    backups = "us-west-2"
    assets  = "us-east-1"
  }
}

resource "aws_s3_bucket" "app" {
  for_each = var.buckets   # one bucket per map entry

  bucket = "myapp-${each.key}"   # each.key = "logs", "backups", "assets"

  tags = {
    Name   = "myapp-${each.key}"
    Region = each.value    # each.value = the region string
  }
}

# Reference a specific bucket
output "logs_bucket_arn" {
  value = aws_s3_bucket.app["logs"].arn
}
```

With a set (use `toset()` to convert a list):

```hcl
variable "environments" {
  default = ["dev", "staging", "prod"]
}

resource "aws_iam_role" "app" {
  for_each = toset(var.environments)

  name = "app-role-${each.value}"   # each.key == each.value for sets
}
```

---

## count vs for_each — Decision Guide

```
┌──────────────────────────────────────────────────────────────────┐
│              count vs for_each                                   │
├─────────────────────┬────────────────────────────────────────────┤
│      count          │          for_each                          │
├─────────────────────┼────────────────────────────────────────────┤
│ Use when count is   │ Use when you have a map/set                │
│ a simple number     │ of named items                             │
│                     │                                            │
│ Resources named:    │ Resources named:                           │
│ resource[0]         │ resource["key"]                            │
│ resource[1]         │ resource["other-key"]                      │
│                     │                                            │
│ Removing index 1    │ Removing "key" only                        │
│ reshuffles all      │ removes that one resource                  │
│ subsequent indexes  │                                            │
│                     │                                            │
│ Good for: identical │ Good for: named resources that             │
│ copies (3 web nodes)│ might change over time                     │
└─────────────────────┴────────────────────────────────────────────┘
```

---

## Splat Expressions

A splat expression (`[*]`) extracts a single attribute from all instances of a resource created with `count`.

```hcl
resource "aws_instance" "web" {
  count = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# Without splat: tedious
output "ips_verbose" {
  value = [aws_instance.web[0].public_ip, aws_instance.web[1].public_ip, aws_instance.web[2].public_ip]
}

# With splat: clean
output "all_public_ips" {
  value = aws_instance.web[*].public_ip
  # → ["1.2.3.4", "5.6.7.8", "9.10.11.12"]
}

output "all_instance_ids" {
  value = aws_instance.web[*].id
}
```

---

## Dynamic Blocks

`dynamic` blocks generate repeated nested blocks based on a collection. This avoids copy-pasting the same block structure many times.

Without dynamic blocks (repetitive):
```hcl
resource "aws_security_group" "web" {
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

With a dynamic block (clean and scalable):
```hcl
variable "ingress_rules" {
  default = [
    { port = 80,   cidr = "0.0.0.0/0" },
    { port = 443,  cidr = "0.0.0.0/0" },
    { port = 8080, cidr = "0.0.0.0/0" },
  ]
}

resource "aws_security_group" "web" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  String interpolation: "hello-${var.name}"                      │
│  Conditional: condition ? value_if_true : value_if_false        │
│  for expression: [for x in list : transform(x) if condition]    │
│  count: create N copies, referenced by [0], [1], [2]            │
│  for_each: create copies from map/set, referenced by ["key"]    │
│  Splat [*]: extract attr from all count instances               │
│  dynamic: generate repeated nested blocks from a collection     │
│                                                                 │
│  Rule: use for_each over count when items have unique names     │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Data Types](./data_types.md) &nbsp;|&nbsp; **Next:** [Providers →](../03_providers_resources/providers.md)

**Related Topics:** [Data Types](./data_types.md) · [Variables](../04_variables_outputs/variables.md) · [Locals](../04_variables_outputs/locals.md)
