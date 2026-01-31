# Deployment Guide

This guide provides step-by-step instructions for deploying the ECS Fargate application from scratch.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Network Configuration](#network-configuration)
4. [Container Setup](#container-setup)
5. [Service Deployment](#service-deployment)
6. [Post-Deployment](#post-deployment)

---

## Prerequisites

### Required Tools
- AWS CLI v2.x or higher
- Docker 20.x or higher
- Git
- Text editor (VS Code recommended)

### AWS Account Setup
```bash
# Configure AWS CLI
aws configure
# Enter:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region: eu-west-1
# - Default output format: json
```

### Verify Prerequisites
```bash
# Check AWS CLI
aws --version

# Check Docker
docker --version

# Verify AWS credentials
aws sts get-caller-identity
```

---

## Initial Setup

### 1. Clone Repository
```bash
git clone <your-repo-url>
cd ecs-fargate-project
```

### 2. Set Environment Variables
```bash
# Create environment file
cat > .env << EOF
AWS_REGION=eu-west-1
AWS_ACCOUNT_ID=<your-account-id>
PROJECT_NAME=ecs-fargate-webapp
ENVIRONMENT=production
EOF

# Load environment variables
source .env
```

### 3. Create S3 Bucket
```bash
# Create bucket for application data
aws s3 mb s3://my-web-app-data-09 --region eu-west-1

# Create uploads folder
aws s3api put-object \
  --bucket my-web-app-data-09 \
  --key uploads/

# Verify bucket creation
aws s3 ls
```

---

## Network Configuration

### Step 1: Create VPC
```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 192.168.0.0/24 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=ECS-VPC}]' \
  --query 'Vpc.VpcId' \
  --output text)

echo "VPC ID: $VPC_ID"

# Enable DNS hostnames
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames
```

### Step 2: Create Subnets
```bash
# Public Subnet 1 (AZ-1a)
PUBLIC_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 192.168.0.0/26 \
  --availability-zone eu-west-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1}]' \
  --query 'Subnet.SubnetId' \
  --output text)

# Public Subnet 2 (AZ-1b)
PUBLIC_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 192.168.0.64/26 \
  --availability-zone eu-west-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-2}]' \
  --query 'Subnet.SubnetId' \
  --output text)

# Private Subnet 1 (AZ-1a)
PRIVATE_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 192.168.0.128/26 \
  --availability-zone eu-west-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1}]' \
  --query 'Subnet.SubnetId' \
  --output text)

# Private Subnet 2 (AZ-1b)
PRIVATE_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 192.168.0.192/26 \
  --availability-zone eu-west-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-2}]' \
  --query 'Subnet.SubnetId' \
  --output text)

# Enable auto-assign public IP for public subnets
aws ec2 modify-subnet-attribute \
  --subnet-id $PUBLIC_SUBNET_1 \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id $PUBLIC_SUBNET_2 \
  --map-public-ip-on-launch

echo "Public Subnet 1: $PUBLIC_SUBNET_1"
echo "Public Subnet 2: $PUBLIC_SUBNET_2"
echo "Private Subnet 1: $PRIVATE_SUBNET_1"
echo "Private Subnet 2: $PRIVATE_SUBNET_2"
```

### Step 3: Create and Attach Internet Gateway
```bash
# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=IG_ECS}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

# Attach to VPC
aws ec2 attach-internet-gateway \
  --vpc-id $VPC_ID \
  --internet-gateway-id $IGW_ID

echo "Internet Gateway ID: $IGW_ID"
```

### Step 4: Create NAT Gateway
```bash
# Allocate Elastic IP
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

# Create NAT Gateway in public subnet
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET_1 \
  --allocation-id $EIP_ALLOC_ID \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=ECS-NAT}]' \
  --query 'NatGateway.NatGatewayId' \
  --output text)

# Wait for NAT Gateway to be available
echo "Waiting for NAT Gateway to be available..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID

echo "NAT Gateway ID: $NAT_GW_ID"
echo "Elastic IP Allocation: $EIP_ALLOC_ID"
```

### Step 5: Configure Route Tables
```bash
# Create Public Route Table
PUBLIC_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=RT-for-Public}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

# Add route to Internet Gateway
aws ec2 create-route \
  --route-table-id $PUBLIC_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Associate with public subnets
aws ec2 associate-route-table \
  --subnet-id $PUBLIC_SUBNET_1 \
  --route-table-id $PUBLIC_RT_ID

aws ec2 associate-route-table \
  --subnet-id $PUBLIC_SUBNET_2 \
  --route-table-id $PUBLIC_RT_ID

# Create Private Route Table
PRIVATE_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=RT-for-Private}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

# Add route to NAT Gateway
aws ec2 create-route \
  --route-table-id $PRIVATE_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_ID

# Associate with private subnets
aws ec2 associate-route-table \
  --subnet-id $PRIVATE_SUBNET_1 \
  --route-table-id $PRIVATE_RT_ID

aws ec2 associate-route-table \
  --subnet-id $PRIVATE_SUBNET_2 \
  --route-table-id $PRIVATE_RT_ID

echo "Public Route Table: $PUBLIC_RT_ID"
echo "Private Route Table: $PRIVATE_RT_ID"
```

### Step 6: Create Security Groups
```bash
# ALB Security Group
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name ALB-SG \
  --description "Security group for Application Load Balancer" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

# Allow HTTP traffic from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# ECS Tasks Security Group
ECS_SG_ID=$(aws ec2 create-security-group \
  --group-name ECS-Tasks-SG \
  --description "Security group for ECS Fargate tasks" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

# Allow traffic from ALB only
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 80 \
  --source-group $ALB_SG_ID

echo "ALB Security Group: $ALB_SG_ID"
echo "ECS Security Group: $ECS_SG_ID"
```

---

## Container Setup

### Step 1: Create ECR Repository
```bash
# Create repository
aws ecr create-repository \
  --repository-name cloudfolks-php-app \
  --image-scanning-configuration scanOnPush=true \
  --region eu-west-1

# Get repository URI
ECR_URI=$(aws ecr describe-repositories \
  --repository-names cloudfolks-php-app \
  --query 'repositories[0].repositoryUri' \
  --output text)

echo "ECR Repository URI: $ECR_URI"
```

### Step 2: Build and Push Docker Image
```bash
# Login to ECR
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS --password-stdin $ECR_URI

# Build Docker image (ensure you're in the app directory)
docker build -t cloudfolks-php-app .

# Tag image
docker tag cloudfolks-php-app:latest $ECR_URI:latest

# Push to ECR
docker push $ECR_URI:latest

# Verify push
aws ecr describe-images \
  --repository-name cloudfolks-php-app \
  --query 'imageDetails[0].[imageTags[0],imageSizeInBytes]' \
  --output table
```

### Step 3: Create IAM Roles

#### Task Execution Role
```bash
# Create trust policy
cat > task-execution-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://task-execution-trust-policy.json

# Attach managed policy
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

#### Task Role (for S3 access)
```bash
# Create task role policy
cat > task-role-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-web-app-data-09",
        "arn:aws:s3:::my-web-app-data-09/*"
      ]
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name ecsTaskRole \
  --assume-role-policy-document file://task-execution-trust-policy.json

# Create and attach policy
aws iam put-role-policy \
  --role-name ecsTaskRole \
  --policy-name ECS-S3-Access \
  --policy-document file://task-role-policy.json
```

### Step 4: Register Task Definition
```bash
# Update task definition with your account ID and ECR URI
sed -i "s/<ACCOUNT_ID>/$AWS_ACCOUNT_ID/g" docs/task-definition.json

# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://docs/task-definition.json

# Verify registration
aws ecs describe-task-definition \
  --task-definition My-Web-App-Final \
  --query 'taskDefinition.[family,revision,status]' \
  --output table
```

---

## Service Deployment

### Step 1: Create ECS Cluster
```bash
# Create Fargate cluster
aws ecs create-cluster \
  --cluster-name my-fargate-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1

# Verify cluster
aws ecs describe-clusters \
  --clusters my-fargate-cluster \
  --query 'clusters[0].[clusterName,status,registeredContainerInstancesCount]' \
  --output table
```

### Step 2: Create Application Load Balancer

#### Create Target Group
```bash
# Create target group
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name ecs-app-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path / \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

echo "Target Group ARN: $TARGET_GROUP_ARN"
```

#### Create Load Balancer
```bash
# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name ecs-app-alb \
  --subnets $PUBLIC_SUBNET_1 $PUBLIC_SUBNET_2 \
  --security-groups $ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

# Wait for ALB to be active
echo "Waiting for ALB to be active..."
aws elbv2 wait load-balancer-available --load-balancer-arns $ALB_ARN

# Get ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

echo "ALB ARN: $ALB_ARN"
echo "ALB DNS: $ALB_DNS"
```

#### Create Listener
```bash
# Create HTTP listener
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_ARN

echo "Listener created successfully"
```

### Step 3: Create ECS Service
```bash
# Create service with ALB integration
aws ecs create-service \
  --cluster my-fargate-cluster \
  --service-name My-Web-App-Final-service \
  --task-definition My-Web-App-Final:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --platform-version LATEST \
  --network-configuration "awsvpcConfiguration={subnets=[$PRIVATE_SUBNET_1,$PRIVATE_SUBNET_2],securityGroups=[$ECS_SG_ID],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=$TARGET_GROUP_ARN,containerName=cloudfolks-php-app,containerPort=80" \
  --health-check-grace-period-seconds 60

# Wait for service to be stable
echo "Waiting for service to stabilize..."
aws ecs wait services-stable \
  --cluster my-fargate-cluster \
  --services My-Web-App-Final-service

echo "Service created and stable!"
```

### Step 4: Configure Auto Scaling
```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-fargate-cluster/My-Web-App-Final-service \
  --min-capacity 2 \
  --max-capacity 8

# Create scale-out scheduled action
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-fargate-cluster/My-Web-App-Final-service \
  --scheduled-action-name SC-Weekend-Scale-Out \
  --schedule "cron(0 8 * * MON-FRI *)" \
  --scalable-target-action MinCapacity=4,MaxCapacity=10

# Create scale-in scheduled action
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-fargate-cluster/My-Web-App-Final-service \
  --scheduled-action-name SC-Weekend-Scale-IN \
  --schedule "cron(0 18 * * MON-FRI *)" \
  --scalable-target-action MinCapacity=2,MaxCapacity=2

echo "Auto scaling configured successfully"
```

---

## Post-Deployment

### Verification Steps

#### 1. Check Service Status
```bash
aws ecs describe-services \
  --cluster my-fargate-cluster \
  --services My-Web-App-Final-service \
  --query 'services[0].[serviceName,status,runningCount,desiredCount]' \
  --output table
```

#### 2. Check Target Health
```bash
aws elbv2 describe-target-health \
  --target-group-arn $TARGET_GROUP_ARN \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' \
  --output table
```

#### 3. Test Application
```bash
# Test ALB endpoint
echo "Testing application at: http://$ALB_DNS"
curl -I http://$ALB_DNS

# Full response
curl http://$ALB_DNS
```

#### 4. Check CloudWatch Logs
```bash
# List log streams
aws logs describe-log-streams \
  --log-group-name /ecs/my-web-app-final \
  --order-by LastEventTime \
  --descending \
  --max-items 5

# Tail logs
aws logs tail /ecs/my-web-app-final --follow
```

### Save Configuration
```bash
# Save all IDs for future reference
cat > deployment-config.txt << EOF
VPC_ID=$VPC_ID
PUBLIC_SUBNET_1=$PUBLIC_SUBNET_1
PUBLIC_SUBNET_2=$PUBLIC_SUBNET_2
PRIVATE_SUBNET_1=$PRIVATE_SUBNET_1
PRIVATE_SUBNET_2=$PRIVATE_SUBNET_2
IGW_ID=$IGW_ID
NAT_GW_ID=$NAT_GW_ID
ALB_SG_ID=$ALB_SG_ID
ECS_SG_ID=$ECS_SG_ID
ECR_URI=$ECR_URI
TARGET_GROUP_ARN=$TARGET_GROUP_ARN
ALB_ARN=$ALB_ARN
ALB_DNS=$ALB_DNS
EOF

echo "Configuration saved to deployment-config.txt"
```

---

## Cleanup (Optional)

To delete all resources and avoid charges:

```bash
# Delete ECS service
aws ecs update-service \
  --cluster my-fargate-cluster \
  --service My-Web-App-Final-service \
  --desired-count 0

aws ecs delete-service \
  --cluster my-fargate-cluster \
  --service My-Web-App-Final-service \
  --force

# Delete ECS cluster
aws ecs delete-cluster --cluster my-fargate-cluster

# Delete ALB
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 delete-target-group --target-group-arn $TARGET_GROUP_ARN

# Delete NAT Gateway and release EIP
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW_ID
aws ec2 release-address --allocation-id $EIP_ALLOC_ID

# Delete Internet Gateway
aws ec2 detach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# Delete subnets
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_1
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_2
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_1
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_2

# Delete VPC
aws ec2 delete-vpc --vpc-id $VPC_ID

# Delete S3 bucket
aws s3 rb s3://my-web-app-data-09 --force

# Delete ECR repository
aws ecr delete-repository --repository-name cloudfolks-php-app --force
```

---

## Troubleshooting

If you encounter issues during deployment:

1. **Check AWS CLI version**: `aws --version` (should be 2.x)
2. **Verify credentials**: `aws sts get-caller-identity`
3. **Check region**: Ensure all commands use `--region eu-west-1`
4. **Review CloudWatch logs**: Look for errors in ECS task logs
5. **Validate JSON files**: Use `jq` to validate JSON syntax
6. **Check security groups**: Ensure proper ingress/egress rules

For detailed troubleshooting, see the main [README.md](../README.md#troubleshooting) file.

---

## Next Steps

1. Set up CI/CD pipeline (AWS CodePipeline)
2. Configure CloudWatch alarms
3. Implement blue/green deployments
4. Add SSL/TLS certificate to ALB
5. Configure custom domain name
6. Implement WAF rules
7. Set up automated backups

---

*Happy Deploying! 🚀*
