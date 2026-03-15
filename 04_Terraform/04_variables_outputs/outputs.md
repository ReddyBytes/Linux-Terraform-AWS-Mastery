# Terraform Outputs — Exposing Useful Values

## The Receipt Analogy

When you place an order at a restaurant, you get a receipt that shows what you ordered, how much it cost, and your order number. The receipt is not the food itself — it is useful information about the order that you can take away and use later (claim your food, report an issue, track expenses).

Terraform **outputs** work the same way. After `terraform apply` creates your infrastructure, outputs are the receipt — they surface important values like IP addresses, DNS names, resource IDs, and ARNs so you can use them outside of Terraform.

---

## The Output Block

```hcl
output "web_server_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP address of the web server"
}
```

After `terraform apply`:
```
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

web_server_ip = "54.210.167.204"
```

---

## Why Outputs Matter

```
┌──────────────────────────────────────────────────────────────────┐
│                   WHY OUTPUTS EXIST                              │
│                                                                  │
│  1. HUMAN CONSUMPTION                                            │
│     After applying, see the IP/URL/ID of what was created        │
│     No need to log into AWS console to find the value            │
│                                                                  │
│  2. SCRIPT/CI INTEGRATION                                        │
│     terraform output -raw web_server_ip                          │
│     Pipe the value into shell scripts, Ansible, deploy tools     │
│                                                                  │
│  3. MODULE COMMUNICATION                                         │
│     Child module outputs values → root module uses them          │
│     The only way to pass values OUT of a module                  │
│                                                                  │
│  4. REMOTE STATE DATA SOURCE                                     │
│     Another Terraform config can read your outputs via           │
│     data "terraform_remote_state" { }                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## Common Output Examples

```hcl
# EC2 instance details
output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IPv4 address"
  value       = aws_instance.web.public_ip
}

output "instance_public_dns" {
  description = "Public DNS hostname"
  value       = aws_instance.web.public_dns
}

# S3 bucket details
output "bucket_name" {
  value = aws_s3_bucket.app.id
}

output "bucket_arn" {
  value = aws_s3_bucket.app.arn
}

# VPC and networking
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = [for s in aws_subnet.private : s.id]
}

# RDS
output "rds_endpoint" {
  description = "RDS connection endpoint (host:port)"
  value       = aws_db_instance.main.endpoint
}

output "rds_address" {
  description = "RDS hostname only"
  value       = aws_db_instance.main.address
}
```

---

## Sensitive Outputs

If an output contains a secret (password, key, token), mark it as sensitive. Terraform will hide it from normal output:

```hcl
output "db_password" {
  value     = random_password.db.result
  sensitive = true
}

output "api_key" {
  value     = aws_api_gateway_api_key.app.value
  sensitive = true
}
```

Behavior:
```bash
terraform apply
# Outputs:
# db_password = (sensitive value)
# api_key     = (sensitive value)

# To see the value explicitly:
terraform output -raw db_password
# → SuperSecretGeneratedPassword123
```

Sensitive values ARE stored in state. Protect your state file.

---

## The terraform output Command

```bash
# Show all outputs after apply
terraform output

# Show a specific output
terraform output web_server_ip

# Show value without quotes (useful for scripting)
terraform output -raw web_server_ip

# Show output in JSON format
terraform output -json

# Show all outputs as JSON (great for parsing in scripts)
terraform output -json | jq '.web_server_ip.value'
```

Example scripting pattern:
```bash
#!/bin/bash
# Deploy and then configure the server with Ansible

terraform apply -auto-approve

# Get the IP address from Terraform output
SERVER_IP=$(terraform output -raw web_server_ip)

# Pass it to Ansible
ansible-playbook -i "${SERVER_IP}," site.yml
```

---

## Outputs in Modules

Outputs are the ONLY way for a child module to expose values to its parent. Without outputs, the parent cannot see anything the module created.

Child module (`modules/networking/outputs.tf`):
```hcl
output "vpc_id" {
  description = "The ID of the created VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs of all public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of all private subnets"
  value       = aws_subnet.private[*].id
}
```

Parent/root module (`main.tf`):
```hcl
module "networking" {
  source = "./modules/networking"
  # ...
}

# Access the child module's outputs with module.<name>.<output_name>
resource "aws_instance" "app" {
  ami       = data.aws_ami.al2023.id
  subnet_id = module.networking.public_subnet_ids[0]  # from child module
}

# Pass module outputs to root outputs
output "vpc_id" {
  value = module.networking.vpc_id
}
```

---

## Outputs as Remote State Data Source

If you have two separate Terraform configurations (e.g., networking managed separately from compute), one can read the other's outputs via `terraform_remote_state`:

Infrastructure team's config outputs:
```hcl
# networking/outputs.tf
output "vpc_id"            { value = aws_vpc.main.id }
output "private_subnet_ids" { value = aws_subnet.private[*].id }
```

Application team's config reading it:
```hcl
# app/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  ami       = data.aws_ami.al2023.id
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
}
```

---

## Complete Outputs File Example

```hcl
# outputs.tf — a real-world example

output "alb_dns_name" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.app.dns_name
}

output "alb_zone_id" {
  description = "Zone ID for Route53 alias records"
  value       = aws_lb.app.zone_id
}

output "asg_name" {
  description = "Name of the Auto Scaling Group"
  value       = aws_autoscaling_group.app.name
}

output "rds_endpoint" {
  description = "RDS database endpoint"
  value       = aws_db_instance.app.endpoint
}

output "s3_bucket_name" {
  description = "Name of the S3 bucket for application assets"
  value       = aws_s3_bucket.assets.id
}

output "iam_role_arn" {
  description = "ARN of the EC2 instance IAM role"
  value       = aws_iam_role.ec2.arn
}

# Connection string for developers (computed from multiple resources)
output "app_url" {
  description = "HTTPS URL for the application"
  value       = "https://${aws_lb.app.dns_name}"
}
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  output "name" { value = ..., description = "..." }             │
│  terraform output                  → show all outputs           │
│  terraform output -raw name        → value only (for scripts)   │
│  terraform output -json            → JSON format                │
│                                                                 │
│  sensitive = true hides value in terminal (still in state!)     │
│                                                                 │
│  In modules: outputs = the ONLY way to pass values out          │
│  Reference module output: module.module_name.output_name        │
│                                                                 │
│  Remote state: data "terraform_remote_state" reads another      │
│  config's outputs (great for splitting infra into layers)       │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Variables](./variables.md) &nbsp;|&nbsp; **Next:** [Locals →](./locals.md)

**Related Topics:** [Variables](./variables.md) · [Locals](./locals.md) · [Modules](../06_modules/creating_modules.md) · [Remote State](../05_state_management/remote_state.md)
