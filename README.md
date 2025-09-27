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
8. [Cost Optimization Tips](#cost-optimization-tips)
9. [Security Best Practices](#security-best-practices)
10. [Testing and Validation](#testing-and-validation)
11. [Cleanup](#cleanup)
12. [Extras: IaC Examples](#extras-iac-examples)
13. [References](#references)

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

```
![aws 3 tires](https://github.com/user-attachments/assets/3bc5a430-8093-4934-b7e0-4f257a869760)

```

Notes:

* Place web-facing ALB in public subnets with listeners on 80/443.
* EC2 instances should be in private subnets if you want better security (outbound via NAT gateway).
* RDS must always be in private subnets (DB subnet group), Multi-AZ for HA.

---

## Key Components

* **Amazon EC2**: Hosts the web application (stateless preferred).
* **Application Load Balancer (ALB)**: Distributes HTTP/HTTPS traffic.
* **Auto Scaling Group (ASG)**: Maintains minimum instances and scales out/in.
* **Amazon RDS (Optional)**: Managed relational DB with Multi-AZ.
* **IAM**: Roles for instances (SSM, S3 access), least-privilege policies.
* **CloudWatch & SNS**: Metrics, logs, and alerting.
* **S3 (Optional)**: For storing assets, logs, or deployment artifacts.

---

## Prerequisites

* An AWS account with permissions to create VPC, EC2, ALB, ASG, IAM roles, CloudWatch, and (optionally) RDS.
* AWS CLI configured locally (or use the Console).
* (Optional) A simple web application package (static site or simple server app) to serve from EC2.

---

## High-Level Deployment Steps

1. Create or choose a VPC with at least 2 public subnets and 2 private subnets across 2 AZs.
2. Create security groups for ALB and EC2 (least-privilege rules).
3. Create an IAM role for EC2 instances (allow SSM and any necessary permissions).
4. Prepare an AMI or use a community/official AMI and a user-data script to bootstrap the application.
5. Create a Launch Template or Launch Configuration for the ASG.
6. Create an ALB with listeners (HTTP/HTTPS) and target group for EC2 instances.
7. Create an Auto Scaling Group that spans multiple AZs and attaches to the target group.
8. Create CloudWatch alarms and SNS topics for notifications.
9. (Optional) Configure RDS with Multi-AZ for the database tier.

---

## Detailed Implementation Notes

### Networking (VPC/Subnets/Route Tables)

* Use a VPC with at least two AZs. For HA, create public subnets for ALB and private subnets for application instances.
* If EC2 instances need internet access for updates, deploy a NAT Gateway (cost) or NAT instances (lower cost but more management). Consider Session Manager (SSM) for management to avoid SSH and NAT costs.

### Security (IAM / Security Groups / Key Pairs)

* **EC2 Instance Role**: Attach an IAM role allowing SSM (AmazonSSMManagedInstanceCore) for secure remote management.
* **Security Group: ALB**

  * Inbound: HTTP (80) and HTTPS (443) from 0.0.0.0/0 (or lock to known CIDRs)
  * Outbound: Allow to instance SG
* **Security Group: EC2**

  * Inbound: Allow HTTP/HTTPS from ALB security group only
  * Outbound: Allow access to database (if needed) and outbound internet (for updates via NAT)
* Avoid distributing private SSH keys. Use SSM Session Manager for administrative access.

### EC2 Launch Configuration / AMI / User Data

* Use a hardened AMI (Amazon Linux 2, Ubuntu LTS).
* Use user-data to bootstrap the application (install web server, pull code from S3/Git, start service).
* Example user-data (bash):

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

* Bake AMI with Packer for production to reduce boot time.

### Auto Scaling Group (ASG) and Scaling Policies

* Configure ASG with a sensible minimum (e.g., 2), desired, and maximum (based on budget).
* Use scaling policies: target-tracking (e.g., keep average CPU at 50%) or request-based scaling (ALB RequestCountPerTarget).
* Cooldown and scale-in protection: set tune to avoid thrashing.

### Application Load Balancer (ALB) Configuration

* Create target group (health-check path `/health` or `/`) with HTTP health checks.
* Configure listener rules: redirect HTTP -> HTTPS for secure traffic.
* If using HTTPS, provision SSL/TLS cert via ACM (Regional or CloudFront depending on endpoint type).

### Optional: RDS Backend (Multi-AZ)

* Use Amazon RDS (MySQL/PostgreSQL) with Multi-AZ for automatic failover.
* Place RDS in private subnets; create a DB subnet group for AZs.
* Use IAM authentication if supported by your DB engine to reduce password management.

---

## Monitoring and Alerts (CloudWatch & SNS)

* Enable EC2 and ALB metrics in CloudWatch.
* Create alarms for:

  * High CPU (> 80% for 5 minutes)
  * High latency (ALB TargetResponseTime)
  * Instance unhealthy count
  * ASG scale-out events
* Create an SNS topic for alarm notifications and subscribe email/Slack webhook.
* Enable CloudWatch Logs for application logs (via CloudWatch agent or SSM agent forwarding).

---

## Cost Optimization Tips

* Use **Auto Scaling** to reduce instances during off-peak hours.
* Use **Spot Instances** for noncritical worker fleets or batch tasks (not for session-sensitive web tier unless designed for interruptions).
* Use **Savings Plans / Reserved Instances** for steady-state production workloads.
* Use **SSM Session Manager** to avoid NAT costs for SSH management.
* Keep AMI boot times small by baking dependencies to reduce instance runtime costs.

---

## Security Best Practices

* Use IAM roles for EC2, never embed credentials in AMI or user-data.
* Use Security Groups to restrict traffic (ALB -> EC2 only).
* Place DBs in private subnets and restrict access to only app instances.
* Enable encryption (EBS, RDS storage, S3 at rest) and use KMS CMKs for sensitive data.
* Perform regular vulnerability scans and enable Amazon Inspector or third-party tools.

---

## Testing and Validation

* Health checks: ensure ALB health checks match application readiness probe.
* Failover tests: kill an instance in the ASG and observe replacement and steady-state.
* Scaling tests: run a load test (ab using `ab`, `wrk`, or `k6`) and verify scale-out behavior.
* Security tests: verify that ALB is the only public entry and EC2 instances are not directly reachable.

---

## Cleanup

* Remove ASG, ALB, EC2 instances, and related resources when done to avoid costs.
* Delete S3 buckets or empty them and remove IAM roles created for the demo.

---

## Extras: IaC Examples

### Minimal CloudFormation (conceptual)

> NOTE: This is a conceptual snippet. For production, parameterize, modularize, and add outputs/conditions.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
  # ... subnets, IGW, route tables omitted for brevity
  AppRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: ami-0123456789abcdef0 # replace
        InstanceType: t3.micro
        IamInstanceProfile:
          Arn: !GetAtt AppInstanceProfile.Arn
        UserData: !Base64 |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl enable httpd
          systemctl start httpd

  MyASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      MinSize: '2'
      MaxSize: '6'
      DesiredCapacity: '2'
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      VPCZoneIdentifier: [subnet-aaaa, subnet-bbbb]
```

### Terraform / CDK

* Consider using Terraform or AWS CDK for more flexible, programmatic infrastructure definition.

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

Your Name - `your.email@example.com`

Enjoy deploying your scalable web app!
