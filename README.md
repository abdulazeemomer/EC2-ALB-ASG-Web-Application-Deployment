# EC2-ALB-ASG-Web-Application-Deployment
This project demonstrates the deployment of a secure, highly available, and scalable web application on AWS using EC2 instances, an Application Load Balancer (ALB), and an Auto Scaling Group (ASG). The setup ensures fault tolerance, cost optimization, and operational efficiency following AWS best practices.
# EC2 + ALB + ASG Web Application

> Deploy a simple, highly available, and scalable web application on AWS using EC2 instances behind an Application Load Balancer (ALB) and Auto Scaling Groups (ASG). This README documents architecture, deployment steps, security best practices, monitoring, and cost optimization guidance for a demo project.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Key Components](#key-components)
4. [Prerequisites](#prerequisites)
5. [High-Level Deployment Steps](#high-level-deployment-steps)
6. [Detailed Implementation Notes](#detailed-implementation-notes)

   * [Networking (VPC/Subnets/Route Tables)](#networking-vpcsubnetsroute-tables)
   * [Security (IAM / Security Groups / Key Pairs)](#security-iam--security-groups--key-pairs)
   * [EC2 Launch Configuration / AMI / User Data](#ec2-launch-configuration--ami--user-data)
   * [Auto Scaling Group (ASG) and Scaling Policies](#auto-scaling-group-asg-and-scaling-policies)
   * [Application Load Balancer (ALB) Configuration](#application-load-balancer-alb-configuration)
   * [Optional: RDS Backend (Multi-AZ)](#optional-rds-backend-multi-az)
7. [Monitoring and Alerts (CloudWatch & SNS)](#monitoring-and-alerts-cloudwatch--sns)
8. [Testing and Validation](#testing-and-validation)

---

## Overview

This project demonstrates how to run a simple web application on EC2 instances with high availability and automatic scaling. Traffic is balanced across instances using an ALB. An ASG manages EC2 lifecycle and scales the fleet based on demand (CPU, request count, or custom metrics). Optionally, the application can use Amazon RDS (MySQL/PostgreSQL) in Multi-AZ for durable relational storage.

Goals:

* High availability across multiple Availability Zones (AZs)
* Auto-scaling to handle spikes and conserve cost at low demand
* Secure by default (least privilege, secure networking)
* Observability using CloudWatch with alerts via SNS

---

## Architecture Diagram

<img width="2580" height="1270" alt="aws3tires" src="https://github.com/user-attachments/assets/a0c38d2e-9261-44e6-a337-91e6540d6aad" />

Notes:


* Place web-facing ALB in public subnets with listeners on 80/443.
* EC2 instances for application should be in private subnets if you want better security (outbound via NAT gateway).
* RDS must always be in private subnets (DB subnet group), Multi-AZ for HA.

---

## Key Components

* **Amazon EC2**: Hosts the web application.
* **Application Load Balancer (ALB)**: Distributes HTTP/HTTPS traffic.
* **Auto Scaling Group (ASG)**: Maintains minimum instances and scales out/in.
* **Amazon RDS (Optional)**: Managed relational DB with Multi-AZ.
* **IAM**: Roles for instances , least-privilege policies.
* **CloudWatch & SNS**: Metrics, logs, and alerting.

---

## Prerequisites

* An AWS account with permissions to create VPC, EC2, ALB, ASG, IAM roles, CloudWatch, and (optionally) RDS.
* A simple web application package (static site or simple server app) to serve from EC2.

---

## High-Level Deployment Steps

1.Creating  a VPC with  2 public subnets and 4 private subnets across 2 AZs.
2. Creating security groups for ALB and EC2 (least-privilege rules).
3. Creating an IAM role for EC2 instances (allow SSM and any necessary permissions).
4. Preparing an AMI or use a community/official AMI and a user-data script to bootstrap the application.
5. Creating a Launch Template or Launch Configuration for the ASG.
6. Creating an ALB with listeners (HTTP/HTTPS) and target group for EC2 instances.
7. Creating an Auto Scaling Group that spans multiple AZs and attaches to the target group.
8. Creating CloudWatch alarms and SNS topics for notifications.
9. Configure RDS with Multi-AZ for the database tier.

---

## Detailed Implementation Notes

### Networking (VPC/Subnets/Route Tables)

* A VPC with two AZs. For HA,   public subnets for web tire and ALB. And private subnets for application instances and database.
* EC2 instances in private subnet need internet access for updates, deploy a NAT Gateway.

### Security (IAM / Security Groups / Key Pairs)

* **EC2 Instance Role**: Attaching an IAM role allowing SSM (AmazonSSMManagedInstanceCore) for secure remote management.
* **Security Group: ALB**

  * Inbound: HTTP (80) and HTTPS (443) from 0.0.0.0/0 )
  * Outbound: Allow to instance SG
* **Security Group: EC2**

  * Inbound: Allow HTTP/HTTPS from ALB security group only
  * Outbound: Allow access to database and outbound internet (for updates via NAT)
  * Distributing private SSH keys. Use SSM Session Manager for administrative access.

### EC2 Launch Configuration / AMI / User Data

* A hardened AMI (Amazon Linux 2, Ubuntu LTS).
   user-data to bootstrap the application (install web server, pull code from S3/Git, start service).
* user-data (bash):

```bash
#!/bin/bash
yum update -y
yum install -y httpd
# Pull website from S3 or git
aws s3 cp s3://my-app-bucket/site.zip /tmp/site.zip
unzip /tmp/site.zip -d /var/www/html
systemctl enable httpd
systemctl start httpd
# Install SSM agent if not present
```

### Auto Scaling Group (ASG) and Scaling Policies

* Configure ASG with a sensible minimum (2), and maximum (4).
* scaling policies: target-tracking (keep average CPU at 50%) 


### Application Load Balancer (ALB) Configuration

* Target group (health-check path `/health` or `/`) with HTTP health checks.
* Configuring listener rules: redirect HTTP -> HTTPS for secure traffic.
* For HTTPS, provision SSL/TLS cert via ACM.

###  RDS Backend (Multi-AZ)

* Amazon RDS (MySQL) with Multi-AZ for automatic failover.
* RDS placed in private subnets.

---

## Monitoring and Alerts (CloudWatch & SNS)

* Enabling EC2 and ALB metrics in CloudWatch.
* Create alarms for:

  * High CPU (> 80% for 5 minutes)
  * High latency (ALB TargetResponseTime)
  * Instance unhealthy count
  * ASG scale-out events
* Creating an SNS topic for alarm notifications and subscribe email/Slack webhook.
* Enabling CloudWatch Logs for application log.
  
---

## Testing and Validation

* Health checks: ensure ALB health checks match application readiness probe.
* Failover tests: kill an instance in the ASG and observe replacement and steady-state.
* Scaling tests: run a load test (ab using `ab`, `wrk`, or `k6`) and verify scale-out behavior.
* Security tests: verify that ALB is the only public entry and EC2 instances are not directly reachable.

---


## References

* [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
* [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
* [Auto Scaling Groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
* [Amazon EC2 Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/best-practices.html)

---

## License

MIT

---

## Author

Abdulazeem Omer - `abdulazeemomer@gmail.com`


