# Strapi on AWS ECS Fargate — Terraform Infrastructure

Terraform that stands up a complete AWS environment for running a containerised [Strapi](https://strapi.io/) CMS on **ECS Fargate**, fronted by an Application Load Balancer with a Route 53 DNS record.

Everything is created from scratch — VPC, subnets, routing, security groups, IAM roles, the ECS cluster, the task definition, the service, and the load balancer. There is no dependency on pre-existing network infrastructure.

## Why this exists

Strapi is easy to run locally and annoying to run properly on AWS: it needs a persistent container, a stable public endpoint, an execution role that can pull from ECR and write logs, and a task role for whatever it talks to. This configuration encodes that whole shape once so a new environment is a `terraform apply` rather than an afternoon in the console.

Use it as:

- A working reference for a minimal but complete **ECS Fargate + ALB + Route 53** stack in Terraform
- A starting point for any single-container web service on Fargate, not just Strapi — swap the image and the port

## What it builds

| Layer | Resources |
| --- | --- |
| **Network** | VPC (`10.0.0.0/16`), internet gateway, two public subnets across `ap-south-1a` / `ap-south-1b`, public route table and associations |
| **Security** | Security groups for the ALB and the ECS tasks |
| **IAM** | ECS task execution role (ECR pull, CloudWatch Logs) and task role |
| **Compute** | ECS cluster, Fargate task definition, ECS service |
| **Ingress** | Application Load Balancer, target group, listener |
| **DNS** | Route 53 record pointing a subdomain at the ALB |

Two subnets in two availability zones is not decoration — an ALB requires at least two AZs.

## Files

| File | Contents |
| --- | --- |
| `main.tf` | Provider, VPC, subnets, internet gateway, routing |
| `security_groups.tf` | ALB and task security groups |
| `iam_roles.tf` | Task execution role and task role |
| `ecs_tasks.tf` | ECS cluster and task definition |
| `ecs_services.tf` | ECS service and its network configuration |
| `alb_and_dns.tf` | Load balancer, target group, listener, Route 53 record |
| `variables.tf` | Input variables |
| `outputs.tf` | Useful values after apply, including the public URL |
| `backend.tf` | Remote state backend configuration |

## Prerequisites

- Terraform 1.x and AWS credentials with permission to create VPC, ECS, IAM, ELB, and Route 53 resources
- A container image for Strapi in ECR or another registry the task can pull from
- A Route 53 hosted zone you control

## Usage

**1. Configure the remote state backend**

Edit `backend.tf` with your own S3 bucket and key before the first `init`. Remote state matters here — this stack creates IAM roles and networking, and local state is easy to lose.

**2. Set your variables**

Create a `terraform.tfvars`:

```hcl
aws_region      = "ap-south-1"
route53_zone_id = "Z0123456789ABCDEFGHIJ"
subdomain       = "strapi"
dockerimage     = "<account-id>.dkr.ecr.ap-south-1.amazonaws.com/strapi:latest"
```

**3. Apply**

```bash
terraform init
terraform plan
terraform apply
```

**4. Tear down when you are done**

```bash
terraform destroy
```

Fargate tasks, the ALB, and the NAT-free public subnets all bill by the hour. Destroy anything you are not using.

## Notes and limits

- **Tasks run in public subnets** with public IPs so they can reach ECR without a NAT gateway. That keeps the cost down and the topology simple; for anything production-facing, move tasks to private subnets and add a NAT gateway or VPC endpoints.
- **The region and CIDRs are hardcoded** to `ap-south-1` and `10.0.0.0/16` in `main.tf`. Change them there if you need something else.
- **There is no managed database.** Strapi defaults to SQLite inside the container, which does not survive a task replacement. Add RDS and point Strapi at it before storing anything you care about.
- **State is not locked** unless you add a DynamoDB table to the backend configuration. Do that before more than one person runs `apply`.
