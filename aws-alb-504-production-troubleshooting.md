# 🔧 Troubleshooting AWS ALB 504 Gateway Timeout in Production

![AWS](https://img.shields.io/badge/AWS-Cloud-orange)
![ALB](https://img.shields.io/badge/AWS-Application%20Load%20Balancer-blue)
![Troubleshooting](https://img.shields.io/badge/Focus-Production%20Troubleshooting-purple)
![Incident](https://img.shields.io/badge/Incident-HTTP%20504-red)

> **Real-world production troubleshooting case study**
>
> This guide documents a structured approach to investigating intermittent **HTTP 504 Gateway Timeout** errors involving an AWS Application Load Balancer, EC2, Apache, PHP-FPM, Auto Scaling, and CloudFront.

> ⚠️ **Sanitization notice**
>
> Customer names, AWS account IDs, IP addresses, domains, credentials, and proprietary configuration details have been removed or generalized.

---

## 📌 Table of Contents

- [Introduction](#-introduction)
- [Architecture: The Request Path](#-architecture-the-request-path)
- [Incident Symptoms](#-incident-symptoms)
- [Investigation Workflow](#-investigation-workflow)
  - [1. Confirm the ALB Symptoms](#1-confirm-the-alb-symptoms)
  - [2. Check Target Health](#2-check-target-health)
  - [3. Check EC2 Health](#3-check-ec2-health)
  - [4. Investigate Apache and PHP-FPM](#4-investigate-apache-and-php-fpm)
  - [5. Review PHP-FPM Configuration](#5-review-php-fpm-configuration)
  - [6. Validate the Health Endpoint](#6-validate-the-health-endpoint)
  - [7. Investigate Auto Scaling Configuration](#7-investigate-auto-scaling-configuration)
  - [8. Review CloudFront](#8-review-cloudfront)
  - [9. Add Capacity to Isolate the Problem](#9-add-capacity-to-isolate-the-problem)
- [Root Cause Analysis](#-root-cause-analysis)
- [Remediation](#-remediation)
- [Prevention](#-prevention)
- [Troubleshooting Checklist](#-troubleshooting-checklist)
- [Useful Commands](#-useful-commands)
- [Key Takeaways](#-key-takeaways)
- [Final Thoughts](#-final-thoughts)

---

## 🎯 Introduction

An **HTTP 504 Gateway Timeout** from an AWS Application Load Balancer does **not automatically mean that the ALB itself is broken**.

In a production environment, the request can pass through several components before the application returns a response.

A failure at any downstream layer can appear to the user as a timeout.

### The important mindset

Do not start by changing the ALB configuration.

Start by asking:

> **Where is the request getting stuck?**

---

## 🏗️ Architecture: The Request Path

A simplified request path for this environment looked like:

```text
Client
  |
  v
CloudFront
  |
  v
Application Load Balancer
  |
  v
Target Group
  |
  v
EC2
  |
  v
Apache
  |
  v
PHP-FPM
  |
  v
Application
```

This gives us multiple troubleshooting layers:

```text
+--------------------------------------------------+
| Client / CloudFront                              |
+--------------------------------------------------+
                      |
                      v
+--------------------------------------------------+
| Application Load Balancer                        |
| - Target health                                  |
| - 5xx responses                                  |
| - Response time                                  |
+--------------------------------------------------+
                      |
                      v
+--------------------------------------------------+
| EC2                                              |
| - CPU / Memory                                   |
| - Disk / EBS                                     |
| - Status checks                                  |
+--------------------------------------------------+
                      |
                      v
+--------------------------------------------------+
| Apache                                           |
| - Access logs                                    |
| - Error logs                                     |
+--------------------------------------------------+
                      |
                      v
+--------------------------------------------------+
| PHP-FPM                                          |
| - Workers                                        |
| - max_children                                   |
| - Process saturation                             |
+--------------------------------------------------+
                      |
                      v
+--------------------------------------------------+
| Application                                      |
| - Application errors                             |
| - Database calls                                 |
| - External dependencies                          |
+--------------------------------------------------+
```

---

## 🚨 Incident Symptoms

The production application was intermittently returning:

```text
HTTP 504 Gateway Timeout
```

Initial observations included:

- Users were intermittently receiving 504 responses.
- The ALB reported the affected target as unhealthy.
- EC2 status checks were passing.
- CPU utilization was low.
- EBS queue length was normal.
- One application instance was behaving differently from the others.
- Application logs showed PHP-FPM / `proxy_fcgi` / timeout-related errors.
- Replacing the affected instance did not immediately eliminate the problem.

### 🧠 Key Insight

> **Low EC2 CPU does not mean the application is healthy.**

A web application can be unhealthy because of process exhaustion, blocked workers, slow dependencies, connection limits, application-level locking, or other bottlenecks while CPU remains low.

---

# 🔎 Investigation Workflow

## 1. Confirm the ALB Symptoms

Start with the ALB rather than immediately logging into an EC2 instance.

Review metrics such as:

```text
HTTPCode_ELB_5XX_Count
HTTPCode_Target_5XX_Count
TargetResponseTime
HealthyHostCount
UnHealthyHostCount
RequestCount
```

The objective is to determine whether the failure is:

```text
Client
   |
   v
ALB
   |
   +---- ALB-side failure
   |
   +---- Target-side failure
             |
             v
          EC2/Application
```

### Questions to answer

- Are requests reaching the target?
- Is the target healthy?
- Is target response time increasing?
- Are 5xx responses generated by the ALB or the application?
- Did the problem begin after a deployment or infrastructure change?

---

## 2. Check Target Health

Open the ALB target group and inspect the affected target.

Check:

- Target health state
- Health-check path
- Health-check port
- Expected success codes
- Health-check interval
- Health-check timeout
- Recent health-state transitions

A critical distinction is:

```text
EC2 = Healthy
```

does not necessarily mean:

```text
Application = Healthy
```

The EC2 instance can pass infrastructure-level status checks while the web application is unable to serve requests.

---

## 3. Check EC2 Health

Once the affected target is identified, investigate the instance.

### EC2 status

Verify:

```text
System status check
Instance status check
```

### CPU

```bash
top
```

or:

```bash
htop
```

### Memory

```bash
free -m
```

### Disk

```bash
df -h
```

### Disk I/O

```bash
iostat
```

### Processes

```bash
ps aux --sort=-%cpu | head
```

and:

```bash
ps aux --sort=-%mem | head
```

Also review CloudWatch metrics for:

- CPUUtilization
- NetworkIn
- NetworkOut
- EBS volume metrics
- StatusCheckFailed

### Important

Do not stop troubleshooting just because these metrics look normal.

They only tell you that the **instance may be healthy at the infrastructure layer**.

---

## 4. Investigate Apache and PHP-FPM

The next layer is the web server and PHP runtime.

### Check Apache

For RHEL-based systems:

```bash
systemctl status httpd
```

Review the error log:

```bash
tail -f /var/log/httpd/error_log
```

Look for messages involving:

```text
proxy_fcgi
timeout
upstream
connection reset
connection refused
```

### Check PHP-FPM

```bash
systemctl status php-fpm
```

Review service logs:

```bash
journalctl -u php-fpm
```

Depending on the PHP/OS configuration, PHP-FPM logs may also be under:

```text
/var/log/
```

### What are we looking for?

Particularly investigate:

```text
pm.max_children
worker saturation
slow requests
process spawning
process termination
connection limits
```

---

## 5. Review PHP-FPM Configuration

For a dynamic PHP application, PHP-FPM process management can become a critical bottleneck.

Important parameters include:

```ini
pm = dynamic

pm.max_children
pm.start_servers
pm.min_spare_servers
pm.max_spare_servers
```

For example:

```ini
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
```

> ⚠️ These values are examples from an investigated environment. Do not blindly copy them into another production environment.

The correct values depend on:

- Instance memory
- PHP worker memory consumption
- Application response time
- Concurrent requests
- Traffic pattern
- Database latency
- External dependencies

After making a validated configuration change:

```bash
systemctl restart php-fpm
```

Then verify:

```bash
systemctl status php-fpm
```

---

## 6. Validate the Health Endpoint

A lightweight health endpoint can make infrastructure troubleshooting much easier.

Example:

```text
/health.html
```

Test locally:

```bash
curl -I http://localhost/health.html
```

You can also test the application endpoint:

```bash
curl -I http://localhost/
```

This helps separate:

```text
Web server is responding
```

from:

```text
Application is responding correctly
```

### Recommended health-check design

Keep the basic infrastructure health endpoint lightweight.

Avoid making the load balancer health check perform expensive operations such as:

```text
Large database queries
External API calls
Long-running application logic
```

---

## 7. Investigate Auto Scaling Configuration

If instances are managed by an Auto Scaling Group, configuration consistency becomes critical.

A common production problem is:

```text
Instance A
   |
   +-- Correct configuration

Instance B
   |
   +-- Old configuration
```

Then the ASG replaces an instance using an outdated Launch Template or AMI:

```text
ASG
 |
 v
Launch Template
 |
 v
Old AMI
 |
 v
New instance with old configuration
```

This can make an incident appear to "come back" after an instance replacement.

### Better approach

```text
Fix configuration
       |
       v
Validate
       |
       v
Create Golden AMI
       |
       v
Update Launch Template
       |
       v
Update Auto Scaling Group
       |
       v
Launch replacement instance
       |
       v
Validate target health
```

This reduces configuration drift.

---

## 8. Review CloudFront

If CloudFront sits in front of the ALB, include it in the investigation.

Review:

- Origin configuration
- Origin protocol
- Cache behavior
- Cache policy
- Origin request policy
- Allowed methods
- TTL configuration
- Recent changes
- Cache invalidations

The full request path may be:

```text
User
 |
 v
CloudFront
 |
 v
ALB
 |
 v
EC2
 |
 v
Application
```

A successful test at one layer does not prove that every layer is working correctly.

---

## 9. Add Capacity to Isolate the Problem

One useful diagnostic technique is to temporarily add another application instance.

The goal is not simply to "scale up."

The goal is to answer:

> **Does the problem follow the workload, or does it remain isolated to a specific instance?**

For example:

```text
Before

ALB
 |
 +---- EC2-A  --> Problem
 |
 +---- EC2-B  --> Healthy


After adding capacity

ALB
 |
 +---- EC2-A  --> Problem
 |
 +---- EC2-B  --> Healthy
 |
 +---- EC2-C  --> Healthy
```

If traffic successfully moves to healthy instances while one instance consistently fails, the investigation can focus on instance-specific or configuration-specific behavior.

---

# 🧩 Root Cause Analysis

The important outcome of this investigation was that the issue could not be explained by simply saying:

> "The ALB was returning 504."

The ALB was reporting a symptom of a downstream application problem.

The investigation crossed multiple layers:

```text
ALB
 |
 +-- Target health
 |
 v
EC2
 |
 +-- Apache
 |
 v
PHP-FPM
 |
 +-- Worker/process behavior
 |
 v
Application
```

The investigation identified abnormal PHP-FPM worker behavior and configuration as an important part of the application-side problem.

At the same time, Auto Scaling and image/template consistency were important because replacing an instance without fixing the underlying configuration could reproduce the issue.

---

# 🛠️ Remediation

The remediation process included:

- Investigating ALB target health.
- Reviewing ALB metrics.
- Investigating EC2 health.
- Reviewing Apache logs.
- Reviewing PHP-FPM logs.
- Reviewing PHP-FPM process configuration.
- Adjusting PHP-FPM configuration based on workload.
- Restarting and validating PHP-FPM.
- Creating a lightweight health endpoint.
- Testing application response locally.
- Adding an additional instance to isolate the problem.
- Creating a corrected Golden AMI.
- Updating the Launch Template.
- Updating Auto Scaling configuration.
- Reviewing CloudFront behavior.
- Monitoring the environment after remediation.

---

# 🛡️ Prevention

Fixing the immediate incident is only half the job.

The second half is preventing recurrence.

## 1. Use meaningful health checks

Health checks should detect whether the service can actually serve traffic.

## 2. Standardize instance configuration

Use:

```text
Golden AMI
+
Launch Template
+
Auto Scaling Group
```

instead of manually maintaining individual instances.

## 3. Monitor application-layer metrics

Do not monitor only:

```text
CPU
Memory
Disk
```

Also monitor:

```text
ALB TargetResponseTime
ALB 5xx
HealthyHostCount
PHP-FPM worker behavior
Application errors
Apache errors
```

## 4. Centralize logs

Send relevant application and infrastructure logs to a centralized logging solution such as CloudWatch Logs.

## 5. Create meaningful alerts

Useful signals include:

```text
UnHealthyHostCount
TargetResponseTime
HTTP 5xx
Application errors
PHP-FPM saturation
```

## 6. Track configuration changes

When an incident occurs, ask:

```text
What changed?
When did it change?
Which instances received the change?
Was the Launch Template updated?
Was the AMI updated?
```

---

# ✅ Troubleshooting Checklist

Use this checklist when investigating an AWS ALB 504.

```text
[ ] Confirm the affected URL
[ ] Confirm the time window
[ ] Check ALB 5xx metrics
[ ] Check target response time
[ ] Check target health
[ ] Check health-check configuration
[ ] Check EC2 status checks
[ ] Check CPU
[ ] Check memory
[ ] Check disk
[ ] Check EBS metrics
[ ] Check Apache status
[ ] Check Apache error logs
[ ] Check PHP-FPM status
[ ] Check PHP-FPM logs
[ ] Check PHP-FPM worker behavior
[ ] Test the health endpoint locally
[ ] Test the application locally
[ ] Check recent deployments
[ ] Check recent infrastructure changes
[ ] Check Launch Template version
[ ] Check AMI version
[ ] Check Auto Scaling configuration
[ ] Check CloudFront configuration
[ ] Add capacity if required for isolation
[ ] Validate remediation
[ ] Monitor after the fix
[ ] Document the root cause
[ ] Implement preventive controls
```

---

# 🧰 Useful Commands

## Apache

```bash
systemctl status httpd
```

```bash
tail -f /var/log/httpd/error_log
```

## PHP-FPM

```bash
systemctl status php-fpm
```

```bash
journalctl -u php-fpm
```

## System resources

```bash
top
```

```bash
free -m
```

```bash
df -h
```

```bash
iostat
```

## Processes

```bash
ps aux --sort=-%cpu | head
```

```bash
ps aux --sort=-%mem | head
```

## Local application testing

```bash
curl -I http://localhost/
```

```bash
curl -I http://localhost/health.html
```

---

# 💡 Key Takeaways

### 1. 504 is a symptom

Do not assume the ALB is the root cause.

### 2. Infrastructure health is not application health

An EC2 instance can pass status checks while the application is unavailable.

### 3. Troubleshoot layer by layer

```text
CloudFront
   ↓
ALB
   ↓
Target Group
   ↓
EC2
   ↓
Apache
   ↓
PHP-FPM
   ↓
Application
```

### 4. Evidence before changes

Collect metrics and logs before making production changes whenever possible.

### 5. Configuration drift matters

If an ASG launches instances from an outdated AMI or Launch Template, the problem can return.

### 6. Fix the system, not just the instance

A manual fix on one EC2 instance is not a permanent solution when instances are managed by Auto Scaling.

### 7. Prevention matters

A successful incident response should result in better monitoring, health checks, automation, and configuration consistency.

---

# 🧠 Final Thoughts

Production troubleshooting is rarely about knowing a single AWS command.

It is about developing a repeatable investigation process.

When an ALB returns a 504, avoid jumping directly to:

```text
"Increase the timeout."
```

Instead ask:

```text
Where did the request stop progressing?
```

Then follow the request path:

```text
Client
  ↓
CloudFront
  ↓
ALB
  ↓
Target
  ↓
EC2
  ↓
Apache
  ↓
PHP-FPM
  ↓
Application
```

**Observe → Isolate → Investigate → Remediate → Validate → Prevent**

That mindset is more valuable than memorizing a list of AWS troubleshooting commands.

---

## 📚 Related Guides

More production troubleshooting scenarios will be added to this repository:

- AWS RDS connectivity
- EC2 application unresponsiveness
- Auto Scaling configuration drift
- CloudWatch investigation
- AWS networking and Security Groups
- EKS troubleshooting
- ECR → EKS deployments
- Terraform production troubleshooting

---

## 👨‍💻 Author

**Nitish**

AWS Cloud Engineer focused on:

- AWS Cloud Operations
- Kubernetes / EKS
- Terraform
- Infrastructure Automation
- DevOps
- AI Infrastructure / AIOps

---

> ⭐ If this guide helped you troubleshoot an AWS incident, consider starring the repository and sharing your own troubleshooting lessons.
