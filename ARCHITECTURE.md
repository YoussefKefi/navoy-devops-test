# Architecture & Design Document

## Overview

This document explains how I designed a cloud-native architecture to migrate Navoy's JavaScript application from running on a single virtual machine to a scalable, secure AWS infrastructure.

**Current Problem:**
The application currently runs on a single VM, which has several issues:
- Can't handle traffic spikes (scalability problem)
- If the server goes down, the whole app is offline
- No security isolation
- Not following modern cloud best practices

**My Solution:**
Migrate to AWS using containers, load balancing, and infrastructure-as-code to create a system that can scale automatically and stay online even if individual components fail.

---

## Proposed AWS Architecture

### High-Level Flow
```
Internet
    ↓
Application Load Balancer (Public Subnets)
    ↓
ECS Fargate Cluster (Private Subnets)
    ↓
Docker Containers (Auto-scaling)
```

### Main Components

**VPC (Virtual Private Cloud)**

I created a VPC, which is basically our own isolated network space in AWS (like renting a private office building). Everything we build sits inside this VPC with the IP range 10.0.0.0/16, separated from other users' infrastructure.

**Subnets**

Inside the VPC, I set up two types of subnets across two availability zones (different physical locations):

- **Public Subnets (2)**: These can be accessed from the internet. I placed the load balancer here so users can reach our application.
- **Private Subnets (2)**: These are NOT directly accessible from the internet. Our application containers run here for better security.

**Internet Gateway & NAT Gateway**

