# Highly Available Auto-Scaling Web Tier

A CloudFormation lab that provisions a production-style, highly available web tier on AWS. EC2 instances run Apache behind an Application Load Balancer and scale automatically based on CPU utilisation. Deployment is fully automated via CloudFormation Git sync — every push to the linked branch triggers a stack update.

## Architecture

```
Internet
    │
    ▼
[Application Load Balancer]  ← public subnets (AZ-1, AZ-2)
    │
    ▼
[Auto Scaling Group]         ← private subnets (AZ-1, AZ-2)
  ├── EC2 instance (AZ-1)
  └── EC2 instance (AZ-2)
    │
    ▼
[NAT Gateway]                ← outbound internet (yum, SSM)
```

### Resources provisioned

| Resource | Description |
|---|---|
| VPC | Isolated network (`10.0.0.0/16`) |
| Public Subnets (×2) | ALB and NAT Gateway, one per AZ |
| Private Subnets (×2) | EC2 instances, one per AZ |
| Internet Gateway | Inbound/outbound for public subnets |
| NAT Gateway | Outbound-only internet for private subnets |
| ALB + Listener | Internet-facing, HTTP port 80 |
| Target Group | Health checks every 15s on `/` |
| Launch Template | Amazon Linux 2, Apache, stress tool |
| Auto Scaling Group | Min 1 / Desired 1 / Max 4 instances |
| CPU Scaling Policy | Target tracking at 30% average CPU |
| IAM Role + Profile | SSM Session Manager access (no SSH) |

### Security model

- EC2 instances have **no public IP** and sit in private subnets.
- The EC2 security group only allows port 80 **from the ALB security group** — direct instance access from the internet is impossible.
- **No SSH key pair** is configured. Instance access is via AWS Systems Manager Session Manager only.

## Prerequisites

- AWS account with permissions for EC2, ELB, AutoScaling, IAM, and CloudFormation.
- A CloudFormation execution role (e.g. `cloudFormation-role`) with the following managed policies attached:
  - `AmazonEC2FullAccess`
  - `ElasticLoadBalancingFullAccess`
  - `AutoScalingFullAccess`
  - `IAMFullAccess`
- A GitHub repository connected to CloudFormation Git sync.

## Repository structure

```
auto-scaling-lab/
├── template.yaml      # CloudFormation template (all infrastructure)
├── deployment.yaml    # Git sync deployment configuration
└── README.md
```

## Deployment

### Option 1 — CloudFormation Git sync (recommended)

Git sync watches the linked branch and automatically deploys on every push.

1. In the AWS Console go to **CloudFormation → Stacks → Create stack → With Git sync**.
2. Connect your GitHub repository and select the branch to watch.
3. Point Git sync at `deployment.yaml` as the deployment configuration file.
4. Every `git push` to the linked branch triggers a change set and deploys automatically.

### Option 2 — AWS CLI (manual)

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name autoscaling-demo \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
      EnvironmentName=autoscaling-demo \
      InstanceType=t3.micro \
      AsgMinSize=1 \
      AsgDesiredSize=1 \
      AsgMaxSize=4 \
      CpuScaleOutThreshold=30
```

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `EnvironmentName` | `autoscaling-demo` | Prefix applied to all resource names |
| `VpcCidr` | `10.0.0.0/16` | VPC CIDR block |
| `PublicSubnet1Cidr` | `10.0.1.0/24` | Public subnet AZ-1 |
| `PublicSubnet2Cidr` | `10.0.2.0/24` | Public subnet AZ-2 |
| `PrivateSubnet1Cidr` | `10.0.11.0/24` | Private subnet AZ-1 |
| `PrivateSubnet2Cidr` | `10.0.12.0/24` | Private subnet AZ-2 |
| `InstanceType` | `t3.micro` | Allowed: `t2.micro`, `t3.micro`, `t3.small` |
| `AsgMinSize` | `1` | Minimum number of instances |
| `AsgDesiredSize` | `1` | Desired number of instances at launch |
| `AsgMaxSize` | `4` | Maximum number of instances |
| `CpuScaleOutThreshold` | `30` | Average CPU % that triggers scale-out |

## Outputs

After deployment the stack exports:

| Output | Description |
|---|---|
| `AlbDnsName` | Public URL of the ALB — open this in a browser |
| `VpcId` | VPC ID |
| `AutoScalingGroupName` | ASG name |
| `PrivateSubnet1Id` | Private subnet 1 ID |
| `PrivateSubnet2Id` | Private subnet 2 ID |

Retrieve the ALB URL via CLI:

```bash
aws cloudformation describe-stacks \
  --stack-name autoscaling-demo \
  --query "Stacks[0].Outputs[?OutputKey=='AlbDnsName'].OutputValue" \
  --output text
```

## Demo walkthrough

### 1. Verify the app is running

Open the ALB DNS name in a browser. You should see a page showing the instance's ID, private IP, and Availability Zone. The page auto-refreshes every 5 seconds.

### 2. Trigger the CPU stress test

Click **"Trigger CPU Stress"** on the page. This hits the `/stress` CGI endpoint on that specific instance, which runs:

```bash
stress --cpu 2 --timeout 300 &
```

Two CPU worker processes run for 5 minutes, spiking that instance's CPU.

### 3. Watch Auto Scaling react

CloudWatch collects CPU metrics every minute. Once the average CPU across the ASG exceeds 30%, the target tracking policy triggers and the ASG launches new instances (up to the configured maximum of 4).

Monitor the scale-out in real time:

```
EC2 → Auto Scaling Groups → autoscaling-demo-asg → Activity tab
```

### 4. Confirm new instances are serving traffic

Keep refreshing the ALB URL — the Instance ID and Availability Zone on the page will change as the ALB round-robins across all healthy instances.

### 5. Watch scale-in

After the stress test ends (300 seconds), CPU drops back below 30%. The ASG waits out a cooldown period then terminates the extra instances, returning to the desired capacity of 1.

## Instance access (no SSH)

Instances have no SSH key and no public IP. Use AWS Systems Manager Session Manager instead:

```bash
aws ssm start-session --target <instance-id> --region us-east-2
```

Or via the AWS Console: **EC2 → Instances → select instance → Connect → Session Manager**.

## Teardown

```bash
aws cloudformation delete-stack --stack-name autoscaling-demo
```

> **Note:** The NAT Gateway incurs hourly charges even when idle. Delete the stack when the lab is complete to avoid ongoing costs.
