# Navoy DevOps Technical Assessment

## Project Overview

This repository contains a complete cloud-native infrastructure solution for migrating a JavaScript
application from a single VM to a scalable, secure AWS architecture. The project demonstrates modern DevOps
practices including Infrastructure as Code (Terraform), CI/CD (GitHub Actions), and containerization (Docker).

---

## 📁 Repository Structure

navoy-devops-test/
├── README.md                    # This file - setup instructions
├── ARCHITECTURE.md              # Architecture design and decisions
├── DEVOPS_REASONING.md          # DevOps best practices Q&A
├── app/                         # JavaScript application
│   ├── server.js                # Node.js Express application
│   ├── package.json             # Node dependencies
│   ├── Dockerfile               # Container definition
│   └── .dockerignore            # Docker ignore rules
├── terraform/                   # Infrastructure as Code
│   ├── main.tf                  # Main configuration
│   ├── provider.tf              # AWS provider setup
│   ├── variables.tf             # Input variables
│   ├── outputs.tf               # Output values
│   ├── vpc.tf                   # VPC and networking
│   ├── security_groups.tf       # Security group rules
│   ├── alb.tf                   # Application Load Balancer
│   └── ecs.tf                   # ECS cluster and services
└── .github/
    └── workflows/
        └── deploy.yml           # CI/CD pipeline
```

---

## 🎯 What This Project Demonstrates

- ✅ **Infrastructure as Code**: Complete AWS infrastructure defined in Terraform
- ✅ **Cloud-Native Architecture**: Containerized application on ECS Fargate
- ✅ **Scalability**: Auto-scaling, load balancing, multi-AZ deployment
- ✅ **Security**: VPC isolation, security groups, IAM roles
- ✅ **CI/CD**: Automated build and deployment pipeline
- ✅ **Best Practices**: Modular code, documentation, version control

---

## 🛠️ Prerequisites

Before running this project, ensure you have:

### Required Tools

1. **Docker Desktop**
   - Download: https://www.docker.com/products/docker-desktop/
   - Version: 20.10 or higher
   - Must be running before starting LocalStack

2. **Terraform**
   - Download: https://www.terraform.io/downloads
   - Version: 1.0 or higher
   - Add to PATH or note the executable location

3. **LocalStack** (for local testing)
   - Download: https://docs.localstack.cloud/getting-started/installation/
   - Or use Docker: `docker run -d --name localstack -p 4566:4566 localstack/localstack`

4. **Node.js** (for application development)
   - Download: https://nodejs.org/
   - Version: 18 LTS or higher

5. **Git**
   - Download: https://git-scm.com/downloads
   - For version control and GitHub integration

### Optional Tools

- **AWS CLI** (if deploying to real AWS)
- **curl** or **Postman** (for API testing)

---

## 🚀 Quick Start Guide

### Step 1: Clone the Repository
```bash
git clone https://github.com/YoussefKefi/navoy-devops-test
cd navoy-devops-test
```

### Step 2: Test the Application Locally
```bash
cd app
npm install
npm start
```

Visit `http://localhost:3000` - you should see a JSON response.

Press `Ctrl+C` to stop.

### Step 3: Build Docker Image
```bash
cd app
docker build -t navoy-demo-app:latest .
docker run -p 3000:3000 navoy-demo-app:latest
```

Visit `http://localhost:3000` again to verify the containerized app works.

### Step 4: Start LocalStack

**Option A: Using Docker**
```bash
docker run -d --name localstack -p 4566:4566 -p 4510-4559:4510-4559 localstack/localstack
```

**Option B: Using LocalStack CLI**
```bash
localstack start
```

Verify LocalStack is running:
```bash
curl http://localhost:4566/_localstack/health
```

You should see JSON output showing available services.

### Step 5: Deploy Infrastructure with Terraform
```bash
cd terraform
terraform init
terraform validate
terraform plan
terraform apply
```

Type `yes` when prompted.

---

## ⚠️ Known Limitations with LocalStack

**LocalStack Community Edition** (free version) has limited AWS service support:

### ✅ What Works:
- VPC and networking (subnets, route tables, internet gateway)
- Security Groups
- IAM roles and policies
- CloudWatch Log Groups

### ❌ What Doesn't Work (Requires LocalStack Pro):
- **ECS (Elastic Container Service)** - Not fully emulated
- **ELB/ALB (Load Balancers)** - Not included in free tier

### Expected Errors:

When running `terraform apply`, you'll see errors like:
```
Error: The API for service elbv2 is not included in your current license plan
Error: The API for service ecs is not included in your current license plan
```

**This is expected and acceptable!** The assignment requires demonstrating the infrastructure can be created, and the Terraform code is valid. The limitations are LocalStack's, not the code's.

---

## ☁️ Deploying to Real AWS (Optional)

To deploy to actual AWS:

### Step 1: Configure AWS Credentials
```bash
aws configure
```

Enter your AWS Access Key ID, Secret Access Key, and region.

### Step 2: Update Terraform Variables

In `terraform/variables.tf`, change:
```hcl
variable "use_localstack" {
  default = false  # Change from true to false
}
```

Or create a `terraform.tfvars` file:
```hcl
use_localstack = false
aws_region     = "us-east-1"
```

### Step 3: Deploy
```bash
cd terraform
terraform init
terraform plan
terraform apply
```

### Step 4: Access the Application

After apply completes, Terraform will output the ALB URL:
```
Outputs:
alb_url = "http://navoy-demo-alb-1234567890.us-east-1.elb.amazonaws.com"
```

