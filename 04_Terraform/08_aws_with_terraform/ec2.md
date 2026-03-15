# EC2 with Terraform — Launching Virtual Machines

## The Hotel Room Analogy

An EC2 instance is like a hotel room. You pick the room size (instance type), the floor and building (availability zone and VPC), the key card policy (security group), and you might leave a wake-up request (user_data). When you check out, the room goes back to the pool. With Terraform, you automate the entire hotel booking process.

---

## EC2 Architecture for This Example

```
┌──────────────────────────────────────────────────────────────────┐
│                     VPC (10.0.0.0/16)                            │
│                                                                  │
│  ┌─────────────────────────────────────────────┐                 │
│  │           Public Subnet (10.0.1.0/24)        │                │
│  │                                             │                 │
│  │   ┌─────────────────────────────────────┐   │                 │
│  │   │         aws_instance.web            │   │                 │
│  │   │  ami: latest Amazon Linux 2023      │   │                 │
│  │   │  type: t3.micro                     │   │                 │
│  │   │  key: aws_key_pair.deployer         │   │                 │
│  │   │  sg:  aws_security_group.web        │   │                 │
│  │   └─────────────────────────────────────┘   │                 │
│  │             │                               │                 │
│  │    aws_eip.web ──► public IP                │                 │
│  └─────────────────────────────────────────────┘                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## Complete EC2 Configuration

### Data Source: Find the Latest AMI

```hcl
# Always get the latest Amazon Linux 2023 AMI — never hardcode AMI IDs
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

### Key Pair

```hcl
# Method 1: Reference an existing key pair in AWS
resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("~/.ssh/id_rsa.pub")   # reads your local public key
}

# Method 2: Generate a key pair with Terraform
resource "tls_private_key" "web" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "web" {
  key_name   = "${var.project}-${var.environment}-key"
  public_key = tls_private_key.web.public_key_openssh
}

# Save the private key locally (sensitive!)
resource "local_sensitive_file" "private_key" {
  content         = tls_private_key.web.private_key_pem
  filename        = "${path.module}/keys/${var.project}-${var.environment}.pem"
  file_permission = "0600"
}
```

### Security Group

```hcl
resource "aws_security_group" "web" {
  name        = "${var.project}-${var.environment}-web-sg"
  description = "Web server security group"
  vpc_id      = var.vpc_id

  # SSH access (restrict to your IP in production!)
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.ssh_allowed_cidr]   # e.g., "203.0.113.0/32" (your IP)
  }

  # HTTP
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # All outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project}-${var.environment}-web-sg" }
}
```

### EC2 Instance

```hcl
resource "aws_instance" "web" {
  # What OS image to use
  ami = data.aws_ami.al2023.id

  # How powerful the machine should be
  instance_type = var.instance_type   # "t3.micro"

  # Network placement
  subnet_id                   = var.public_subnet_id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true

  # SSH key
  key_name = aws_key_pair.deployer.key_name

  # Startup script — runs once when instance first launches
  user_data = <<-EOF
    #!/bin/bash
    set -e

    # Update system packages
    dnf update -y

    # Install a web server
    dnf install -y httpd

    # Start and enable httpd
    systemctl start httpd
    systemctl enable httpd

    # Create a simple test page
    cat > /var/www/html/index.html <<HTML
    <!DOCTYPE html>
    <html>
      <body>
        <h1>Hello from ${var.project} on $(hostname)</h1>
        <p>Environment: ${var.environment}</p>
      </body>
    </html>
    HTML
  EOF

  # Replace instance when user_data changes (instead of just ignoring)
  # Comment this out if you don't want replacement on script changes
  user_data_replace_on_change = true

  # Root volume
  root_block_device {
    volume_size           = 20     # GB
    volume_type           = "gp3"
    encrypted             = true
    delete_on_termination = true
  }

  # IAM instance profile (optional — for EC2 to call AWS APIs)
  iam_instance_profile = aws_iam_instance_profile.web.name

  tags = {
    Name        = "${var.project}-${var.environment}-web"
    Environment = var.environment
    Project     = var.project
  }
}
```

### Elastic IP (Static Public IP)

```hcl
# An Elastic IP persists even if the instance is stopped/started
resource "aws_eip" "web" {
  domain   = "vpc"
  instance = aws_instance.web.id

  # EIP requires IGW to exist first
  depends_on = [var.internet_gateway_id]

  tags = { Name = "${var.project}-${var.environment}-web-eip" }
}
```

### IAM Instance Profile (for AWS API Access)

```hcl
# Allow EC2 to call AWS APIs (e.g., read from S3, write to CloudWatch)
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "web" {
  name               = "${var.project}-${var.environment}-web-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
}

# Attach CloudWatch agent policy for monitoring
resource "aws_iam_role_policy_attachment" "cloudwatch" {
  role       = aws_iam_role.web.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# Attach SSM policy for AWS Systems Manager (no SSH required)
resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.web.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "web" {
  name = "${var.project}-${var.environment}-web-profile"
  role = aws_iam_role.web.name
}
```

### Outputs

```hcl
output "instance_id"         { value = aws_instance.web.id }
output "instance_public_ip"  { value = aws_eip.web.public_ip }
output "instance_private_ip" { value = aws_instance.web.private_ip }
output "public_dns"          { value = aws_instance.web.public_dns }

output "ssh_command" {
  value = "ssh -i ~/.ssh/id_rsa ec2-user@${aws_eip.web.public_ip}"
}

output "web_url" {
  value = "http://${aws_eip.web.public_ip}"
}
```

---

## Connecting to Your Instance

After `terraform apply`:

```bash
# Get the public IP
terraform output instance_public_ip

# SSH to the instance
ssh -i ~/.ssh/id_rsa ec2-user@$(terraform output -raw instance_public_ip)

# Or use the SSH command output directly
$(terraform output -raw ssh_command)

# Or connect via AWS Systems Manager (no SSH key needed if SSM is attached)
aws ssm start-session --target $(terraform output -raw instance_id)
```

---

## Common instance_type Options

```
┌────────────────┬────────────────────────────────────┬─────────────┐
│  Instance Type │  Use Case                          │  Free Tier? │
├────────────────┼────────────────────────────────────┼─────────────┤
│  t3.micro      │  Learning, dev, low traffic apps   │  Yes        │
│  t3.small      │  Small production workloads        │  No         │
│  t3.medium     │  Medium web servers                │  No         │
│  t3.large      │  Production web/app servers        │  No         │
│  m5.large      │  General purpose, balanced         │  No         │
│  c5.large      │  Compute-intensive (ML, video)     │  No         │
│  r5.large      │  Memory-intensive (databases)      │  No         │
└────────────────┴────────────────────────────────────┴─────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  aws_instance: ami, instance_type, subnet_id, key_name          │
│  data.aws_ami: always find latest AMI dynamically               │
│  aws_key_pair: reference your local public key                  │
│  aws_security_group: firewall — restrict SSH to your IP!        │
│  user_data: startup script (runs once at first boot)            │
│  aws_eip: static public IP that survives stop/start             │
│  aws_iam_instance_profile: lets EC2 call AWS APIs               │
│                                                                 │
│  Connect: ssh -i key ec2-user@<public_ip>                       │
│  Or: aws ssm start-session --target <instance-id>               │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← VPC](./vpc.md) &nbsp;|&nbsp; **Next:** [S3 →](./s3.md)

**Related Topics:** [VPC](./vpc.md) · [IAM](./iam.md) · [Data Sources](../03_providers_resources/data_sources.md)
