# Stage 09a — CloudFormation: Infrastructure as Code

> Define your entire AWS infrastructure in code. Version it, review it, replicate it, destroy it — all from a template.

## 1. Core Intuition

You've built a production environment: VPC, subnets, EC2 auto-scaling group, RDS database, ALB, security groups, IAM roles. It took 3 days to set up.

Now you need a **staging environment** that's identical. Do you click through the console again for 3 days?

**CloudFormation** = Write your infrastructure as code (YAML/JSON). Deploy it in one command. Create 10 identical environments in minutes. Delete it all when done.

Infrastructure as Code (IaC) is the practice of managing infrastructure through version-controlled code rather than manual clicks.

## 2. Story-Based Analogy — The Architect's Blueprint

```
Traditional approach = Building a house by memory
  "I think the kitchen goes here... or was it there?"
  Every house you build is slightly different.
  If the house burns down, rebuilding from memory is error-prone.

CloudFormation = The Architect's blueprint
  Blueprint defines EVERYTHING:
    • 3 bedrooms, 2 bathrooms
    • Kitchen on the east side
    • Foundation: concrete, 12 inches thick
    • Wiring: follows code 14.2.1

  From one blueprint → build identical houses anywhere.
  Blueprint is version-controlled → see every change ever made.
  "Before this change, the kitchen was 10x10. Now 12x12."
  Rebuild from blueprint after fire → identical house, every time.
```

## 3. CloudFormation Concepts

```
Template   = YAML or JSON file defining resources
Stack      = A deployed instance of a template
             (one stack = one set of resources created together)
StackSet   = Deploy same stack to multiple accounts/regions
Change Set = Preview what will change before applying
Drift      = Resources manually changed outside CloudFormation

Template Structure:
  AWSTemplateFormatVersion: '2010-09-09'
  Description: My application infrastructure

  Parameters:         ← Input values (environment name, instance type)
  Mappings:          ← Key-value lookup tables
  Conditions:        ← Logic (if production → use larger instance)
  Resources:         ← REQUIRED: actual AWS resources to create
  Outputs:           ← Values to export (ALB URL, DB endpoint)
```

## 4. Template Examples

### Simple EC2 Instance

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple EC2 with Security Group

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
    Description: EC2 instance type

  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Name of an existing EC2 key pair

Resources:
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP and SSH
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 10.0.0.0/8  # Internal only

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Sub '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64}}'
      InstanceType: !Ref InstanceType  # Reference parameter
      KeyName: !Ref KeyName
      SecurityGroups:
        - !Ref WebServerSecurityGroup  # Reference security group above
      UserData:
        Fn::Base64: |
          #!/bin/bash
          dnf install -y nginx
          systemctl start nginx

Outputs:
  InstancePublicIP:
    Description: Public IP of web server
    Value: !GetAtt WebServer.PublicIp

  InstanceId:
    Description: Instance ID
    Value: !Ref WebServer
```

### Full VPC Stack

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Production VPC with Public and Private Subnets

Parameters:
  Environment:
    Type: String
    AllowedValues: [production, staging, development]
    Default: development

  VpcCidr:
    Type: String
    Default: 10.0.0.0/16

Conditions:
  IsProduction: !Equals [!Ref Environment, production]

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-vpc"
        - Key: Environment
          Value: !Ref Environment

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-igw"

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [0, !Cidr [!Ref VpcCidr, 6, 8]]
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-public-1a"

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [2, !Cidr [!Ref VpcCidr, 6, 8]]
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-private-1a"

  NatGateway:
    Type: AWS::EC2::NatGateway
    Condition: IsProduction  # Only create NAT GW for production!
    Properties:
      AllocationId: !GetAtt NatEIP.AllocationId
      SubnetId: !Ref PublicSubnet1

  NatEIP:
    Type: AWS::EC2::EIP
    Condition: IsProduction
    Properties:
      Domain: vpc

Outputs:
  VpcId:
    Value: !Ref VPC
    Export:
      Name: !Sub "${Environment}-VpcId"  # Export for other stacks to import

  PublicSubnet1Id:
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub "${Environment}-PublicSubnet1Id"
```

## 5. CloudFormation Intrinsic Functions

```yaml
!Ref           → Reference a resource or parameter
!GetAtt        → Get an attribute of a resource
!Sub           → String substitution
!Join          → Join strings with delimiter
!Select        → Select from a list by index
!If            → Conditional value
!Equals        → Compare two values (use in Conditions)
!FindInMap     → Look up value in Mappings
!ImportValue   → Import an exported value from another stack
!Base64        → Base64 encode (for UserData)
!Cidr          → Generate CIDR blocks

Examples:
  VpcId: !Ref MyVPC              # Reference the VPC resource
  PublicIp: !GetAtt EC2.PublicIp # Get EC2 public IP attribute
  Name: !Sub "${Env}-web-server" # String interpolation
  DB: !ImportValue prod-DatabaseEndpoint  # Cross-stack reference
```

## 6. Stack Operations

