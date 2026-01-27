# Architecture & Design Document

## Overview
This document outlines the cloud-native architecture designed for migrating Navoy's JavaScript application from a single VM to a scalable, secure AWS infrastructure.

## Proposed AWS Architecture

### High-Level Architecture
```
Internet
    ↓
Application Load Balancer (Public Subnets)
    ↓
ECS Fargate Cluster (Private Subnets)
    ↓
Docker Containers (Auto-scaling)
```

### Core Components

**1. Networking Layer**
- **VPC**: Isolated network environment (CIDR: 10.0.0.0/16)
- **Public Subnets** (2 AZs): Host ALB, NAT Gateway
- **Private Subnets** (2 AZs): Host ECS tasks (more secure)
- **Internet Gateway**: Allows public subnet internet access
- **NAT Gateway**: Allows private subnets outbound internet access

**2. Compute Layer**
- **ECS Fargate**: Serverless container orchestration
- **Task Definition**: Defines our containerized application
- **Service**: Manages desired task count and auto-scaling
- **ECR**: Container image registry

**3. Load Balancing**
- **Application Load Balancer**: Distributes traffic across containers
- **Target Group**: Routes to healthy ECS tasks
- **Health Checks**: `/health` endpoint monitoring

**4. Security**
- **Security Groups**: Firewall rules (ALB, ECS)
- **IAM Roles**: Least privilege access for ECS tasks
- **Private Subnets**: Application runs in isolated network

---

## Scalability

### Auto-Scaling Strategy
- **Target Tracking Scaling**: Scale based on CPU/Memory utilization
- **Min Tasks**: 2 (for high availability)
- **Max Tasks**: 10 (cost control)
- **Scale-out threshold**: >70% CPU
- **Scale-in threshold**: <30% CPU

### Load Balancing
- ALB distributes traffic across multiple AZs
- Health checks ensure only healthy tasks receive traffic
- Connection draining during scale-in events

---

## Reliability & Security

### Reliability
- **Multi-AZ Deployment**: Resources span 2 availability zones
- **Health Checks**: Automatic unhealthy task replacement
- **Rolling Updates**: Zero-downtime deployments
- **Auto-Recovery**: ECS automatically replaces failed tasks

### Security
- **Network Isolation**: Application in private subnets
- **Security Groups**: Restrictive ingress/egress rules
  - ALB: Only ports 80/443 from internet
  - ECS: Only port 3000 from ALB
- **IAM Roles**: Least privilege access
- **Secrets Management**: AWS Secrets Manager (production)
- **No Hardcoded Credentials**: All via IAM roles

---

## CI/CD Strategy

### Pipeline Overview
```
Code Push → GitHub Actions → Build → Test → Docker Build → Push to ECR → Deploy to ECS
```

### GitHub Actions Workflow
1. **Trigger**: On push to `main` branch
2. **Build**: Install dependencies, run tests
3. **Containerize**: Build Docker image
4. **Push**: Upload to ECR
5. **Deploy**: Update ECS service with new image

### Deployment Strategy
- **Rolling updates**: Gradual task replacement
- **Health checks**: Ensure new tasks healthy before continuing
- **Rollback**: Automated if health checks fail

---

## Key Trade-offs & Assumptions

### Decisions Made

**1. ECS Fargate vs. EC2**
- ✅ **Chose Fargate**: Serverless, no server management, easier scaling
- ❌ **Trade-off**: Slightly higher cost vs EC2, less control
- **Why**: Simpler operations, faster to market, good for MVP

**2. ECS vs. EKS (Kubernetes)**
- ✅ **Chose ECS**: Simpler, AWS-native, easier for small teams
- ❌ **Trade-off**: Less portable, smaller ecosystem than K8s
- **Why**: Assignment specifies EKS as "design-only", ECS sufficient for needs

**3. Application Load Balancer vs. Network Load Balancer**
- ✅ **Chose ALB**: HTTP/HTTPS routing, health checks, better for web apps
- **Why**: Our app is HTTP-based, need path-based routing

**4. Multi-AZ vs. Single AZ**
- ✅ **Chose Multi-AZ**: Better availability
- ❌ **Trade-off**: Higher cost (NAT Gateway x2, data transfer)
- **Why**: Production-ready design, high availability requirement

### Assumptions

1. **Application is stateless**: No persistent local storage needed
2. **Traffic patterns**: Moderate, predictable load (not extreme spikes)
3. **Database**: Not included in MVP (can add RDS/DynamoDB later)
4. **Region**: Single region deployment (us-east-1)
5. **SSL/TLS**: Not implemented in LocalStack version (would use ACM in production)

---

## What Would I Improve With More Time?

### Short-term (1-2 weeks)
- Add **RDS database** (PostgreSQL/MySQL)
- Implement **CloudWatch dashboards** for monitoring
- Add **SSL/TLS** with ACM certificates
- Implement **WAF** for security
- Add **CloudFront CDN** for caching

### Medium-term (1-2 months)
- **Multi-region deployment** for disaster recovery
- **ElastiCache** (Redis) for session management
- **Secrets rotation** automation
- **Cost optimization**: Spot instances, reserved capacity
- **Advanced monitoring**: Distributed tracing (X-Ray)

### Long-term (3-6 months)
- **Kubernetes (EKS)** migration for complex microservices
- **Service mesh** (App Mesh) for advanced traffic management
- **Infrastructure testing** (Terratest)
- **GitOps** workflow (ArgoCD/Flux)

---

## Architecture Diagram
```
                                    ┌─────────────────┐
                                    │   Internet      │
                                    └────────┬────────┘
                                             │
                                    ┌────────▼────────┐
                                    │  Internet GW    │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                   VPC (10.0.0.0/16)             │
                    │                                                 │
        ┌───────────▼──────────┐                   ┌─────────────────▼──┐
        │  Public Subnet (AZ1) │                   │ Public Subnet (AZ2)│
        │     10.0.1.0/24      │                   │    10.0.2.0/24     │
        └───────────┬──────────┘                   └─────────────────┬──┘
                    │                                                │
                    └──────────────┬─────────────────────────────────┘
                                   │
                          ┌────────▼────────┐
                          │  Load Balancer  │
                          └────────┬────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                                                      │
┌───────▼──────────┐                              ┌───────────▼─────────┐
│ Private Subnet   │                              │  Private Subnet     │
│   (AZ1)          │                              │    (AZ2)            │
│  10.0.11.0/24    │                              │   10.0.12.0/24      │
└───────┬──────────┘                              └───────────┬─────────┘
        │                                                     │
   ┌────▼────┐  ┌─────────┐                        ┌─────────┐  ┌────────┐
   │ECS Task │  │ECS Task │                        │ECS Task │  │ECS Task│
   └─────────┘  └─────────┘                        └─────────┘  └────────┘
```

---

## Conclusion

This architecture provides a solid foundation for a scalable, reliable, and secure cloud-native application. The design balances simplicity with production-readiness, making it suitable for immediate deployment while allowing for future enhancements.

The use of containers, infrastructure as code, and automated CI/CD ensures the system is maintainable and can evolve with business needs.