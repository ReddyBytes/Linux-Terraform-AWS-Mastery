<div align="center">

<img src="./docs/assets/banner.svg" alt="Linux Bash AWS Terraform Mastery" width="100%"/>

[![Linux](https://img.shields.io/badge/Linux-22C55E?style=for-the-badge&logo=linux&logoColor=white)](./01_Linux/README.md)
[![Bash](https://img.shields.io/badge/Bash_Scripting-F59E0B?style=for-the-badge&logo=gnubash&logoColor=white)](./02_Bash-Scripting/README.md)
[![AWS](https://img.shields.io/badge/AWS-F97316?style=for-the-badge&logo=amazonaws&logoColor=white)](./03_AWS/README.md)
[![Terraform](https://img.shields.io/badge/Terraform-7C3AED?style=for-the-badge&logo=terraform&logoColor=white)](./04_Terraform/README.md)

[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-14B8A6?style=flat-square)](https://github.com/)
[![Learning Path](https://img.shields.io/badge/Learning_Path-Structured-0EA5E9?style=flat-square)](./01_Linux/README.md)
[![Files](https://img.shields.io/badge/Files-100+-14B8A6?style=flat-square)](#curriculum)

**From zero to production-ready DevOps/Cloud engineer — in the right order, with real-world examples throughout.**

</div>

---

## 🗺️ Learning Path

```
01 Linux  ──►  02 Bash Scripting  ──►  03 AWS Cloud  ──►  04 Terraform
    │                  │                     │                   │
How servers work   Automate tasks       Deploy on cloud    Infrastructure as Code
Shell & files      Write scripts        EC2/S3/RDS/VPC     Plan, apply, destroy
Processes & SSH    Cron & debugging     IAM & security     Modules & remote state
```

**Start here if new:** `01_Linux → 02_Bash-Scripting → 03_AWS → 04_Terraform`

---

## 📚 Curriculum

### 🐧 [01 — Linux](./01_Linux/README.md)

The foundation of everything. Every server, container, and cloud instance runs Linux.

| Folder | Topics |
|--------|--------|
| [01 Fundamentals](./01_Linux/01_fundamentals/) | What is Linux, architecture, distributions |
| [02 Filesystem](./01_Linux/02_filesystem/) | Directory structure, file operations, links & inodes |
| [03 Shell Basics](./01_Linux/03_shell_basics/) | Commands, pipes & redirection, text processing |
| [04 Users & Permissions](./01_Linux/04_users_permissions/) | Users/groups, file permissions, sudo & root |
| [05 Processes](./01_Linux/05_processes/) | Process management, signals, jobs & daemons |
| [06 Networking](./01_Linux/06_networking/) | Network commands, SSH, firewall |
| [07 Package Management](./01_Linux/07_package_management/) | apt & yum, building from source |
| [08 System Administration](./01_Linux/08_system_administration/) | systemd, logs & journalctl, disk management |
| [99 Interview Master](./01_Linux/99_interview_master/) | Beginner → Advanced Q&A |

---

### 🔧 [02 — Bash Scripting](./02_Bash-Scripting/README.md)

Automate repetitive tasks, build deployment pipelines, and manage systems at scale.

| Folder | Topics |
|--------|--------|
| [01 Shell Basics](./02_Bash-Scripting/01_shell_basics/) | First script, shebang & execution |
| [02 Variables & Data](./02_Bash-Scripting/02_variables_and_data/) | Variables, strings, arrays |
| [03 Control Flow](./02_Bash-Scripting/03_control_flow/) | Conditionals, loops, case statements |
| [04 Functions](./02_Bash-Scripting/04_functions/) | Functions, scope & return values |
| [05 Input & Output](./02_Bash-Scripting/05_input_output/) | User input, file operations, pipes |
| [06 Error Handling](./02_Bash-Scripting/06_error_handling/) | Exit codes, traps, debugging |
| [07 Automation](./02_Bash-Scripting/07_automation/) | Cron jobs, scheduling |
| [08 Real World Scripts](./02_Bash-Scripting/08_real_world_scripts/) | Backup, deployment, monitoring scripts |
| [99 Interview Master](./02_Bash-Scripting/99_interview_master/) | Bash scripting Q&A |

---

### ☁️ [03 — AWS](./03_AWS/README.md)

The world's most used cloud platform. Deploy, scale, and manage cloud infrastructure.

| Folder | Topics |
|--------|--------|
| [01 Cloud Foundations](./03_AWS/01_cloud_foundations/) | Cloud concepts, IaaS/PaaS/SaaS |
| [02 Global Infrastructure](./03_AWS/02_global_infrastructure/) | Regions, AZs, edge locations |
| [03 Compute](./03_AWS/03_compute/) | EC2, Auto Scaling, Load Balancers |
| [04 Storage](./03_AWS/04_storage/) | S3, EBS, EFS, Glacier |
| [05 Networking](./03_AWS/05_networking/) | VPC, subnets, Route 53, CloudFront |
| [06 Security](./03_AWS/06_security/) | IAM, KMS, Cognito, GuardDuty |
| [07 Databases](./03_AWS/07_databases/) | RDS, Aurora, DynamoDB, ElastiCache |
| [08 Monitoring](./03_AWS/08_monitoring/) | CloudWatch, CloudTrail, X-Ray |
| [09 IaC](./03_AWS/09_iac/) | CloudFormation, CDK |
| [10 Containers](./03_AWS/10_containers/) | ECS, EKS, Fargate, ECR |
| [11 Serverless](./03_AWS/11_serverless/) | Lambda, API Gateway, SQS, SNS |
| [12 Data Analytics](./03_AWS/12_data_analytics/) | Athena, Glue, Kinesis, Redshift |
| [13 DevOps/CICD](./03_AWS/13_devops_cicd/) | CodePipeline, CodeBuild, CodeDeploy |
| [14 Architecture](./03_AWS/14_architecture/) | Well-Architected, HA, DR |
| [15 Cost Optimization](./03_AWS/15_cost_optimization/) | Reserved, Spot, Savings Plans |
| [16 AI/ML](./03_AWS/16_ai_ml/) | Bedrock, SageMaker, Rekognition |
| [99 Interview Master](./03_AWS/99_interview_master/) | AWS Q&A across all domains |

---

### 🏗️ [04 — Terraform](./04_Terraform/README.md)

Write infrastructure as code. Define, preview, and deploy cloud resources reproducibly.

| Folder | Topics |
|--------|--------|
| [01 Introduction](./04_Terraform/01_introduction/) | What is Terraform, installation, vs others |
| [02 HCL Basics](./04_Terraform/02_hcl_basics/) | Syntax, data types, expressions |
| [03 Providers & Resources](./04_Terraform/03_providers_resources/) | Providers, resources, data sources |
| [04 Variables & Outputs](./04_Terraform/04_variables_outputs/) | Variables, outputs, locals |
| [05 State Management](./04_Terraform/05_state_management/) | State file, remote state, state commands |
| [06 Modules](./04_Terraform/06_modules/) | Creating modules, composition, registry |
| [07 Workspaces](./04_Terraform/07_workspaces/) | Workspaces, environments |
| [08 AWS with Terraform](./04_Terraform/08_aws_with_terraform/) | EC2, VPC, S3, RDS, IAM with Terraform |
| [09 Best Practices](./04_Terraform/09_best_practices/) | Code organization, security, CI/CD |
| [99 Interview Master](./04_Terraform/99_interview_master/) | Terraform Q&A |

---

## 🎯 Who Is This For?

| Role | Recommended Path |
|------|-----------------|
| **Complete beginners** | Linux → Bash → AWS basics |
| **Developers moving to cloud** | Linux + AWS (skip Bash if comfortable with scripting) |
| **DevOps / Platform engineers** | All 4 sections in order |
| **SRE candidates** | Linux deep dive + Bash + AWS monitoring |
| **Cloud architects** | AWS architecture + Terraform |
| **Interview prep** | All `99_interview_master/` folders |

---

## 💡 How to Use This Repo

1. **Follow the order** — Linux first, Bash second, AWS third, Terraform last
2. **Read the analogy** — every topic starts with a real-world analogy to make it stick
3. **Try the commands** — spin up a Linux VM (Ubuntu on VirtualBox or EC2 free tier) and run everything
4. **Build projects** — after AWS + Terraform, provision a real VPC + EC2 + RDS entirely with Terraform
5. **Interview sections** — before any interview, go through the `99_interview_master/` folder for that topic

---

<div align="center">

[![Start: Linux](https://img.shields.io/badge/Start_Here-01_Linux-22C55E?style=for-the-badge&logo=linux&logoColor=white)](./01_Linux/README.md)

*Linux · Bash · AWS · Terraform · Story-Based · Real-World Examples · Beginner to Expert*

</div>