```bash
# Create a new stack
aws cloudformation create-stack \
  --stack-name my-app-prod \
  --template-body file://vpc.yaml \
  --parameters \
    ParameterKey=Environment,ParameterValue=production \
    ParameterKey=VpcCidr,ParameterValue=10.0.0.0/16 \
  --capabilities CAPABILITY_IAM

# Watch creation progress
aws cloudformation describe-stack-events \
  --stack-name my-app-prod \
  --query 'StackEvents[*].[Timestamp,ResourceStatus,ResourceType,LogicalResourceId]' \
  --output table

# Update a stack (with change set)
aws cloudformation create-change-set \
  --stack-name my-app-prod \
  --change-set-name my-update \
  --template-body file://vpc-updated.yaml

# Review what will change
aws cloudformation describe-change-set \
  --stack-name my-app-prod \
  --change-set-name my-update

# Execute the change set
aws cloudformation execute-change-set \
  --stack-name my-app-prod \
  --change-set-name my-update

# Delete a stack (deletes ALL resources in it!)
aws cloudformation delete-stack --stack-name my-app-prod
```

## 7. Console Walkthrough

```
Step 1: Navigate
━━━━━━━━━━━━━━━━
Console: CloudFormation → Stacks → Create stack

Step 2: Template source
  Upload a template file → upload your YAML
  OR: Amazon S3 URL → paste S3 URL of template
  OR: Use a sample template (for learning)

Step 3: Stack details
  Stack name: my-app-staging
  Parameters: fill in prompted values (Environment, KeyName, etc.)

Step 4: Configure stack options
  Tags: Environment=staging, Team=platform
  Permissions: IAM role for CloudFormation to create resources
    (or use the current user's permissions)
  Stack failure options:
    "Roll back all stack resources" (default)
    → If anything fails, CloudFormation undoes everything

Step 5: Review and create
  Summary of all resources that will be created
  Click: Submit

Monitor in Events tab:
  Watch each resource being created in real time
  ✅ CREATE_COMPLETE for each resource
  ❌ CREATE_FAILED → error message shown, stack rolls back

Outputs tab:
  After successful creation, see all exported values
  (ALB URL, VPC ID, DB endpoint, etc.)
```

## 8. AWS CDK — CloudFormation with Real Code

```
CloudFormation = Write YAML/JSON (verbose, no loops/functions)
AWS CDK       = Write Python/TypeScript/Java/Go code
               → CDK generates CloudFormation from your code

CDK Example (Python):
from aws_cdk import (
    Stack,
    aws_ec2 as ec2,
    aws_ecs as ecs,
    aws_ecs_patterns as ecs_patterns
)
from constructs import Construct

class WebAppStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Create VPC (CDK creates 2 public + 2 private subnets automatically!)
        vpc = ec2.Vpc(self, "WebVPC",
            max_azs=2,
            nat_gateways=1
        )

        # Create ECS Fargate service with ALB in ONE construct!
        fargate_service = ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "WebService",
            vpc=vpc,
            cpu=512,
            memory_limit_mib=1024,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("nginx:latest"),
                container_port=80,
            ),
            public_load_balancer=True,
            desired_count=2
        )

# CDK turns this 30-line code into 500+ lines of CloudFormation YAML!

Deploy:
  cdk deploy  → synthesizes CloudFormation + deploys
  cdk diff    → shows what would change
  cdk destroy → deletes all resources
```

## 9. Interview Perspective

**Q: What is Infrastructure as Code and why does it matter?**
IaC means defining infrastructure in code files that can be version-controlled, reviewed, and executed repeatably. Benefits: (1) Consistency — no manual config drift between environments, (2) Version control — see every change ever made, (3) Repeatability — spin up identical prod/staging/dev environments, (4) Speed — deploy complex infrastructure in minutes, (5) Disaster recovery — rebuild entire environment from templates.

**Q: What is a CloudFormation Change Set?**
A Change Set lets you preview exactly what CloudFormation will add, modify, or delete before actually applying the changes. Create a Change Set, review it, then Execute if safe. This prevents accidental deletions or unexpected resource replacements.

**Q: What is the difference between CloudFormation and CDK?**
CloudFormation uses YAML/JSON templates — verbose but accessible without programming knowledge. CDK uses real programming languages (Python, TypeScript, Java) to define stacks as code. CDK synthesizes CloudFormation templates under the hood. CDK provides higher-level abstractions (e.g., one line to create an ECS service with ALB vs hundreds of YAML lines). CDK is preferred for complex applications where programming constructs (loops, functions, conditions) simplify the code.

---

**[🏠 Back to README](../README.md)**

**Prev:** [← OpenTelemetry on AWS](../stage-08_monitoring/otel.md) &nbsp;|&nbsp; **Next:** [CDK & Terraform →](../stage-09_iac/cdk_terraform.md)

**Related Topics:** [CDK & Terraform](../stage-09_iac/cdk_terraform.md) · [EC2](../stage-03_compute/ec2.md) · [ECS](../stage-10_containers/ecs.md) · [CI/CD Pipeline](../stage-13_devops_cicd/cicd_pipeline.md)
