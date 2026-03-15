# HCL Syntax — Learning the Language of Terraform

## Every Language Has Grammar

When you learn Spanish, you first learn the alphabet, then words, then sentence structure. Only then can you write full paragraphs. HCL (HashiCorp Configuration Language) works the same way. Before you can write infrastructure code, you need to understand HCL's grammar.

HCL was designed to be readable by both humans and machines. It sits somewhere between JSON (which computers love but humans find tedious) and plain English (which humans love but computers struggle with).

---

## The Building Block: The Block

Everything in HCL is organized into **blocks**. A block is like a section in a form — it has a type, optional labels, and a body with arguments inside curly braces.

```
block_type "label1" "label2" {
  argument_name = value
}
```

Real examples:

```hcl
# A resource block — creates an AWS EC2 instance
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# A provider block — configures the AWS provider
provider "aws" {
  region = "us-east-1"
}

# A variable block — defines an input variable
variable "environment" {
  type    = string
  default = "dev"
}

# A terraform block — configures Terraform itself
terraform {
  required_version = ">= 1.5"
}
```

---

## Block Structure Anatomy

```
┌─────────────────────────────────────────────────────────────┐
│                    BLOCK ANATOMY                            │
│                                                             │
│   resource    "aws_instance"   "web_server"   {            │
│   ────────    ─────────────    ────────────   │            │
│   Block type  Label 1          Label 2        │            │
│               (resource type)  (local name)   │            │
│                                               │            │
│       ami           =   "ami-0c55b159cbfafe1f0"│            │
│       ─────────────     ────────────────────── │            │
│       Argument name     Argument value         │            │
│                                               }            │
│                                               └── Block body│
└─────────────────────────────────────────────────────────────┘
```

---

## Arguments and Values

Arguments are key-value pairs inside a block. The value can be a literal (hardcoded), a reference to another resource, or a function call.

```hcl
resource "aws_instance" "app" {
  # String literal
  ami = "ami-0c55b159cbfafe1f0"

  # Number literal
  monitoring = true

  # Boolean
  ebs_optimized = false

  # Reference to a variable
  instance_type = var.instance_type

  # Reference to another resource's attribute
  subnet_id = aws_subnet.public.id

  # Function call
  user_data = file("user_data.sh")

  # Multiline string (heredoc)
  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
  EOF
}
```

---

## References: Connecting Resources Together

References are how you make resources talk to each other. Instead of hardcoding an ID (which you don't know until the resource exists), you reference the resource and Terraform resolves the value automatically.

```
┌─────────────────────────────────────────────────────────────┐
│               REFERENCE SYNTAX                              │
│                                                             │
│   <resource_type>.<local_name>.<attribute>                  │
│                                                             │
│   aws_subnet.public.id                                      │
│   aws_security_group.web.id                                 │
│   aws_vpc.main.cidr_block                                   │
│                                                             │
│   var.<variable_name>                                       │
│   var.environment                                           │
│   var.instance_type                                         │
│                                                             │
│   local.<local_name>                                        │
│   local.common_tags                                         │
│                                                             │
│   data.<data_source_type>.<name>.<attribute>                │
│   data.aws_ami.ubuntu.id                                    │
└─────────────────────────────────────────────────────────────┘
```

Example showing references in practice:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id          # reference to vpc above
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  ami       = "ami-0c55b159cbfafe1f0"
  subnet_id = aws_subnet.public.id      # reference to subnet above
}
```

Terraform automatically creates resources in the correct order because it sees that `aws_subnet.public` depends on `aws_vpc.main` (it references its `.id`).

---

## Comments

```hcl
# This is a single-line comment (most common)

// This is also a single-line comment (less common)

/*
  This is a
  multi-line comment
*/
```

---

## Nested Blocks

Some arguments are themselves blocks (nested blocks). They do not have a `=` sign — just a block name and curly braces:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Nested block — no equals sign
  root_block_device {
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true
  }

  # Another nested block
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

---

## String Interpolation

String interpolation lets you embed expressions inside strings using `${}`:

```hcl
variable "environment" {
  default = "production"
}

resource "aws_s3_bucket" "logs" {
  # Interpolation: embed a variable in a string
  bucket = "myapp-${var.environment}-logs"
  # Result: "myapp-production-logs"
}

resource "aws_instance" "web" {
  tags = {
    Name = "web-server-${var.environment}-01"
    # Result: "web-server-production-01"
  }
}
```

---

## Built-in Functions Overview

Terraform has dozens of built-in functions. Here are the most common categories:

```hcl
# String functions
upper("hello")              # → "HELLO"
lower("HELLO")              # → "hello"
trimspace("  hi  ")         # → "hi"
replace("hello", "l", "r")  # → "herro"
split(",", "a,b,c")         # → ["a", "b", "c"]
join("-", ["a", "b", "c"])  # → "a-b-c"
format("Hello, %s!", "world") # → "Hello, world!"

# Numeric functions
max(1, 2, 3)     # → 3
min(1, 2, 3)     # → 1
abs(-5)          # → 5
ceil(1.2)        # → 2
floor(1.9)       # → 1

# Collection functions
length(["a", "b", "c"])       # → 3
contains(["a", "b"], "a")     # → true
distinct(["a", "a", "b"])     # → ["a", "b"]
flatten([["a"], ["b", "c"]])  # → ["a", "b", "c"]
merge({a=1}, {b=2})           # → {a=1, b=2}

# File functions
file("user_data.sh")          # read a file's contents
filebase64("script.sh")       # read file, base64 encode it
templatefile("tmpl.tpl", {})  # render a template file

# Encoding functions
base64encode("hello")   # → "aGVsbG8="
jsonencode({a = 1})     # → "{\"a\":1}"
jsondecode("{\"a\":1}") # → {a = 1}
```

Example using functions in a real resource:

```hcl
variable "project" {
  default = "myapp"
}

variable "environment" {
  default = "dev"
}

locals {
  # Build a consistent name prefix using functions
  name_prefix = "${lower(var.project)}-${lower(var.environment)}"
}

resource "aws_s3_bucket" "data" {
  bucket = "${local.name_prefix}-data-${formatdate("YYYY", timestamp())}"
  # Result: "myapp-dev-data-2024"
}
```

---

## Whitespace and Formatting

HCL is whitespace-insensitive (indentation does not matter technically), but the community follows consistent style. Use `terraform fmt` to auto-format your code:

```bash
terraform fmt          # format all .tf files in current dir
terraform fmt -recursive  # format all subdirectories too
terraform fmt -check      # exit 1 if files need formatting (for CI)
```

Standard style:
- 2-space indentation
- Align `=` signs in blocks of related arguments
- One blank line between resource blocks

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  Block = type + labels + { arguments }                          │
│  Arguments = key = value pairs inside blocks                    │
│  References: resource_type.name.attr  |  var.name  |  local.x  │
│  String interpolation: "hello-${var.name}"                      │
│  Nested blocks: no = sign, just { }                             │
│  Functions: upper(), lower(), join(), length(), merge(), etc.   │
│  terraform fmt: auto-formats your code                          │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Installation](../01_introduction/installation.md) &nbsp;|&nbsp; **Next:** [Data Types →](./data_types.md)

**Related Topics:** [Data Types](./data_types.md) · [Expressions](./expressions.md) · [Variables](../04_variables_outputs/variables.md)
