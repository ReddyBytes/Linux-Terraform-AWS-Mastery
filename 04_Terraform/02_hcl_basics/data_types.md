# HCL Data Types — Storing and Shaping Your Values

## The Container Analogy

Think of data types like different containers in your kitchen. A glass holds liquid. A plate holds solid food. A bag holds loose items. A filing cabinet holds folders of documents. Each container is designed for a specific kind of content.

In HCL, data types tell Terraform what kind of value to expect and how to work with it. Using the wrong type causes errors, just like pouring water into a paper bag.

---

## The Seven Core Data Types

```
┌─────────────────────────────────────────────────────────────────┐
│                    HCL DATA TYPES                               │
│                                                                 │
│   PRIMITIVE (single values)                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│   │  string  │  │  number  │  │   bool   │                     │
│   │ "hello"  │  │  42      │  │  true    │                     │
│   └──────────┘  └──────────┘  └──────────┘                     │
│                                                                 │
│   COLLECTION (multiple values)                                  │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│   │   list   │  │   set    │  │   map    │                     │
│   │ ordered  │  │ unique   │  │ key=value│                     │
│   └──────────┘  └──────────┘  └──────────┘                     │
│                                                                 │
│   STRUCTURAL (mixed values)                                     │
│   ┌──────────┐  ┌──────────┐                                   │
│   │  object  │  │  tuple   │                                   │
│   │ schema   │  │ ordered  │                                   │
│   │ required │  │ mixed    │                                   │
│   └──────────┘  └──────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. String

A string is a sequence of characters wrapped in double quotes.

```hcl
variable "environment" {
  type    = string
  default = "production"
}

variable "bucket_name" {
  type    = string
  default = "my-app-data"
}

# Multi-line string using heredoc
variable "description" {
  type = string
  default = <<-EOT
    This is a
    multi-line description.
  EOT
}

# Using strings
resource "aws_s3_bucket" "app" {
  bucket = "${var.bucket_name}-${var.environment}"
  # Result: "my-app-data-production"
}
```

---

## 2. Number

Numbers in HCL are either whole integers or decimals. No quotes around them.

```hcl
variable "instance_count" {
  type    = number
  default = 3
}

variable "disk_size_gb" {
  type    = number
  default = 100
}

variable "min_threshold" {
  type    = number
  default = 0.75   # decimal supported
}

# Using numbers
resource "aws_autoscaling_group" "app" {
  min_size         = 1
  max_size         = var.instance_count * 2    # math works: result is 6
  desired_capacity = var.instance_count        # 3
}
```

---

## 3. Bool

A boolean is either `true` or `false`. No quotes. Used for toggles and flags.

```hcl
variable "enable_monitoring" {
  type    = bool
  default = true
}

variable "is_production" {
  type    = bool
  default = false
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  monitoring    = var.enable_monitoring   # true/false
  ebs_optimized = var.is_production       # false

  root_block_device {
    encrypted = true   # bool literal
  }
}
```

---

## 4. List

A list is an ordered collection of values of the same type. Use `[]` syntax. Items can repeat. Access by index (zero-based).

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "allowed_ports" {
  type    = list(number)
  default = [80, 443, 8080]
}

# Access items by index
locals {
  first_az  = var.availability_zones[0]   # "us-east-1a"
  second_az = var.availability_zones[1]   # "us-east-1b"
}

# Use list in a resource
resource "aws_subnet" "public" {
  count             = length(var.availability_zones)   # 3
  vpc_id            = aws_vpc.main.id
  availability_zone = var.availability_zones[count.index]
  cidr_block        = "10.0.${count.index}.0/24"
}
```

---

## 5. Set

A set is like a list but **unordered** and **all items must be unique**. Use when you need uniqueness and order does not matter.

```hcl
variable "allowed_cidrs" {
  type    = set(string)
  default = ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]
}

# Sets are commonly used in security group rules
resource "aws_security_group" "internal" {
  name = "internal-sg"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = tolist(var.allowed_cidrs)   # convert set → list
  }
}

# Difference: list can have duplicates, set cannot
# ["a", "a", "b"] is valid list
# toset(["a", "a", "b"]) = {"a", "b"}  — duplicates removed
```

