# AWS Production Troubleshooting

A practical collection of AWS production troubleshooting guides based on real-world cloud engineering scenarios.

This repository focuses on diagnosing and resolving production issues across AWS infrastructure, networking, compute, databases, load balancing, monitoring, and application infrastructure.

## What You'll Find

* Application Load Balancer troubleshooting
* EC2 application and infrastructure issues
* Auto Scaling troubleshooting
* RDS connectivity issues
* CloudWatch investigation techniques
* Security Group and network troubleshooting
* Application health-check problems
* PHP-FPM and Apache troubleshooting
* Infrastructure configuration drift
* Production incident investigation methodologies

## Troubleshooting Philosophy

Production troubleshooting should not start with random configuration changes.

The general approach used in these guides is:

1. Identify the symptom
2. Confirm the scope and impact
3. Collect evidence
4. Check AWS infrastructure health
5. Check application health
6. Identify the root cause
7. Apply the smallest safe remediation
8. Validate the fix
9. Prevent recurrence

## Case Studies

### Application Load Balancer

* [ALB 504 Gateway Timeout](alb/alb-504-gateway-timeout.md)

### EC2

* [EC2 Instance Healthy but Application Unresponsive](ec2/ec2-application-unresponsive.md)

### Auto Scaling

* [Auto Scaling Configuration Drift](autoscaling/asg-configuration-drift.md)

### RDS

* [RDS Connectivity Troubleshooting](rds/rds-connectivity-troubleshooting.md)

### CloudWatch

* [Production Investigation Workflow](cloudwatch/production-investigation-workflow.md)

### Networking

* [Security Group Connectivity Troubleshooting](networking/security-group-connectivity.md)

## AWS Services Covered

* Amazon EC2
* Elastic Load Balancing
* Auto Scaling
* Amazon CloudWatch
* Amazon RDS
* Amazon VPC
* AWS Systems Manager
* Amazon CloudFront
* Amazon S3
* AWS IAM

## Important Note

The examples in this repository are educational and intentionally sanitized.

No customer names, account IDs, IP addresses, credentials, proprietary configurations, or confidential information are included.

## Goal

The goal of this repository is to help cloud engineers move beyond simply knowing AWS services and develop a structured approach to troubleshooting real production problems.

---

Maintained by **Nitish** — AWS Cloud Engineer.