- **Internet Gateway**: Allows the public subnets to communicate with the internet
- **NAT Gateway**: Allows containers in private subnets to reach the internet for things like downloading updates (but internet users can't reach them directly)

**Application Load Balancer (ALB)**

The load balancer sits in the public subnets and acts like a traffic controller. When users access our app, the load balancer distributes their requests evenly across all running containers. This prevents any single container from getting overloaded and provides a single entry point for all traffic.

**ECS Fargate**

ECS (Elastic Container Service) is AWS's service for running Docker containers. I chose Fargate, which is the "serverless" option - meaning AWS manages the underlying servers for us. ECS handles:
- Running our Docker containers in the private subnets
- Monitoring container health
- Automatically restarting failed containers  
- Scaling the number of containers based on demand

**Security Groups**

Security groups are firewall rules that control what traffic is allowed:
- **ALB Security Group**: Accepts HTTP traffic (port 80) from anyone on the internet
- **ECS Security Group**: Only accepts traffic from the load balancer on port 3000 (our app's port)

This layered approach means users can't directly access our containers - they must go through the load balancer.

**IAM Roles**

I set up IAM roles (AWS's permission system) so our containers can:
- Pull Docker images from the registry
- Write logs to CloudWatch
- Access other AWS services they need

This follows the "least privilege" principle - giving only the minimum permissions needed.

---

## How Scalability Works

**Auto-Scaling**

I configured ECS to automatically add or remove containers based on CPU and memory usage:
- **Minimum**: 2 containers (for reliability - if one fails, we still have one running)
- **Maximum**: 10 containers (to control costs)
- **Scale up**: When CPU goes above 70%
- **Scale down**: When CPU < 70%, remove containers (with 5-minute cooldown to prevent rapid scaling)

**Load Balancing**

The Application Load Balancer distributes incoming traffic across all available containers. It also performs health checks on the `/health` endpoint every 30 seconds. If a container is unhealthy, the load balancer stops sending traffic to it until it recovers.

**Multi-AZ Deployment**

By spreading resources across two availability zones (us-east-1a and us-east-1b), the system stays online even if one entire data center goes down.

---

## Security & Reliability

### Security Approach

**Network Isolation**
- Application containers run in private subnets with no direct internet access
- Only the load balancer is publicly accessible
- Traffic must flow through the load balancer to reach containers

**Firewall Rules (Security Groups)**
- Restrictive rules: Only allow necessary traffic
- ALB: Accepts HTTP/HTTPS from anywhere
- ECS: Only accepts traffic from ALB on port 3000

**No Hardcoded Secrets**
- No passwords or API keys in code
- Would use AWS Secrets Manager for sensitive data in production
- IAM roles provide permissions instead of storing credentials

### Reliability Features

**Health Checks**
- Load balancer checks container health every 30 seconds
- Unhealthy containers are automatically replaced
- Traffic only goes to healthy containers

**Multi-AZ Deployment**
- Resources spread across 2 availability zones
- If one zone fails, the other keeps running

**Rolling Updates**
- When deploying new code, containers update gradually
- Old containers keep running until new ones are healthy
- Zero downtime during deployments

---

## CI/CD Strategy

### The Deployment Pipeline

I set up a GitHub Actions workflow that automates the deployment process:
```
1. Code Push (to main branch)
   ↓
2. Run Tests
   ↓
3. Build Docker Image
   ↓
4. Push Image to Registry (ECR)
   ↓
5. Update ECS Service
   ↓
6. Rolling Deployment
```

**How It Works:**

1. **Trigger**: When code is pushed to the main branch
2. **Build & Test**: Installs dependencies and runs tests
3. **Containerize**: Builds a Docker image with the new code
4. **Push**: Uploads the image to AWS ECR (container registry)
5. **Deploy**: Updates the ECS service to use the new image
6. **Health Checks**: ECS gradually replaces old containers, checking that new ones are healthy before continuing

**Rollback Strategy:**
If the new deployment fails health checks, ECS automatically stops the deployment and keeps the old version running.

---

## Key Decisions & Trade-offs

### Why ECS Fargate Instead of EC2?

**Decision**: Use Fargate (serverless containers)

**Pros:**
- No need to manage servers
- Easier to scale
- Only pay for containers running, not idle servers

**Cons:**
- Slightly more expensive than managing your own EC2 servers
- Less control over the underlying infrastructure

**Why I chose it**: For this project, the simplicity and ease of management outweigh the extra cost. It's also more modern and follows AWS best practices.

### Why ECS Instead of Kubernetes (EKS)?

**Decision**: Use ECS

**Pros:**
- Simpler to set up and manage
- Better integration with AWS services
- Easier for small teams

**Cons:**
- Less portable (locked into AWS)
- Smaller community than Kubernetes

**Why I chose it**: The assignment mentioned EKS as "design-only", and for this scale of application, ECS is sufficient and simpler.

### Why Multi-AZ?

**Decision**: Deploy across 2 availability zones

**Pros:**
- High availability - survives data center failure
- Production-ready architecture

**Cons:**
- Higher cost (need resources in both zones, 2 NAT gateways)

**Why I chose it**: Reliability is important, and this is the standard approach for production applications.

---

## Assumptions I Made

1. **The application is stateless**: It doesn't store data locally on the container. Any state would be stored in a database (not implemented in this MVP).

2. **Traffic is moderate**: Not expecting millions of requests per second. The 2-10 container range should handle normal loads.

3. **Single region is acceptable**: Deploying only in us-east-1. Multi-region would add complexity and cost.

4. **No database needed yet**: The application can run without a database for now. In production, I'd add RDS (managed database service).

5. **HTTP is fine for now**: Not implementing HTTPS/SSL in the LocalStack version, but would use AWS Certificate Manager for real deployment.

---

## What I Would Improve With More Time

### Short-term (Next 1-2 weeks)

- **Add a database**: Set up RDS (PostgreSQL or MySQL) for data persistence
- **Implement HTTPS**: Use AWS Certificate Manager for SSL certificates
- **Better monitoring**: Create CloudWatch dashboards to visualize metrics
- **WAF (Web Application Firewall)**: Add protection against common web attacks
- **CDN**: Add CloudFront for faster content delivery worldwide

### Medium-term (1-2 months)

- **Multi-region deployment**: Deploy to multiple AWS regions for disaster recovery
- **Caching**: Add ElastiCache (Redis) for faster response times
- **Better CI/CD**: Add automated integration tests, staging environment
- **Cost optimization**: Analyze usage patterns and optimize resource allocation
- **Advanced monitoring**: Implement distributed tracing with X-Ray

### Long-term (3-6 months)

- **Migrate to Kubernetes**: If the application grows into multiple microservices
- **Infrastructure testing**: Add automated tests for Terraform code (Terratest)
- **GitOps**: Implement continuous deployment with ArgoCD
- **Advanced security**: Implement automated security scanning, compliance checks

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

This architecture solves the original problem of running on a single VM by creating a scalable, reliable, and secure cloud-native system. The design uses modern tools (Docker, Terraform, CI/CD) and AWS best practices while keeping things simple enough to understand and maintain.

The infrastructure can automatically scale to handle more users, stays online even when components fail, and follows security best practices. While there's room for improvement (database, HTTPS, better monitoring), this provides a solid foundation that can grow with the application's needs.