---

## 6. Map

A map is a collection of key-value pairs where all values are the same type. Like a dictionary or lookup table.

```hcl
variable "instance_types" {
  type = map(string)
  default = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }
}

variable "ami_by_region" {
  type = map(string)
  default = {
    "us-east-1" = "ami-0c55b159cbfafe1f0"
    "us-west-2" = "ami-0892d3c7ee96c0bf7"
    "eu-west-1" = "ami-0fc970315c2d38f01"
  }
}

# Look up a value from a map
resource "aws_instance" "app" {
  instance_type = var.instance_types["prod"]        # "t3.large"
  ami           = var.ami_by_region["us-east-1"]    # the AMI ID

  # Or use lookup() with a default:
  instance_type = lookup(var.instance_types, var.environment, "t3.micro")
}
```

---

## 7. Object

An object is like a map, but each key can have a **different type**. It has a fixed schema — you define the structure.

```hcl
variable "database_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    port           = number
    multi_az       = bool
  })
  default = {
    engine         = "postgres"
    engine_version = "14.7"
    instance_class = "db.t3.medium"
    port           = 5432
    multi_az       = false
  }
}

# Access object attributes with dot notation
resource "aws_db_instance" "main" {
  engine         = var.database_config.engine
  engine_version = var.database_config.engine_version
  instance_class = var.database_config.instance_class
  port           = var.database_config.port
  multi_az       = var.database_config.multi_az
}
```

---

## 8. Tuple

A tuple is an ordered collection where items can be **different types**. Like a fixed-size list with mixed types. Less commonly used than list or object.

```hcl
# A tuple: [string, number, bool]
variable "server_spec" {
  type    = tuple([string, number, bool])
  default = ["t3.micro", 20, true]
  #           type       disk_gb  encrypted
}

locals {
  instance_type = var.server_spec[0]   # "t3.micro"
  disk_gb       = var.server_spec[1]   # 20
  encrypted     = var.server_spec[2]   # true
}
```

---

## Type Conversion Functions

When you need to convert between types:

```hcl
# tostring() — convert to string
tostring(42)         # → "42"
tostring(true)       # → "true"

# tonumber() — convert to number
tonumber("42")       # → 42

# tobool() — convert to bool
tobool("true")       # → true

# tolist() — convert set/tuple to list
tolist(toset(["a", "b", "c"]))   # → ["a", "b", "c"]

# toset() — convert list to set (removes duplicates, loses order)
toset(["a", "b", "a"])           # → {"a", "b"}

# tomap() — convert object to map (all values must be same type)
tomap({name = "alice", role = "admin"})   # → map(string)
```

---

## Type Constraints in Variables

When you declare a variable, specifying the type lets Terraform validate input:

```hcl
# Simple type constraints
variable "name"    { type = string }
variable "count"   { type = number }
variable "enabled" { type = bool }

# Collection type constraints
variable "zones"   { type = list(string) }
variable "ports"   { type = set(number) }
variable "tags"    { type = map(string) }

# Complex type constraints
variable "config" {
  type = object({
    name    = string
    size    = number
    enabled = bool
    tags    = map(string)
  })
}

# any — accepts any type (avoid in production)
variable "anything" { type = any }
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  string  = text in double quotes: "hello"                       │
│  number  = numeric value: 42 or 3.14                            │
│  bool    = true or false                                        │
│  list    = ordered, duplicates allowed: ["a", "b", "a"]         │
│  set     = unordered, unique only: {"a", "b"}                   │
│  map     = key-value, same type values: {dev = "t3.micro"}      │
│  object  = key-value, mixed types: {name=string, port=number}   │
│  tuple   = ordered, mixed types: ["t3.micro", 20, true]         │
│                                                                 │
│  Conversion: tostring() tolist() toset() tomap() tonumber()     │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← HCL Syntax](./syntax.md) &nbsp;|&nbsp; **Next:** [Expressions →](./expressions.md)

**Related Topics:** [HCL Syntax](./syntax.md) · [Expressions](./expressions.md) · [Variables](../04_variables_outputs/variables.md)
