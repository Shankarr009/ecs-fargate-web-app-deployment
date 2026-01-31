# 🚀 Production-Ready Web Application Deployment on AWS ECS Fargate

[![AWS](https://img.shields.io/badge/AWS-ECS%20Fargate-orange?style=flat-square&logo=amazon-aws)](https://aws.amazon.com/ecs/)
[![Docker](https://img.shields.io/badge/Docker-Container-blue?style=flat-square&logo=docker)](https://www.docker.com/)
[![Status](https://img.shields.io/badge/Status-Production-success?style=flat-square)]()
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)]()

A complete end-to-end implementation of a scalable, highly-available web application deployed on **Amazon ECS Fargate** with automated scaling, load balancing, and secure network architecture.

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Features](#-features)
- [Prerequisites](#-prerequisites)
- [Infrastructure Components](#-infrastructure-components)
- [Deployment Steps](#-deployment-steps)
- [Auto Scaling Configuration](#-auto-scaling-configuration)
- [Security](#-security)
- [Monitoring & Verification](#-monitoring--verification)
- [Cost Optimization](#-cost-optimization)
- [Troubleshooting](#-troubleshooting)
- [Best Practices](#-best-practices)
- [Screenshots](#-screenshots)
- [Contributing](#-contributing)
- [License](#-license)

---

## 🎯 Overview

This project demonstrates a **production-grade containerized application deployment** on AWS using ECS Fargate. The architecture eliminates the operational overhead of managing EC2 instances while providing automatic scaling, high availability, and secure networking.

### Key Highlights

- **Serverless Containers**: No EC2 instance management required
- **Multi-AZ Deployment**: High availability across 2 Availability Zones
- **Auto Scaling**: Dynamic task scaling based on scheduled policies (2-8 tasks)
- **Load Balanced**: Application Load Balancer distributing traffic
- **Secure Architecture**: Private subnets for tasks, public subnets for ALB
- **Container Registry**: Docker images stored in Amazon ECR
- **Storage Integration**: S3 bucket for application data persistence

---

## 🏗️ Architecture

### High-Level Architecture Diagram

```
                                    Internet
                                       |
                                       |
                            ┌──────────▼───────────┐
                            │   Internet Gateway   │
                            └──────────┬───────────┘
                                       |
                ┌──────────────────────┴──────────────────────┐
                |           Application Load Balancer          |
                |              (ecs-app-alb)                   |
                |         Public Subnets (2 AZs)               |
                └──────────────────────┬──────────────────────┘
                                       |
                        ┌──────────────┴──────────────┐
                        |                             |
                ┌───────▼────────┐          ┌────────▼───────┐
                │  ECS Task      │          │  ECS Task      │
                │  (Fargate)     │          │  (Fargate)     │
                │  AZ 1a         │          │  AZ 1b         │
                │  Private Subnet│          │  Private Subnet│
                └────────┬───────┘          └────────┬───────┘
                         |                           |
                         └───────────┬───────────────┘
                                     |
                            ┌────────▼─────────┐
                            │   NAT Gateway    │
                            └────────┬─────────┘
                                     |
                            ┌────────▼─────────┐
                            │   ECR + S3       │
                            └──────────────────┘
```

### Network Architecture

- **VPC CIDR**: `192.168.0.0/24`
- **Public Subnets**: 
  - `192.168.0.0/26` (AZ-1a)
  - `192.168.0.64/26` (AZ-1b)
- **Private Subnets**: 
  - `192.168.0.128/26` (AZ-1a)
  - `192.168.0.192/26` (AZ-1b)

---

## ✨ Features

### Infrastructure Features
- ✅ **Custom VPC** with public and private subnets
- ✅ **Internet Gateway** for public internet access
- ✅ **NAT Gateway** for secure outbound connectivity from private subnets
- ✅ **Application Load Balancer** with health checks
- ✅ **ECS Fargate Cluster** (my-fargate-cluster)
- ✅ **Target Groups** with automatic task registration
- ✅ **Route Tables** for traffic management

### Application Features
- ✅ **Containerized PHP Application** (cloudfolks-php-app)
- ✅ **Container Size**: 176.76 MB
- ✅ **Environment Variables**: AWS_REGION, S3_BUCKET
- ✅ **S3 Integration** for data persistence
- ✅ **CloudWatch Logs** for monitoring

### Security Features
- ✅ **Security Groups** with least-privilege access
- ✅ **Private Task Placement** (no public IPs)
- ✅ **IAM Roles** (Task Execution Role + Task Role)
- ✅ **Network Isolation** between layers

### Scaling Features
- ✅ **Scheduled Auto Scaling** policies
- ✅ **Scale-Out**: 2 → 4-10 tasks during peak hours
- ✅ **Scale-In**: 4 → 2 tasks during off-peak hours
- ✅ **Cost-Optimized** task management

---

## 📦 Prerequisites

Before deploying this infrastructure, ensure you have:

### Required
- AWS Account with appropriate permissions
- AWS CLI configured (`aws configure`)
- Docker installed locally (for building images)
- Basic understanding of:
  - AWS VPC and networking
  - Docker containers
  - ECS concepts (tasks, services, clusters)

### AWS Services Used
- Amazon ECS (Fargate)
- Amazon ECR (Elastic Container Registry)
- Amazon VPC
- Application Load Balancer
- Amazon S3
- AWS IAM
- Amazon CloudWatch

---

## 🔧 Infrastructure Components

### 1. Networking Layer

| Component | Configuration | Purpose |
|-----------|--------------|---------|
| VPC | `192.168.0.0/24` | Isolated network environment |
| Public Subnets | 2 subnets across 2 AZs | Host ALB |
| Private Subnets | 2 subnets across 2 AZs | Host ECS tasks |
| Internet Gateway | 1 IGW | Public internet access |
| NAT Gateway | 1 NGW in public subnet | Outbound internet for private subnets |
| Route Tables | 2 tables (public/private) | Traffic routing |

### 2. Compute Layer

| Component | Configuration | Details |
|-----------|--------------|---------|
| ECS Cluster | my-fargate-cluster | Fargate launch type |
| Task Definition | My-Web-App-Final | Latest revision |
| Service | My-Web-App-Final-service | 2 desired tasks |
| Container Image | cloudfolks-php-app:latest | 176.76 MB |
| ECR Repository | cloudfolks-php-app | Private registry |

### 3. Load Balancing

| Component | Configuration | Details |
|-----------|--------------|---------|
| ALB | ecs-app-alb | Internet-facing |
| Target Group | ecs-app-tg | IP target type |
| Listener | HTTP:80 | Forward to target group |
| Health Checks | / (root path) | 30s interval |
| Registered Targets | 2 healthy targets | Auto-registered by ECS |

### 4. Security

| Component | Configuration | Details |
|-----------|--------------|---------|
| ALB Security Group | ALB-SG | HTTP/HTTPS from 0.0.0.0/0 |
| ECS Security Group | ECS-Tasks-SG | Traffic only from ALB-SG |
| IAM Execution Role | ecsTaskExecutionRole | ECR pull + CloudWatch logs |
| IAM Task Role | Custom role | S3 read/write permissions |

### 5. Storage

| Component | Configuration | Details |
|-----------|--------------|---------|
| S3 Bucket | my-web-app-data-09 | Application data storage |
| Folder Structure | /uploads | User-uploaded content |

---

## 🚀 Deployment Steps

### Step 1: Build Secure Network Environment

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 192.168.0.0/24 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=ECS-VPC}]'

# Create Subnets (repeat for all 4 subnets)
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 192.168.0.0/26 --availability-zone eu-west-1a

# Create and attach Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id <vpc-id> --internet-gateway-id <igw-id>

# Create NAT Gateway
aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id <public-subnet-id> --allocation-id <eip-allocation-id>

# Configure Route Tables
aws ec2 create-route-table --vpc-id <vpc-id>
aws ec2 create-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <igw-id>
```

### Step 2: Configure Security Groups

```bash
# ALB Security Group
aws ec2 create-security-group --group-name ALB-SG --description "Security group for ALB" --vpc-id <vpc-id>
aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol tcp --port 80 --cidr 0.0.0.0/0

# ECS Tasks Security Group
aws ec2 create-security-group --group-name ECS-Tasks-SG --description "Security group for ECS tasks" --vpc-id <vpc-id>
aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol tcp --port 80 --source-group <alb-sg-id>
```

### Step 3: Create ECS Cluster

```bash
# Create Fargate cluster
aws ecs create-cluster --cluster-name my-fargate-cluster --capacity-providers FARGATE FARGATE_SPOT
```

### Step 4: Deploy Application Load Balancer

```bash
# Create Target Group
aws elbv2 create-target-group \
  --name ecs-app-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id <vpc-id> \
  --target-type ip \
  --health-check-path /

# Create Application Load Balancer
aws elbv2 create-load-balancer \
  --name ecs-app-alb \
  --subnets <public-subnet-1> <public-subnet-2> \
  --security-groups <alb-sg-id> \
  --scheme internet-facing \
  --type application

# Create Listener
aws elbv2 create-listener \
  --load-balancer-arn <alb-arn> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<tg-arn>
```

### Step 5: Build and Push Docker Image

```bash
# Login to ECR
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.eu-west-1.amazonaws.com

# Build Docker image
docker build -t cloudfolks-php-app .

# Tag image
docker tag cloudfolks-php-app:latest <account-id>.dkr.ecr.eu-west-1.amazonaws.com/cloudfolks-php-app:latest

# Push to ECR
docker push <account-id>.dkr.ecr.eu-west-1.amazonaws.com/cloudfolks-php-app:latest
```

### Step 6: Create Task Definition

```bash
# Register task definition (use JSON file)
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

### Step 7: Create ECS Service

```bash
# Create ECS service with ALB integration
aws ecs create-service \
  --cluster my-fargate-cluster \
  --service-name My-Web-App-Final-service \
  --task-definition My-Web-App-Final:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[<private-subnet-1>,<private-subnet-2>],securityGroups=[<ecs-sg-id>],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=<tg-arn>,containerName=cloudfolks-php-app,containerPort=80"
```

### Step 8: Enable Auto Scaling

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-fargate-cluster/My-Web-App-Final-service \
  --min-capacity 2 \
  --max-capacity 8

# Create scheduled scale-out action
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-fargate-cluster/My-Web-App-Final-service \
  --scheduled-action-name SC-Weekend-Scale-Out \
  --schedule "cron(26 15 * * ? *)" \
  --scalable-target-action MinCapacity=4,MaxCapacity=10

# Create scheduled scale-in action
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-fargate-cluster/My-Web-App-Final-service \
  --scheduled-action-name SC-Weekend-Scale-IN \
  --schedule "cron(28 15 * * ? *)" \
  --scalable-target-action MinCapacity=2,MaxCapacity=2
```

---

## 📊 Auto Scaling Configuration

### Scheduled Scaling Policies

| Policy Name | Type | Schedule | Min Tasks | Max Tasks | Purpose |
|-------------|------|----------|-----------|-----------|---------|
| SC-Weekend-Scale-Out | Scale Out | 15:26 UTC | 4 | 10 | Increase capacity during peak hours |
| SC-Weekend-Scale-IN | Scale In | 15:28 UTC | 2 | 2 | Reduce capacity during off-peak hours |

### Scaling Behavior

```
Current State: 2 tasks running
  ↓
Scale-Out Trigger (15:26 UTC)
  ↓
Target State: 4-10 tasks available
  ↓
Scale-In Trigger (15:28 UTC)
  ↓
Back to: 2 tasks running
```

### Benefits
- **Cost Optimization**: Pay only for resources needed
- **Performance**: More tasks during high-traffic periods
- **Predictable**: Schedule-based, not reactive
- **Flexible**: Easy to adjust based on traffic patterns

---

## 🔒 Security

### Network Security

1. **Multi-Layer Defense**
   - Public subnets: Only ALB exposed to internet
   - Private subnets: ECS tasks isolated
   - NAT Gateway: One-way outbound for updates

2. **Security Groups**
   ```
   Internet → ALB-SG (HTTP:80/HTTPS:443) → ECS-Tasks-SG (HTTP:80) → Tasks
   ```

3. **No Public IPs**
   - ECS tasks have no direct internet access
   - All inbound traffic through ALB
   - Outbound through NAT Gateway

### IAM Security

1. **Task Execution Role** (ecsTaskExecutionRole)
   - Pull images from ECR
   - Write logs to CloudWatch
   - Managed by AWS

2. **Task Role** (Custom)
   - Read/Write to S3 bucket
   - Access to other AWS services as needed
   - Principle of least privilege

### Best Practices Implemented
- ✅ Least privilege IAM policies
- ✅ Security groups with minimal ports
- ✅ Private subnet placement
- ✅ No hardcoded credentials
- ✅ Environment variables for configuration
- ✅ Regular security group audits

---

## 📈 Monitoring & Verification

### Health Checks

```bash
# Check ECS service status
aws ecs describe-services \
  --cluster my-fargate-cluster \
  --services My-Web-App-Final-service

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>

# View CloudWatch logs
aws logs tail /ecs/my-web-app-final --follow
```

### Verification Steps

1. **ALB Access Test**
   ```bash
   curl http://<alb-dns-name>
   ```

2. **Task Status**
   - Navigate to ECS Console → Clusters → my-fargate-cluster
   - Verify: Running tasks = Desired tasks

3. **Target Group Health**
   - EC2 Console → Target Groups → ecs-app-tg
   - Verify: All targets show "healthy" status

4. **Auto Scaling Activities**
   - ECS Console → Service → Auto Scaling tab
   - Monitor scaling activities in real-time

### Key Metrics to Monitor

| Metric | Normal Range | Alert Threshold |
|--------|--------------|-----------------|
| CPU Utilization | 30-70% | >85% |
| Memory Utilization | 40-75% | >90% |
| Target Response Time | <200ms | >1000ms |
| Healthy Target Count | 2 | <1 |
| 5XX Errors | 0 | >10/min |

---

## 💰 Cost Optimization

### Current Configuration Costs (Estimated)

| Resource | Quantity | Monthly Cost (USD) |
|----------|----------|-------------------|
| Fargate Tasks (2 running) | 2 × 0.5 vCPU, 1GB RAM | ~$20-30 |
| Application Load Balancer | 1 ALB | ~$16-23 |
| NAT Gateway | 1 NGW + data transfer | ~$35-45 |
| ECR Storage | 176.76 MB | <$1 |
| S3 Storage | Minimal | <$1 |
| **Total** | | **~$72-100/month** |

### Cost Optimization Strategies

1. **Scheduled Scaling**
   - Reduces to 2 tasks during off-peak hours
   - Potential savings: 30-40%

2. **Fargate Spot** (Future Enhancement)
   - Use Spot capacity for non-critical tasks
   - Potential savings: Up to 70%

3. **S3 Lifecycle Policies**
   - Move old data to cheaper storage classes
   - Potential savings: 50-70% on storage

4. **Reserved Capacity** (Long-term)
   - Commit to 1-year or 3-year terms
   - Potential savings: 30-50%

---

## 🐛 Troubleshooting

### Common Issues

#### Tasks Not Starting

**Symptoms**: Tasks stuck in PENDING or immediately STOPPED

**Solutions**:
```bash
# Check task definition
aws ecs describe-task-definition --task-definition My-Web-App-Final

# Check stopped task reasons
aws ecs describe-tasks --cluster my-fargate-cluster --tasks <task-id>

# Common fixes:
# 1. Verify ECR image URI is correct
# 2. Check IAM execution role permissions
# 3. Ensure sufficient vCPU/memory allocated
# 4. Verify environment variables are set
```

#### Unhealthy Targets

**Symptoms**: Targets showing "unhealthy" in target group

**Solutions**:
```bash
# Check health check configuration
aws elbv2 describe-target-health --target-group-arn <tg-arn>

# Verify application is responding on correct port
docker run -p 80:80 cloudfolks-php-app:latest
curl http://localhost

# Common fixes:
# 1. Verify container port matches target group port
# 2. Check security group allows ALB → Tasks traffic
# 3. Ensure health check path returns 200 OK
# 4. Adjust health check interval/timeout
```

#### Auto Scaling Not Working

**Symptoms**: Tasks not scaling according to schedule

**Solutions**:
```bash
# Check scheduled actions
aws application-autoscaling describe-scheduled-actions \
  --service-namespace ecs

# Verify scalable target
aws application-autoscaling describe-scalable-targets \
  --service-namespace ecs

# Common fixes:
# 1. Verify schedule is in UTC timezone
# 2. Check IAM permissions for auto scaling
# 3. Ensure min/max capacity is correct
# 4. Wait for schedule to trigger
```

#### Connection Timeout

**Symptoms**: Cannot access application via ALB DNS

**Solutions**:
```bash
# Verify ALB is active
aws elbv2 describe-load-balancers --names ecs-app-alb

# Check security group rules
aws ec2 describe-security-groups --group-ids <alb-sg-id>

# Test ALB listener
curl -I http://<alb-dns-name>

# Common fixes:
# 1. Verify ALB security group allows inbound HTTP
# 2. Check route table has IGW route for public subnets
# 3. Ensure ALB is in "active" state
# 4. Verify listener is configured correctly
```

---

## 🎯 Best Practices

### Development Best Practices
- ✅ Use infrastructure as code (CloudFormation/Terraform)
- ✅ Version control for task definitions
- ✅ Automated CI/CD pipelines
- ✅ Blue/green deployments for zero downtime
- ✅ Container image scanning for vulnerabilities

### Operational Best Practices
- ✅ CloudWatch alarms for critical metrics
- ✅ Centralized logging (CloudWatch Logs)
- ✅ Regular backup of S3 data
- ✅ Document incident response procedures
- ✅ Automated rollback procedures

### Security Best Practices
- ✅ Rotate IAM credentials regularly
- ✅ Enable AWS CloudTrail for audit logs
- ✅ Use AWS Secrets Manager for sensitive data
- ✅ Implement WAF rules on ALB
- ✅ Regular security assessments

---

## 📸 Screenshots

### ECS Cluster Overview
![ECS Cluster](docs/screenshots/ecs-cluster.png)
- Cluster status: Active
- Services: 1 active
- Running tasks: 2/2

### ECR Repository
![ECR Repository](docs/screenshots/ecr-repository.png)
- Repository: cloudfolks-php-app
- Image size: 176.76 MB
- Tag: latest

### VPC Configuration
![VPC Configuration](docs/screenshots/vpc-config.png)
- VPC CIDR: 192.168.0.0/24
- Subnets: 4 (2 public, 2 private)
- Network connections: IGW, NAT

### Application Load Balancer
![ALB](docs/screenshots/alb.png)
- Status: Active
- Target group: 2 healthy targets
- Listener: HTTP:80

### Auto Scaling Configuration
![Auto Scaling](docs/screenshots/auto-scaling.png)
- Task count: 2-8
- Scheduled actions: 2 active policies

---

## 📚 Additional Resources

### Documentation
- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [Fargate Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/intro.html)
- [Docker Documentation](https://docs.docker.com/)

### Related Projects
- [Task Definition Template](./docs/task-definition.json)
- [Deployment Scripts](./scripts/)
- [Architecture Diagrams](./architecture/)

### Learning Resources
- AWS ECS Workshop
- Container Security Best Practices
- Microservices Architecture Patterns

---

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

### How to Contribute
1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 👤 Author

**Shankar Suthar**

- LinkedIn: www.linkedin.com/in/shankarsuthar-
- GitHub: https://github.com/Shankarr009
- Email: shankarsuthar499@gmail.com

---

## 🙏 Acknowledgments

- AWS Documentation Team for comprehensive guides
- CloudFolks Community for project inspiration
- Open source community for tools and resources

---

## 📊 Project Status

- ✅ Initial deployment complete
- ✅ Auto scaling configured
- ✅ Production ready
- 🔄 Monitoring setup in progress
- 📋 Documentation updates ongoing

---

**⭐ If you find this project helpful, please consider giving it a star!**

**📧 Questions? Open an issue or reach out directly!**

---

*Last Updated: January 31, 2026*
