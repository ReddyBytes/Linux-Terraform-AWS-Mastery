# CI/CD Integration — Automating Terraform in Pipelines

## The Assembly Line Analogy

In a car factory, every car goes through the same assembly line. Each station does a specific job: weld the frame, install the engine, paint the body, test drive. No car skips steps. No car gets installed in a different order. The assembly line guarantees consistency.

Integrating Terraform into a CI/CD pipeline creates an assembly line for your infrastructure changes. Every change — from a developer's PR to production deployment — goes through the same automated steps: validate, plan, security scan, review, apply. No more "I ran it manually from my laptop" inconsistencies.

---

## The CI/CD Terraform Workflow

```
┌──────────────────────────────────────────────────────────────────┐
│              TERRAFORM CI/CD PIPELINE                            │
│                                                                  │
│  Developer opens PR                                              │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  On Pull Request                                        │     │
│  │  1. terraform fmt -check     (formatting)               │     │
│  │  2. terraform validate       (syntax)                   │     │
│  │  3. checkov -d .             (security scan)            │     │
│  │  4. terraform init           (download providers)       │     │
│  │  5. terraform plan           (show what changes)        │     │
│  │     → Post plan output as PR comment                    │     │
│  └────────────────────────────────┬────────────────────────┘     │
│                                   │ Team reviews plan             │
│                                   ▼                              │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  On Merge to main branch                                 │    │
│  │  1. terraform init                                       │    │
│  │  2. terraform apply -auto-approve                        │    │
│  │     (or require manual approval step in pipeline)        │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

---

## GitHub Actions: Complete Terraform Pipeline

```yaml
# .github/workflows/terraform.yml

name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write    # for OIDC auth to AWS
  contents: read
  pull-requests: write  # to post plan as PR comment

env:
  TF_VERSION: "1.7.0"
  AWS_REGION: "us-east-1"

jobs:
  terraform:
    name: Terraform Plan and Apply
    runs-on: ubuntu-latest

    steps:
      # ─── Checkout code ───────────────────────────────────────
      - name: Checkout
        uses: actions/checkout@v4

      # ─── Configure AWS credentials via OIDC (no static keys!) ─
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-terraform-role
          aws-region: ${{ env.AWS_REGION }}

      # ─── Install Terraform ────────────────────────────────────
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      # ─── Check formatting ─────────────────────────────────────
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        working-directory: environments/prod

      # ─── Initialize ───────────────────────────────────────────
      - name: Terraform Init
        run: terraform init
        working-directory: environments/prod

      # ─── Validate syntax ──────────────────────────────────────
      - name: Terraform Validate
        run: terraform validate
        working-directory: environments/prod

      # ─── Security scan ────────────────────────────────────────
      - name: Security Scan with Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: environments/prod
          soft_fail: true   # don't block on findings (or set to false in strict mode)

      # ─── Plan ─────────────────────────────────────────────────
      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        working-directory: environments/prod
        env:
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}

      # ─── Post plan as PR comment ──────────────────────────────
      - name: Comment Plan on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📋
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Pushed by @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      # ─── Apply (only on merge to main) ───────────────────────
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply tfplan
        working-directory: environments/prod
        env:
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
```

---

## GitHub Actions: OIDC Role Setup

To avoid storing AWS access keys as GitHub secrets, use OIDC:

```hcl
# One-time setup: create the OIDC provider and role in AWS

resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

data "aws_iam_policy_document" "github_actions_assume" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:myorg/myrepo:*"]   # restrict to your repo
    }
  }
}

resource "aws_iam_role" "github_actions_terraform" {
  name               = "github-actions-terraform-role"
  assume_role_policy = data.aws_iam_policy_document.github_actions_assume.json
}

# Attach the permissions this role needs to run terraform
resource "aws_iam_role_policy_attachment" "terraform_permissions" {
  role       = aws_iam_role.github_actions_terraform.name
  policy_arn = aws_iam_policy.terraform_deploy.arn
}
```

---

## Atlantis — GitOps for Terraform

[Atlantis](https://www.runatlantis.io/) is a dedicated tool for Terraform automation in pull requests. It runs as a server in your infrastructure:

```yaml
# atlantis.yaml — configuration file in your repo root

