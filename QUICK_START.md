# Quick Start Guide

Get your ECS Fargate application up and running in 30 minutes!

## What You'll Deploy

- ✅ Complete VPC with public and private subnets
- ✅ Application Load Balancer
- ✅ ECS Fargate cluster with 2 tasks
- ✅ Auto-scaling configuration
- ✅ S3 bucket for storage
- ✅ Container image in ECR

**Estimated Time**: 30-45 minutes  
**Estimated Cost**: ~$72-100/month

---

## Prerequisites Checklist

Before you begin, ensure you have:

- [ ] AWS Account with administrator access
- [ ] AWS CLI installed and configured (`aws --version`)
- [ ] Docker installed (`docker --version`)
- [ ] Git installed (`git --version`)
- [ ] Your application Docker image ready (or use our sample)

### AWS CLI Setup

```bash
# Install AWS CLI (if not installed)
# macOS: brew install awscli
# Linux: pip install awscli
# Windows: Download from aws.amazon.com

# Configure AWS CLI
aws configure
# Enter your:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region: eu-west-1
# - Output format: json

# Verify configuration
aws sts get-caller-identity
```

---

## Step-by-Step Deployment

### Step 1: Clone This Repository (5 minutes)

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/ecs-fargate-project.git
cd ecs-fargate-project

