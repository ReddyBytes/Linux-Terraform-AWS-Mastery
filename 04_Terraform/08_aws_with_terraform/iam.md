# IAM with Terraform — Roles, Policies, and Least Privilege

## The Employee Badge Analogy

In a large office building, not everyone has access to every room. A junior developer cannot walk into the server room. A finance employee cannot enter the engineering lab. Each person gets a badge that grants specific access — exactly what they need, nothing more.

AWS IAM (Identity and Access Management) is that badge system for your cloud resources. EC2 instances, Lambda functions, and CI/CD pipelines all need "badges" (IAM roles) to interact with AWS services. Terraform lets you define exactly what each badge allows — no more, no less.

---

## IAM Concepts

```
┌──────────────────────────────────────────────────────────────────┐
│                     IAM COMPONENTS                               │
│                                                                  │
│  IAM Role       → A badge. An identity that can be assumed.      │
│                   Not tied to a person — tied to a service.      │
│                                                                  │
│  IAM Policy     → A set of permissions: what actions are         │
│                   Allowed or Denied on which resources.          │
│                                                                  │
│  Role Attachment → Attaching a policy to a role.                 │
│                   "This badge grants these permissions."         │
│                                                                  │
│  Instance Profile → Wrapper that lets an EC2 instance assume     │
│                     an IAM role.                                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## IAM Role for EC2

The most common pattern: an EC2 instance needs to call AWS APIs (read from S3, write to CloudWatch, get secrets from SSM).

```hcl
# 1. Define who can ASSUME this role (trust policy)
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    sid     = "EC2AssumeRole"
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

# 2. Create the role
resource "aws_iam_role" "app_server" {
  name               = "${var.project}-${var.environment}-app-server-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
  description        = "Role for app server EC2 instances"

  tags = {
    Project     = var.project
    Environment = var.environment
  }
}

# 3. Create an instance profile (required for EC2 to use the role)
resource "aws_iam_instance_profile" "app_server" {
  name = "${var.project}-${var.environment}-app-server-profile"
  role = aws_iam_role.app_server.name
}

# 4. Attach the role to your EC2 instance
resource "aws_instance" "app" {
  ami                  = data.aws_ami.al2023.id
  instance_type        = "t3.micro"
  iam_instance_profile = aws_iam_instance_profile.app_server.name
  # ...
}
```

---

## IAM Policy — Custom Least-Privilege

Never attach `AdministratorAccess` to a role. Write specific policies:

```hcl
# Custom policy: allow EC2 to read from a specific S3 bucket only
data "aws_iam_policy_document" "app_s3_access" {
  statement {
    sid    = "AllowS3BucketList"
    effect = "Allow"
    actions = [
      "s3:ListBucket",
      "s3:GetBucketLocation"
    ]
    resources = [
      aws_s3_bucket.app_data.arn
    ]
  }

  statement {
    sid    = "AllowS3ObjectAccess"
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject"
    ]
    resources = [
      "${aws_s3_bucket.app_data.arn}/*"
    ]
  }
}

resource "aws_iam_policy" "app_s3_access" {
  name        = "${var.project}-${var.environment}-app-s3-policy"
  description = "Allow app servers to read/write the app data bucket"
  policy      = data.aws_iam_policy_document.app_s3_access.json
}

# Attach the policy to the role
resource "aws_iam_role_policy_attachment" "app_s3" {
  role       = aws_iam_role.app_server.name
  policy_arn = aws_iam_policy.app_s3_access.arn
}
```

---

## Attaching AWS Managed Policies

For common use cases, AWS provides pre-built managed policies:

```hcl
# CloudWatch agent — write logs and metrics
resource "aws_iam_role_policy_attachment" "cloudwatch_agent" {
  role       = aws_iam_role.app_server.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# SSM — allow AWS Systems Manager access (no SSH needed)
resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.app_server.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Read-only access to ECR (for pulling Docker images)
resource "aws_iam_role_policy_attachment" "ecr_read" {
  role       = aws_iam_role.app_server.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}
```

---

## IAM Role for Lambda

```hcl
data "aws_iam_policy_document" "lambda_assume_role" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda_exec" {
  name               = "${var.project}-${var.environment}-lambda-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume_role.json
}

# Basic Lambda execution: write to CloudWatch Logs
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

---

## IAM Role for CI/CD (GitHub Actions / Jenkins)

```hcl
# Allow a CI/CD system to deploy to AWS using OIDC (no static keys!)
data "aws_iam_policy_document" "github_actions_assume" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:myorg/myrepo:*"]
    }
  }
}

resource "aws_iam_role" "github_actions" {
  name               = "github-actions-deploy-role"
  assume_role_policy = data.aws_iam_policy_document.github_actions_assume.json
}

# Policy for what GitHub Actions can do (e.g., deploy to ECS)
data "aws_iam_policy_document" "deploy_permissions" {
  statement {
    effect = "Allow"
    actions = [
      "ecs:UpdateService",
      "ecs:DescribeServices",
      "ecr:GetAuthorizationToken",
      "ecr:BatchCheckLayerAvailability",
      "ecr:PutImage"
    ]
    resources = ["*"]   # In real life, scope this to specific resources
  }
}

resource "aws_iam_role_policy" "deploy" {
  name   = "deploy-permissions"
  role   = aws_iam_role.github_actions.id
  policy = data.aws_iam_policy_document.deploy_permissions.json
}
```

---

## Conditions in IAM Policies

Conditions make policies smarter:

```hcl
data "aws_iam_policy_document" "conditional_access" {
  statement {
    effect  = "Allow"
    actions = ["s3:*"]
    resources = ["*"]

    # Only allow access from within the VPC
    condition {
      test     = "StringEquals"
      variable = "aws:SourceVpc"
      values   = [aws_vpc.main.id]
    }
  }

  statement {
    effect  = "Deny"
    actions = ["*"]
    resources = ["*"]

    # Deny if not using MFA (for human users)
    condition {
      test     = "BoolIfExists"
      variable = "aws:MultiFactorAuthPresent"
      values   = ["false"]
    }
  }
}
```

---

## Complete IAM Setup Summary

```hcl
# Complete working IAM for an app server

data "aws_iam_policy_document" "ec2_trust" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "app" {
  name               = "${var.project}-app-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_trust.json
}

resource "aws_iam_role_policy_attachment" "ssm"        { role = aws_iam_role.app.name; policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore" }
resource "aws_iam_role_policy_attachment" "cloudwatch" { role = aws_iam_role.app.name; policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy" }

resource "aws_iam_instance_profile" "app" {
  name = "${var.project}-app-profile"
  role = aws_iam_role.app.name
}
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  IAM Role = identity a service assumes (not a human)            │
│  Trust policy = who can assume the role (ec2/lambda/etc.)       │
│  IAM Policy = list of allowed/denied actions on resources       │
│  Instance profile = container that lets EC2 use a role          │
│                                                                 │
│  Workflow:                                                      │
│  1. Create role with trust policy (who can assume it)           │
│  2. Create policy with permissions (what it can do)             │
│  3. Attach policy to role (aws_iam_role_policy_attachment)      │
│  4. Create instance profile (for EC2 use)                       │
│  5. Reference instance profile in aws_instance                  │
│                                                                 │
│  Least privilege: grant only what is needed, nothing more       │
│  Use OIDC for CI/CD — avoid static long-lived credentials       │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← S3](./s3.md) &nbsp;|&nbsp; **Next:** [RDS →](./rds.md)

**Related Topics:** [EC2](./ec2.md) · [S3](./s3.md) · [Security Best Practices](../09_best_practices/security.md)