version: 3
projects:
  - name: myapp-prod
    dir: environments/prod
    workspace: default
    autoplan:
      when_modified: ["*.tf", "../modules/**/*.tf"]
      enabled: true

  - name: myapp-dev
    dir: environments/dev
    workspace: default
    autoplan:
      when_modified: ["*.tf"]
      enabled: true
```

When a PR is opened:
```
# Atlantis comments on PR automatically:
atlantis plan   ← shows the plan
atlantis apply  ← applies after approval

# Or type in PR comments to trigger:
atlantis plan -p myapp-prod
atlantis apply -p myapp-prod
```

**Atlantis vs GitHub Actions:**
- GitHub Actions: more flexible, integrated with GitHub ecosystem
- Atlantis: purpose-built for Terraform, simpler setup for Terraform-only teams

---

## Essential Pipeline Variables and Secrets

```
┌──────────────────────────────────────────────────────────────────┐
│              PIPELINE SECRETS TO CONFIGURE                       │
│                                                                  │
│  GitHub Actions secrets (or equivalent in your CI platform):    │
│                                                                  │
│  AWS_ACCOUNT_ID         → for OIDC role ARN construction        │
│  DB_PASSWORD            → database password (TF_VAR_db_password) │
│  API_KEY                → any sensitive API keys                 │
│                                                                  │
│  If not using OIDC (less preferred):                             │
│  AWS_ACCESS_KEY_ID                                               │
│  AWS_SECRET_ACCESS_KEY                                           │
│                                                                  │
│  Pipeline environment variables:                                 │
│  TF_VERSION = "1.7.0"                                            │
│  AWS_REGION = "us-east-1"                                        │
│  TF_IN_AUTOMATION = "1"  ← disables interactive prompts         │
└──────────────────────────────────────────────────────────────────┘
```

---

## Pipeline Checklist

```
┌──────────────────────────────────────────────────────────────────┐
│              CI/CD TERRAFORM PIPELINE CHECKLIST                  │
├──────────────────────────────────────────────────────────────────┤
│  [ ] terraform fmt -check  on every PR                           │
│  [ ] terraform validate    on every PR                           │
│  [ ] security scan (checkov/tfsec) on every PR                  │
│  [ ] terraform plan output posted to PR as comment               │
│  [ ] Plan saved as artifact (terraform plan -out=tfplan)         │
│  [ ] Apply uses the saved plan (not a new plan)                  │
│  [ ] Apply only on merge to main (not on PR)                     │
│  [ ] Remote state configured (S3 + DynamoDB locking)            │
│  [ ] No credentials in pipeline code (use OIDC or secrets)      │
│  [ ] TF_IN_AUTOMATION=1 set in pipeline environment             │
│  [ ] Notifications on apply failure                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                 │
├─────────────────────────────────────────────────────────────────┤
│  PR pipeline: fmt check → validate → security scan → plan       │
│  Merge pipeline: init → apply (using saved plan file)           │
│                                                                 │
│  GitHub Actions: hashicorp/setup-terraform + OIDC auth          │
│  OIDC: no static AWS keys needed — use role assumption          │
│  Atlantis: purpose-built GitOps tool for Terraform              │
│  Checkov: security scan embedded in pipeline                    │
│                                                                 │
│  Key settings:                                                  │
│    TF_IN_AUTOMATION=1   → disables interactive prompts          │
│    -out=tfplan          → save plan, apply the exact same plan  │
│    sensitive secrets    → pipeline secrets, not in code         │
└─────────────────────────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Security](./security.md) &nbsp;|&nbsp; **Next:** [Terraform Interview Questions →](../99_interview_master/terraform_questions.md)

**Related Topics:** [Security](./security.md) · [Remote State](../05_state_management/remote_state.md) · [Code Organization](./code_organization.md)
