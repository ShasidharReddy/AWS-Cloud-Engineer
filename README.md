# AWS Cloud Engineer — Reference & Lab Notes

A curated collection of AWS command references, architecture diagrams, scripts, and step-by-step guides covering core AWS services. Organized by topic — use this as a quick reference while working on AWS projects.

---

## 📁 Directory Structure

| Directory | Description |
|-----------|-------------|
| [`Architecture/`](./Architecture/) | 🔷 **Visual diagrams** — 20+ Mermaid flow diagrams for all major AWS services |
| [`Networking/`](./Networking/) | VPC, Security Groups, NACLs, Transit Gateway, Direct Connect, Route 53, CloudFront |
| [`IAM-Security/`](./IAM-Security/) | IAM, KMS, WAF, GuardDuty, Security Hub, CloudTrail, Organizations, SCPs |
| [`Compute/`](./Compute/) | EC2, Auto Scaling, EBS, AMIs, Placement Groups, Spot, Elastic Beanstalk |
| [`Storage/`](./Storage/) | S3, EBS, EFS, FSx, Storage Gateway, Snow Family, DataSync, Backup |
| [`Database/`](./Database/) | RDS, Aurora, DynamoDB, ElastiCache, Redshift, DocumentDB, DMS |
| [`Monitoring/`](./Monitoring/) | CloudWatch, CloudTrail, X-Ray, EventBridge, Systems Manager, Config |
| [`DataPipeline/`](./DataPipeline/) | Kinesis, SQS/SNS, Glue, Athena, EMR, Step Functions, Redshift, QuickSight |
| [`CostOptimization/`](./CostOptimization/) | Reserved Instances, Savings Plans, Spot, Cost Explorer, Budgets, FinOps |
| [`Serverless/`](./Serverless/) | Lambda, API Gateway, Step Functions, EventBridge, SAM, Cognito, AppSync |
| [`Containers/`](./Containers/) | ECS, EKS, Fargate, ECR, App Runner, Service Mesh |
| [`Migration/`](./Migration/) | 7 Rs, AWS MGN, DMS, Snow Family, cross-cloud migration guides |
| [`CICD/`](./CICD/) | CodePipeline, CodeBuild, CodeDeploy, CloudFormation, CDK, Terraform, GitOps |

---

## 🚀 Quick Start

### Prerequisites

- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured
- An AWS account with appropriate permissions:
  ```bash
  aws configure
  aws sts get-caller-identity
  ```

### Recommended Reading Order

1. Start with [`Architecture/`](./Architecture/) for visual overview of all services
2. Explore [`Compute/`](./Compute/) for EC2, Auto Scaling, and VM management
3. Review [`Networking/`](./Networking/) for VPC, subnets, and connectivity
4. Use the topic directories for specific service deep-dives

---

## 🏷️ Topics Covered

`EC2` · `VPC` · `S3` · `IAM` · `RDS` · `Aurora` · `DynamoDB` · `Lambda` · `API Gateway` · `ECS` · `EKS` · `Fargate` · `CloudWatch` · `CloudTrail` · `CloudFormation` · `CDK` · `Terraform` · `Route 53` · `CloudFront` · `ElastiCache` · `Redshift` · `Kinesis` · `SQS/SNS` · `Step Functions` · `Glue` · `Athena` · `EMR` · `KMS` · `WAF` · `GuardDuty` · `Security Hub` · `Direct Connect` · `Transit Gateway` · `Cost Explorer` · `Savings Plans` · `Organizations` · `Control Tower`

---

## 🔷 Visual Diagrams

This repo includes **Mermaid flow diagrams** that render directly on GitHub — no images needed. See [`Architecture/`](./Architecture/) for:

- AWS Global Infrastructure (Regions, AZs, Edge Locations)
- Compute decision tree (EC2 vs ECS vs EKS vs Lambda vs Fargate)
- EC2 instance lifecycle state diagram
- VPC multi-AZ architecture with NAT Gateway
- ALB request flow and target groups
- S3 storage classes and lifecycle transitions
- EKS cluster architecture
- Lambda + API Gateway sequence flow
- RDS Multi-AZ and Aurora architecture
- DynamoDB partitioning and DAX caching
- CI/CD pipeline with CodePipeline
- 3-tier web application reference architecture
- Data lake architecture (S3 → Glue → Athena → QuickSight)
- Disaster recovery strategies
- Well-Architected Framework pillars
- On-premises and cross-cloud migration flows

---

## 📝 Notes

- All guides include practical `aws` CLI commands — replace placeholder values before running.
- Each guide includes Mermaid diagrams, best practices, and comparison tables.
- Guides are for **learning and reference** purposes — review before running in production.