# Make scripts executable (if any)
chmod +x scripts/*.sh
```

### Step 2: Set Your Configuration (2 minutes)

```bash
# Copy environment template
cp .env.example .env

# Edit with your details
nano .env
# or
vim .env
# or
code .env
```

Required variables:
```bash
AWS_REGION=eu-west-1
AWS_ACCOUNT_ID=123456789012  # Replace with your account ID
PROJECT_NAME=ecs-fargate-webapp
ENVIRONMENT=production
```

Get your AWS Account ID:
```bash
aws sts get-caller-identity --query Account --output text
```

### Step 3: Create S3 Bucket (1 minute)

```bash
# Create bucket for application data
aws s3 mb s3://my-web-app-data-09 --region eu-west-1

# Verify
aws s3 ls | grep my-web-app-data-09
```

### Step 4: Build Network Infrastructure (10 minutes)

Follow the commands in [DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md) or use this condensed version:

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 192.168.0.0/24 --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
echo "VPC_ID=$VPC_ID" >> .env

# Create subnets (repeat for all 4)
# See full commands in DEPLOYMENT_GUIDE.md

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Create NAT Gateway (in public subnet)
# Allocate Elastic IP first
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_GW=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_SUBNET_1 --allocation-id $EIP_ALLOC --query 'NatGateway.NatGatewayId' --output text)

# Wait for NAT Gateway
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

# Configure route tables
# See full commands in DEPLOYMENT_GUIDE.md
```

**TIP**: Save all resource IDs as you create them!

### Step 5: Setup Security Groups (3 minutes)

```bash
# Create ALB Security Group
ALB_SG=$(aws ec2 create-security-group \
  --group-name ALB-SG \
  --description "Security group for ALB" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Allow HTTP traffic
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# Create ECS Tasks Security Group
ECS_SG=$(aws ec2 create-security-group \
  --group-name ECS-Tasks-SG \
  --description "Security group for ECS tasks" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Allow traffic from ALB only
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG \
  --protocol tcp --port 80 --source-group $ALB_SG
```

### Step 6: Container Setup (7 minutes)

```bash
# Create ECR repository
aws ecr create-repository \
  --repository-name cloudfolks-php-app \
  --image-scanning-configuration scanOnPush=true

# Get ECR URI
ECR_URI=$(aws ecr describe-repositories \
  --repository-names cloudfolks-php-app \
  --query 'repositories[0].repositoryUri' \
  --output text)

# Login to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI

# Build and push (if you have your Dockerfile)
docker build -t cloudfolks-php-app .
docker tag cloudfolks-php-app:latest $ECR_URI:latest
docker push $ECR_URI:latest
```

### Step 7: Create ECS Cluster (2 minutes)

```bash
# Create Fargate cluster
aws ecs create-cluster \
  --cluster-name my-fargate-cluster \
  --capacity-providers FARGATE FARGATE_SPOT

# Verify
aws ecs describe-clusters --clusters my-fargate-cluster
```

### Step 8: Deploy Load Balancer (5 minutes)

```bash
# Create target group
TG_ARN=$(aws elbv2 create-target-group \
  --name ecs-app-tg \
  --protocol HTTP --port 80 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path / \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name ecs-app-alb \
  --subnets $PUBLIC_SUBNET_1 $PUBLIC_SUBNET_2 \
  --security-groups $ALB_SG \
  --scheme internet-facing \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

# Wait for ALB
aws elbv2 wait load-balancer-available --load-balancer-arns $ALB_ARN

# Get DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' \
  --output text)
echo "Your ALB DNS: $ALB_DNS"

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### Step 9: Register Task Definition (3 minutes)

```bash
# Update task definition with your account ID
sed -i "s/<ACCOUNT_ID>/$(aws sts get-caller-identity --query Account --output text)/g" docs/task-definition.json

# Register
aws ecs register-task-definition \
  --cli-input-json file://docs/task-definition.json

# Verify
aws ecs list-task-definitions
```

### Step 10: Create ECS Service (5 minutes)

```bash
# Create service
aws ecs create-service \
  --cluster my-fargate-cluster \
  --service-name My-Web-App-Final-service \
  --task-definition My-Web-App-Final:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$PRIVATE_SUBNET_1,$PRIVATE_SUBNET_2],securityGroups=[$ECS_SG],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=cloudfolks-php-app,containerPort=80" \
  --health-check-grace-period-seconds 60

# Wait for stable (may take 2-3 minutes)
aws ecs wait services-stable \
  --cluster my-fargate-cluster \
  --services My-Web-App-Final-service
```

### Step 11: Configure Auto Scaling (2 minutes)

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-fargate-cluster/My-Web-App-Final-service \
  --min-capacity 2 --max-capacity 8

# Create scheduled scale-out
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-fargate-cluster/My-Web-App-Final-service \
  --scheduled-action-name SC-Weekend-Scale-Out \
  --schedule "cron(0 8 * * MON-FRI *)" \
  --scalable-target-action MinCapacity=4,MaxCapacity=10

# Create scheduled scale-in
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-fargate-cluster/My-Web-App-Final-service \
  --scheduled-action-name SC-Weekend-Scale-IN \
  --schedule "cron(0 18 * * MON-FRI *)" \
  --scalable-target-action MinCapacity=2,MaxCapacity=2
```

---

## Verification (2 minutes)

### Check Everything is Working

```bash
# 1. Check ECS service status
aws ecs describe-services \
  --cluster my-fargate-cluster \
  --services My-Web-App-Final-service \
  --query 'services[0].[serviceName,status,runningCount,desiredCount]'

# 2. Check target health
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN

# 3. Test your application
echo "Testing: http://$ALB_DNS"
curl -I http://$ALB_DNS

# 4. Open in browser
echo "Open this URL in your browser:"
echo "http://$ALB_DNS"
```

### Expected Results

✅ **Service Status**: ACTIVE  
✅ **Running Count**: 2  
✅ **Target Health**: healthy  
✅ **HTTP Response**: 200 OK  

---

## What's Next?

### Immediate Next Steps

1. **Access Your Application**
   ```bash
   open http://$ALB_DNS  # macOS
   # or visit in browser
   ```

2. **Monitor Your Deployment**
   - Go to AWS Console → ECS → Clusters → my-fargate-cluster
   - Check CloudWatch Logs: `/ecs/my-web-app-final`

3. **Test Auto Scaling**
   - Wait for scheduled scaling time
   - Watch tasks scale up/down automatically

### Recommended Enhancements

- [ ] Add SSL/TLS certificate (AWS Certificate Manager)
- [ ] Set up CloudWatch dashboards
- [ ] Configure custom domain name (Route 53)
- [ ] Implement CI/CD pipeline (CodePipeline)
- [ ] Add Web Application Firewall (WAF)
- [ ] Enable CloudFront CDN
- [ ] Set up automated backups

---

## Common Issues & Quick Fixes

### Issue 1: Tasks Not Starting

```bash
# Check task status
aws ecs describe-tasks \
  --cluster my-fargate-cluster \
  --tasks $(aws ecs list-tasks --cluster my-fargate-cluster --query 'taskArns[0]' --output text)

# Common fixes:
# - Verify ECR image URI in task definition
# - Check IAM execution role permissions
# - Ensure sufficient vCPU/memory
```

### Issue 2: Unhealthy Targets

```bash
# Check target health details
aws elbv2 describe-target-health --target-group-arn $TG_ARN

# Common fixes:
# - Verify container port matches (80)
# - Check security group allows ALB → Tasks traffic
# - Ensure health check path returns 200 OK
```

### Issue 3: Can't Access ALB

```bash
# Verify ALB is active
aws elbv2 describe-load-balancers --names ecs-app-alb

# Common fixes:
# - Check ALB security group allows inbound HTTP
# - Verify route table has IGW route
# - Ensure ALB is in "active" state
```

---

## Cost Management

### Monitor Your Costs

```bash
# Check current month costs
aws ce get-cost-and-usage \
  --time-period Start=2026-01-01,End=2026-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=SERVICE
```

### Cost-Saving Tips

1. **Use Scheduled Scaling** (Already configured!)
   - Saves 30-40% by reducing capacity during off-peak

2. **Consider Fargate Spot**
   - Save up to 70% for non-critical workloads
   - Add to capacity provider mix

3. **Right-Size Your Tasks**
   - Monitor CPU/Memory usage
   - Adjust vCPU/memory if over-provisioned

4. **Optimize NAT Gateway Usage**
   - Consider VPC endpoints for AWS services
   - Reduce data transfer through NAT

---

## Cleanup Instructions

### When You're Done Testing

```bash
# Delete in reverse order to avoid dependency issues

# 1. Delete ECS service
aws ecs update-service \
  --cluster my-fargate-cluster \
  --service My-Web-App-Final-service \
  --desired-count 0

aws ecs delete-service \
  --cluster my-fargate-cluster \
  --service My-Web-App-Final-service --force

# 2. Delete ECS cluster
aws ecs delete-cluster --cluster my-fargate-cluster

# 3. Delete ALB and target group
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
sleep 10
aws elbv2 delete-target-group --target-group-arn $TG_ARN

# 4. Delete NAT Gateway and release EIP
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW
aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_GW
aws ec2 release-address --allocation-id $EIP_ALLOC

# 5. Delete Internet Gateway
aws ec2 detach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# 6. Delete subnets and VPC
# (Get subnet IDs from .env or describe-subnets)
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_1
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_2
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_1
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_2
aws ec2 delete-vpc --vpc-id $VPC_ID

# 7. Delete S3 bucket
aws s3 rb s3://my-web-app-data-09 --force

# 8. Delete ECR repository
aws ecr delete-repository --repository-name cloudfolks-php-app --force
```

**Estimated time to delete**: 5-10 minutes

---

## Getting Help

### Resources

- 📖 [Full Documentation](README.md)
- 🏗️ [Architecture Details](docs/ARCHITECTURE.md)
- 📝 [Detailed Deployment Guide](docs/DEPLOYMENT_GUIDE.md)
- 🐛 [Troubleshooting Guide](README.md#troubleshooting)

### Support

- 🐙 [Open an Issue](https://github.com/Shakarr009/ecs-fargate-project/issues)
- 💬 [GitHub Discussions](https://github.com/Shankarr009/ecs-fargate-web-app-deploymen)
- 📧 [Email Support](shankarsuthar499@gmail.com)

---

## Congratulations! 🎉

You've successfully deployed a production-ready web application on AWS ECS Fargate!

**What you've accomplished:**
- ✅ Built secure network infrastructure
- ✅ Deployed containerized application
- ✅ Set up load balancing and auto-scaling
- ✅ Implemented high availability across multiple AZs
- ✅ Configured automated scaling policies

**Next steps:**
- Customize for your specific needs
- Add monitoring and alerts
- Implement CI/CD pipeline
- Share your success! 🚀

---

*Happy deploying! If you found this helpful, please star the repository ⭐*