Visit this URL to access your deployed application.

### Step 5: Clean Up (Important!)

To avoid AWS charges:
```bash
terraform destroy
```

Type `yes` when prompted.

---

## 🧪 Testing

### Test the Application
```bash
# Health check endpoint
curl http://localhost:3000/health

# Main endpoint
curl http://localhost:3000/

# Info endpoint
curl http://localhost:3000/info
```

### Test Docker Build
```bash
cd app
docker build -t navoy-demo-app:latest .
docker run -p 3000:3000 navoy-demo-app:latest
```

### Verify Terraform Configuration
```bash
cd terraform
terraform fmt      # Format code
terraform validate # Validate syntax
terraform plan     # Preview changes
```

---

## 📊 CI/CD Pipeline

The GitHub Actions workflow (`.github/workflows/deploy.yml`) automatically:

1. **Build & Test**: Installs dependencies and runs tests
2. **Build Docker Image**: Creates container image
3. **Push to Registry**: Pushes to ECR (simulated in demo)
4. **Deploy to ECS**: Updates ECS service (simulated in demo)

### Triggering the Pipeline

The pipeline runs on:
- Push to `main` branch
- Pull requests to `main` branch

### AWS Credentials for CI/CD

To enable actual deployments, add these secrets to your GitHub repository:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

Go to: Repository Settings → Secrets and variables → Actions → New repository secret

---

## 🔧 Troubleshooting

### Docker Desktop Not Running

**Error**: `Cannot connect to Docker daemon`

**Solution**:
1. Start Docker Desktop
2. Wait for it to fully start (green icon in system tray)
3. Verify with: `docker ps`

### LocalStack Not Responding

**Error**: `Connection refused to localhost:4566`

**Solution**:
```bash
# Check if LocalStack is running
docker ps

# If not running, start it
docker run -d --name localstack -p 4566:4566 localstack/localstack

# Check health
curl http://localhost:4566/_localstack/health
```

### Terraform Init Fails

**Error**: `Failed to download provider`

**Solution**:
1. Check internet connection
2. Clear Terraform cache: Delete `.terraform/` folder
3. Run `terraform init` again

### Port Already in Use

**Error**: `Port 3000 is already allocated`

**Solution**:
```bash
# Find process using the port
netstat -ano | findstr :3000

# Kill the process (Windows)
taskkill /PID  /F

# Or use a different port
docker run -p 3001:3000 navoy-demo-app:latest
```

### LocalStack ECS/ALB Errors

**Error**: `API for service ecs is not included in your license plan`

**This is expected!** See "Known Limitations with LocalStack" section above.

---

## 📖 Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)**: Detailed architecture design and decisions
- **[DEVOPS_REASONING.md](DEVOPS_REASONING.md)**: Answers to DevOps best practices questions

---

## 🏗️ Architecture Highlights

### Key Components

- **VPC**: Isolated network (10.0.0.0/16)
- **Subnets**: 2 public + 2 private (multi-AZ)
- **ALB**: Application Load Balancer for traffic distribution
- **ECS Fargate**: Serverless container orchestration
- **Auto Scaling**: Automatic scaling based on CPU/memory
- **Security Groups**: Least-privilege network security

### Scalability

- Horizontal auto-scaling (2-10 tasks)
- Multi-AZ deployment for high availability
- Load balancing across multiple containers

### Security

- Private subnets for application tier
- Security groups with minimal required access
- IAM roles with least privilege
- No hardcoded credentials

---

## 💡 Key Design Decisions

### Why ECS Fargate?
- **Serverless**: No server management overhead
- **Scalable**: Automatic scaling with demand
- **Cost-effective**: Pay only for what you use
- **Modern**: Industry-standard container orchestration

### Why Not EKS?
- **Complexity**: EKS adds Kubernetes overhead
- **Requirement**: Assignment specifies EKS as "design-only"
- **Simplicity**: ECS is simpler for this use case

### Why Multi-AZ?
- **High Availability**: Survives single AZ failure
- **Production-Ready**: Meets reliability requirements
- **Best Practice**: AWS recommendation for production workloads

---

## 🎓 What I Learned

- Infrastructure as Code with Terraform
- AWS networking and security best practices
- Container orchestration with ECS
- CI/CD pipeline design
- LocalStack limitations and workarounds

---

## 🚧 Future Improvements

See **[DEVOPS_REASONING.md](DEVOPS_REASONING.md)** for detailed improvement plans, including:
- Database integration (RDS)
- Advanced monitoring (CloudWatch, X-Ray)
- Multi-region deployment
- Cost optimization strategies
- Kubernetes migration path

---

## 📝 Assignment Checklist

- ✅ Architecture documentation (ARCHITECTURE.md)
- ✅ Infrastructure as Code (Terraform)
- ✅ CI/CD pipeline (GitHub Actions)
- ✅ DevOps reasoning (DEVOPS_REASONING.md)
- ✅ Runnable locally (LocalStack)
- ✅ Clean, modular code
- ✅ Clear documentation
- ✅ Trade-offs explained

---

## 📞 Contact

**Candidate**: Youssef Kefi  
**Email**: youssef.kefi@esprit.tn  
**Date**: January 27, 2026

---

## 📄 License

This project is created for the Navoy technical assessment.

---

## 🙏 Acknowledgments

- Navoy team for the interesting challenge
- Terraform and AWS documentation
- LocalStack for local AWS emulation