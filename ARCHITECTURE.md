# Architecture Overview

This document provides a detailed explanation of the system architecture for the ECS Fargate web application deployment.

## Table of Contents
- [High-Level Architecture](#high-level-architecture)
- [Network Architecture](#network-architecture)
- [Component Details](#component-details)
- [Data Flow](#data-flow)
- [Security Architecture](#security-architecture)
- [Scalability Architecture](#scalability-architecture)

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           AWS Cloud                                  │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    VPC (192.168.0.0/24)                        │ │
│  │                                                                 │ │
│  │  ┌──────────────────────────────────────────────────────────┐ │ │
│  │  │                   Internet Gateway                        │ │ │
│  │  └────────────────────────┬─────────────────────────────────┘ │ │
│  │                           │                                    │ │
│  │  ┌────────────────────────▼─────────────────────────────────┐ │ │
│  │  │             Application Load Balancer                     │ │ │
│  │  │              (ecs-app-alb)                                │ │ │
│  │  │  ┌──────────────────┐   ┌──────────────────┐            │ │ │
│  │  │  │  Public Subnet   │   │  Public Subnet   │            │ │ │
│  │  │  │   eu-west-1a     │   │   eu-west-1b     │            │ │ │
│  │  │  │  192.168.0.0/26  │   │  192.168.0.64/26 │            │ │ │
│  │  │  └──────────────────┘   └──────────────────┘            │ │ │
│  │  └────────────────────────┬─────────────────────────────────┘ │ │
│  │                           │                                    │ │
│  │                  ┌────────┴─────────┐                         │ │
│  │                  │  Target Group    │                         │ │
│  │                  │   (ecs-app-tg)   │                         │ │
│  │                  └────────┬─────────┘                         │ │
│  │                           │                                    │ │
│  │           ┌───────────────┴───────────────┐                  │ │
│  │           │                               │                  │ │
│  │  ┌────────▼────────┐           ┌─────────▼──────┐           │ │
│  │  │  Private Subnet │           │ Private Subnet │           │ │
│  │  │   eu-west-1a    │           │   eu-west-1b   │           │ │
│  │  │ 192.168.0.128/26│           │192.168.0.192/26│           │ │
│  │  │                 │           │                │           │ │
│  │  │ ┌─────────────┐ │           │ ┌────────────┐ │           │ │
│  │  │ │ ECS Task 1  │ │           │ │ ECS Task 2 │ │           │ │
│  │  │ │  (Fargate)  │ │           │ │  (Fargate) │ │           │ │
│  │  │ │             │ │           │ │            │ │           │ │
│  │  │ │ Container:  │ │           │ │ Container: │ │           │ │
│  │  │ │ php-app:80  │ │           │ │ php-app:80 │ │           │ │
│  │  │ └─────────────┘ │           │ └────────────┘ │           │ │
│  │  └─────────┬───────┘           └────────┬───────┘           │ │
│  │            └─────────────┬───────────────┘                   │ │
│  │                          │                                    │ │
│  │                  ┌───────▼────────┐                          │ │
│  │                  │  NAT Gateway   │                          │ │
│  │                  │   (ECS-NAT)    │                          │ │
│  │                  └───────┬────────┘                          │ │
│  │                          │                                    │ │
│  └──────────────────────────┼────────────────────────────────────┘ │
│                             │                                      │
│     ┌───────────────────────┴─────────────────┐                  │
│     │                                         │                  │
│  ┌──▼─────────────┐                  ┌───────▼─────┐            │
│  │ Amazon ECR     │                  │ Amazon S3   │            │
│  │ cloudfolks-    │                  │ my-web-app- │            │
│  │ php-app        │                  │ data-09     │            │
│  └────────────────┘                  └─────────────┘            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Network Architecture

### VPC Design

**CIDR Block**: `192.168.0.0/24` (256 IP addresses)

#### Subnet Breakdown

| Subnet Type | AZ | CIDR | Usable IPs | Purpose |
|-------------|----|-----------|----|---------|
| Public Subnet 1 | eu-west-1a | 192.168.0.0/26 | 59 | ALB, NAT Gateway |
| Public Subnet 2 | eu-west-1b | 192.168.0.64/26 | 59 | ALB |
| Private Subnet 1 | eu-west-1a | 192.168.0.128/26 | 59 | ECS Tasks |
| Private Subnet 2 | eu-west-1b | 192.168.0.192/26 | 59 | ECS Tasks |

### Routing Architecture

#### Public Route Table (RT-for-Public)
```
Destination         Target
0.0.0.0/0    →     Internet Gateway (IGW)
192.168.0.0/24 →   Local
```

**Associated Subnets**: 
- Public Subnet 1 (192.168.0.0/26)
- Public Subnet 2 (192.168.0.64/26)

#### Private Route Table (RT-for-Private)
```
Destination         Target
0.0.0.0/0    →     NAT Gateway (NGW)
192.168.0.0/24 →   Local
```

**Associated Subnets**:
- Private Subnet 1 (192.168.0.128/26)
- Private Subnet 2 (192.168.0.192/26)

### Gateway Architecture

#### Internet Gateway
- **Purpose**: Provides internet access to public subnets
- **Attachment**: Attached to VPC
- **Usage**: ALB receives inbound traffic from internet

#### NAT Gateway
- **Location**: Public Subnet 1
- **Elastic IP**: Required for stable outbound IP
- **Purpose**: Allows private subnets to access internet for:
  - Pulling images from ECR
  - Accessing S3 buckets
  - Software updates
  - API calls to AWS services

---

## Component Details

### 1. Application Load Balancer (ALB)

**Configuration**:
- **Type**: Application Load Balancer
- **Scheme**: Internet-facing
- **Subnets**: Both public subnets (multi-AZ)
- **Security Group**: ALB-SG
- **Listener**: HTTP:80

**Functionality**:
- Distributes incoming traffic across ECS tasks
- Performs health checks on targets
- Maintains session stickiness (if enabled)
- Provides SSL termination (when configured)

**Target Group Settings**:
- **Target Type**: IP (required for Fargate)
- **Protocol**: HTTP
- **Port**: 80
- **Health Check Path**: /
- **Health Check Interval**: 30 seconds
- **Healthy Threshold**: 2
- **Unhealthy Threshold**: 3

### 2. ECS Fargate Cluster

**Cluster Configuration**:
- **Name**: my-fargate-cluster
- **Launch Type**: Fargate
- **Capacity Providers**: FARGATE, FARGATE_SPOT

**Service Configuration**:
- **Name**: My-Web-App-Final-service
- **Desired Count**: 2 tasks
- **Min Capacity**: 2 tasks
- **Max Capacity**: 8 tasks
- **Deployment Type**: Rolling update

**Task Definition**:
- **Family**: My-Web-App-Final
- **CPU**: 512 (0.5 vCPU)
- **Memory**: 1024 MB (1 GB)
- **Network Mode**: awsvpc

**Container Specification**:
- **Name**: cloudfolks-php-app
- **Image**: ECR URI (176.76 MB)
- **Port**: 80
- **Environment Variables**:
  - AWS_REGION: eu-west-1
  - S3_BUCKET: my-web-app-data-09

### 3. Amazon ECR (Elastic Container Registry)

**Repository**:
- **Name**: cloudfolks-php-app
- **Type**: Private
- **Image Scanning**: Enabled on push
- **Image Size**: 176.76 MB
- **Tag**: latest

**Access**:
- Task execution role has permissions to pull images
- Images are versioned by tags
- Automatic image scanning for vulnerabilities

### 4. Amazon S3

**Bucket Configuration**:
- **Name**: my-web-app-data-09
- **Region**: eu-west-1
- **Structure**: /uploads folder
- **Access**: Via IAM task role

**Usage**:
- Application data storage
- User-uploaded content
- Static assets (if applicable)

### 5. Security Groups

#### ALB Security Group (ALB-SG)

**Inbound Rules**:
```
Type        Protocol    Port    Source          Description
HTTP        TCP         80      0.0.0.0/0       Allow HTTP from internet
HTTPS       TCP         443     0.0.0.0/0       Allow HTTPS from internet (if configured)
```

**Outbound Rules**:
```
Type        Protocol    Port    Destination     Description
All Traffic All         All     0.0.0.0/0       Allow all outbound
```

#### ECS Tasks Security Group (ECS-Tasks-SG)

**Inbound Rules**:
```
Type        Protocol    Port    Source          Description
HTTP        TCP         80      ALB-SG          Allow traffic from ALB only
```

**Outbound Rules**:
```
Type        Protocol    Port    Destination     Description
All Traffic All         All     0.0.0.0/0       Allow outbound for ECR/S3 access
```

---

## Data Flow

### 1. User Request Flow

```
User Browser
    │
    │ HTTP Request
    ▼
Internet Gateway
    │
    ▼
Application Load Balancer
    │
    │ (Health Check: Is target healthy?)
    │ (Load Balance: Which task to route to?)
    ▼
Target Group
    │
    ├──→ ECS Task 1 (Private Subnet 1)
    │    └─→ Container responds
    │
    └──→ ECS Task 2 (Private Subnet 2)
         └─→ Container responds
    ▲
    │
    │ Response back through same path
    ▼
User Browser receives response
```

### 2. Container Image Pull Flow

```
ECS Service initiates task
    │
    ▼
Task Execution Role authenticates
    │
    ▼
Task pulls image from ECR
    │
    ├─→ Private Subnet → NAT Gateway
    │                      │
    │                      ▼
    │               Internet Gateway
    │                      │
    │                      ▼
    └──────────────→ Amazon ECR
```

### 3. S3 Access Flow

```
Application needs to read/write data
    │
    ▼
Task Role provides permissions
    │
    ▼
Container makes S3 API call
    │
    ├─→ Private Subnet → NAT Gateway
    │                      │
    │                      ▼
    │               Internet Gateway
    │                      │
    │                      ▼
    └──────────────→ Amazon S3
```

### 4. Auto Scaling Flow

```
Scheduled Time Reached
    │
    ▼
CloudWatch Events triggers
    │
    ▼
Application Auto Scaling evaluates
    │
    ├─→ Scale Out: Increase desired count
    │   │
    │   └─→ ECS launches new tasks
    │       │
    │       └─→ Tasks register with Target Group
    │           │
    │           └─→ ALB starts routing to new tasks
    │
    └─→ Scale In: Decrease desired count
        │
        └─→ ECS marks tasks for termination
            │
            └─→ ALB drains connections
                │
                └─→ Tasks terminated gracefully
```

---

## Security Architecture

### Multi-Layer Security Model

```
Layer 1: Network Isolation
    ├─→ VPC provides isolated network
    ├─→ Private subnets for sensitive workloads
    └─→ Public subnets only for load balancer

Layer 2: Security Groups
    ├─→ ALB Security Group (internet → ALB)
    ├─→ ECS Security Group (ALB → tasks only)
    └─→ Stateful firewall at instance level

Layer 3: IAM Roles
    ├─→ Task Execution Role (ECR, CloudWatch)
    ├─→ Task Role (S3, application permissions)
    └─→ Principle of least privilege

Layer 4: Resource Isolation
    ├─→ Fargate isolation (no shared infrastructure)
    ├─→ Network isolation (awsvpc mode)
    └─→ Container isolation

Layer 5: Data Protection
    ├─→ Encryption in transit (HTTPS when enabled)
    ├─→ Encryption at rest (S3, EBS)
    └─→ Private ECR repository
```

### Security Best Practices Implemented

1. **No Public IPs on Tasks**
   - All ECS tasks in private subnets
   - No direct internet access
   - All traffic through ALB

2. **Minimal Security Group Rules**
   - ALB: Only necessary ports (80, 443)
   - ECS: Only from ALB security group
   - No wide-open rules (0.0.0.0/0 on tasks)

3. **IAM Best Practices**
   - Separate execution and task roles
   - Least privilege permissions
   - No long-term credentials in containers

4. **Network Segmentation**
   - Clear separation of public/private
   - Controlled ingress/egress
   - Defense in depth

---

## Scalability Architecture

### Horizontal Scaling

```
Current State: 2 tasks
    │
    │ (Traffic increases OR Scheduled time)
    ▼
Auto Scaling triggers scale-out
    │
    ├─→ New tasks launched
    │   ├─→ Task 3 in AZ-1a
    │   └─→ Task 4 in AZ-1b
    │
    └─→ Tasks register with ALB
        │
        └─→ Load distributed across 4 tasks

Later...
    │
    │ (Traffic decreases OR Scheduled time)
    ▼
Auto Scaling triggers scale-in
    │
    ├─→ Tasks marked for termination
    │   ├─→ Task 3 draining
    │   └─→ Task 4 draining
    │
    └─→ Back to 2 tasks
```

### High Availability Design

```
Multi-AZ Deployment
    │
    ├─→ Availability Zone 1a
    │   ├─→ Public Subnet 1
    │   │   ├─→ ALB (50% traffic)
    │   │   └─→ NAT Gateway
    │   │
    │   └─→ Private Subnet 1
    │       └─→ ECS Task 1
    │
    └─→ Availability Zone 1b
        ├─→ Public Subnet 2
        │   └─→ ALB (50% traffic)
        │
        └─→ Private Subnet 2
            └─→ ECS Task 2
```

**Benefits**:
- Survives AZ failure
- ALB automatically fails over
- No single point of failure
- 99.99% availability SLA

### Scaling Policies

**Scheduled Scaling**:
```
Business Hours (08:00 - 18:00 Mon-Fri)
    Min: 4 tasks
    Max: 10 tasks
    ↓
Off-Peak Hours (18:00 - 08:00, Weekends)
    Min: 2 tasks
    Max: 2 tasks
```

**Target Tracking Scaling** (Optional Future Enhancement):
```
Metric: Average CPU > 70%
    → Scale out: Add 1 task
    
Metric: Average CPU < 30%
    → Scale in: Remove 1 task
```

---

## Performance Considerations

### Resource Allocation

| Component | Configuration | Performance Impact |
|-----------|--------------|-------------------|
| Task CPU | 512 (0.5 vCPU) | Handles ~50 concurrent requests |
| Task Memory | 1024 MB (1 GB) | Supports moderate memory workloads |
| ALB | Standard | ~100k requests/hour per task |

### Optimization Strategies

1. **Task Placement**
   - Spread across AZs for availability
   - Even distribution for load balancing

2. **Health Check Tuning**
   - 30s interval (balance between cost and detection)
   - 3 retries before marking unhealthy

3. **Connection Draining**
   - 300s deregistration delay
   - Graceful task termination

---

## Cost Architecture

### Monthly Cost Breakdown (Estimated)

```
Fargate Tasks (2 tasks @ 0.5 vCPU, 1GB)
    Base cost: $20-30/month
    │
    ├─→ CPU: $0.04048/vCPU/hour × 0.5 × 730 hours = ~$15
    └─→ Memory: $0.004445/GB/hour × 1 × 730 hours = ~$3

Application Load Balancer
    Base: $16-23/month
    │
    ├─→ Load Balancer hours: $0.0225/hour × 730 = ~$16
    └─→ LCU charges: Variable based on traffic

NAT Gateway
    Base: $35-45/month
    │
    ├─→ NAT Gateway hours: $0.045/hour × 730 = ~$33
    └─→ Data processing: $0.045/GB × usage

Total: ~$72-100/month
```

**Cost Optimization Opportunities**:
- Fargate Spot: Save up to 70%
- Scheduled scaling: Save 30-40%
- Reserved capacity: Save 30-50% (long-term)

---

## Monitoring Architecture

### CloudWatch Integration

```
ECS Tasks
    ├─→ Container Logs → CloudWatch Logs
    │   └─→ /ecs/my-web-app-final
    │
    ├─→ Task Metrics → CloudWatch Metrics
    │   ├─→ CPU Utilization
    │   ├─→ Memory Utilization
    │   └─→ Network I/O
    │
    └─→ Service Events → CloudWatch Events
        ├─→ Task state changes
        ├─→ Service updates
        └─→ Auto scaling activities

ALB
    ├─→ Target Health
    ├─→ Request Count
    ├─→ Response Time
    └─→ Error Rates (4xx, 5xx)
```

---

## Disaster Recovery

### Backup Strategy
- S3 bucket versioning enabled
- Task definitions versioned
- Infrastructure as code (reproducible)

### Recovery Procedures

**Scenario 1: Task Failure**
- Automatic: ECS launches replacement
- Time to recovery: ~60 seconds

**Scenario 2: AZ Failure**
- Automatic: ALB routes to healthy AZ
- Time to recovery: Immediate

**Scenario 3: Complete Service Failure**
- Manual: Restore from infrastructure code
- Time to recovery: ~15-20 minutes

---

## Future Enhancements

### Planned Improvements

1. **SSL/TLS Termination**
   - Add ACM certificate to ALB
   - Redirect HTTP to HTTPS

2. **CDN Integration**
   - CloudFront distribution
   - Edge caching for static content

3. **Database Integration**
   - RDS or Aurora in private subnets
   - Automatic backups

4. **CI/CD Pipeline**
   - CodePipeline for automated deployments
   - Blue/green deployment strategy

5. **Enhanced Monitoring**
   - CloudWatch dashboards
   - Custom metrics
   - Automated alarms

6. **WAF Integration**
   - Web Application Firewall rules
   - DDoS protection
   - Rate limiting

---

*This architecture is designed for production use with high availability, security, and scalability in mind.*
