# 🗄️ AWS RDS Connectivity Troubleshooting

![AWS](https://img.shields.io/badge/AWS-Cloud-orange)
![RDS](https://img.shields.io/badge/AWS-RDS-blue)
![Networking](https://img.shields.io/badge/Focus-Networking-purple)
![Troubleshooting](https://img.shields.io/badge/Type-Production%20Troubleshooting-green)

> **A practical engineer's guide to diagnosing AWS RDS connectivity problems**

An RDS instance being **available** does not automatically mean that an application can connect to it.

When an application cannot connect to Amazon RDS, the fastest approach is not to randomly modify Security Groups or restart the database. Instead, identify the exact layer where the connection fails.

---

## 📌 Table of Contents

- [Introduction](#-introduction)
- [The Connectivity Path](#-the-connectivity-path)
- [Typical Symptoms](#-typical-symptoms)
- [Troubleshooting Methodology](#-troubleshooting-methodology)
  - [1. DNS Resolution](#1-dns-resolution)
  - [2. Network Reachability](#2-network-reachability)
  - [3. Security Groups](#3-security-groups)
  - [4. Route Tables](#4-route-tables)
  - [5. Network ACLs](#5-network-acls)
  - [6. RDS Configuration](#6-rds-configuration)
  - [7. Database Authentication](#7-database-authentication)
- [PostgreSQL Troubleshooting](#-postgresql-troubleshooting)
- [MySQL Troubleshooting](#-mysql-troubleshooting)
- [Common Failure Scenarios](#-common-failure-scenarios)
- [Troubleshooting Checklist](#-troubleshooting-checklist)
- [Useful Commands](#-useful-commands)
- [Security Best Practices](#-security-best-practices)
- [Key Takeaways](#-key-takeaways)
- [Final Thoughts](#-final-thoughts)

---

# 🎯 Introduction

One of the most common AWS application issues is:

```text
Application
     |
     v
Amazon RDS
     |
     X
Connection failed
```

The error may look like a database problem, but the actual failure could be caused by:

- DNS resolution
- Routing
- Security Groups
- Network ACLs
- Incorrect database port
- Subnet configuration
- Database listener availability
- SSL/TLS configuration
- Authentication
- Incorrect database name
- Incorrect credentials

The goal is therefore not simply:

> "Make the connection work."

The goal is:

> **Identify the failing layer and fix the actual problem.**

---

# 🏗️ The Connectivity Path

A typical private AWS architecture looks like:

```text
                    AWS VPC
┌─────────────────────────────────────────────────────┐
│                                                     │
│   Application Subnet             Database Subnet    │
│                                                     │
│   ┌───────────────┐              ┌──────────────┐  │
│   │   EC2 / EKS   │              │     RDS      │  │
│   │  Application  │─────────────>│  PostgreSQL  │  │
│   └───────────────┘ TCP 5432     └──────────────┘  │
│          │                                │         │
│          │                                │         │
│     Security Group                 Security Group  │
│          │                                │         │
│          └────────── Route Tables ─────────┘         │
│                                                     │
│                Network ACLs                          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

The connection can fail at any of these layers.

---

# 🚨 Typical Symptoms

Common application errors include:

```text
connection timed out
```

```text
connection refused
```

```text
could not connect to server
```

```text
no route to host
```

```text
temporary failure in name resolution
```

```text
password authentication failed
```

```text
database does not exist
```

These errors are not equivalent.

For example:

```text
DNS failure
```

is fundamentally different from:

```text
TCP connection timeout
```

which is different again from:

```text
authentication failure
```

Understanding the error category immediately narrows the investigation.

---

# 🔎 Troubleshooting Methodology

Use this sequence:

```text
1. DNS
   ↓
2. Network reachability
   ↓
3. Security Groups
   ↓
4. Route Tables
   ↓
5. Network ACLs
   ↓
6. Database listener / port
   ↓
7. Database configuration
   ↓
8. Authentication
```

Do not skip directly to credentials if TCP connectivity itself is failing.

---

# 1. DNS Resolution

Start by verifying that the RDS endpoint resolves.

Example:

```bash
nslookup <rds-endpoint>
```

On Linux:

```bash
dig <rds-endpoint>
```

Example:

```bash
dig +short example-db.xxxxxx.ap-south-1.rds.amazonaws.com
```

### What are we checking?

We want to know whether:

```text
Application host
      |
      v
RDS DNS endpoint
      |
      v
IP address
```

is working correctly.

### If DNS fails

Investigate:

- VPC DNS support
- VPC DNS hostnames
- Resolver configuration
- `/etc/resolv.conf`
- Custom DNS servers
- Network-level DNS restrictions

Do not start changing RDS Security Groups when DNS resolution itself is failing.

---

# 2. Network Reachability

Once DNS works, test TCP connectivity.

For PostgreSQL:

```bash
nc -zv <rds-endpoint> 5432
```

For MySQL:

```bash
nc -zv <rds-endpoint> 3306
```

Example:

```bash
nc -zv example-db.xxxxxx.ap-south-1.rds.amazonaws.com 5432
```

Possible outcomes:

### Successful

```text
Connection succeeded
```

The network path and port are reachable.

Move to database-level testing.

### Timeout

```text
Connection timed out
```

Investigate:

- Security Groups
- Route Tables
- Network ACLs
- Subnets
- Network path

### Connection refused

```text
Connection refused
```

Investigate:

- Database availability
- Port
- Listener
- Service configuration
- Incorrect endpoint/port

---

# 3. Security Groups

Security Groups are one of the first things engineers check, but they should be checked **correctly**.

For example:

```text
Application EC2
Security Group: sg-app
        |
        | TCP 5432
        v
RDS
Security Group: sg-db
```

The RDS Security Group should allow the required database port from the application Security Group.

Example:

```text
Inbound rule

Type: PostgreSQL
Protocol: TCP
Port: 5432
Source: sg-app
```

For MySQL:

```text
Type: MySQL/Aurora
Protocol: TCP
Port: 3306
Source: sg-app
```

### Prefer Security Group references

Prefer:

```text
Source = sg-app
```

over:

```text
Source = 0.0.0.0/0
```

when the application is inside the same AWS environment.

Opening a database to the entire internet is usually a terrible troubleshooting shortcut.

---

# 4. Route Tables

Security Groups are not the only networking component.

Verify the route between the application subnet and RDS subnet.

Conceptually:

```text
Application Subnet
        |
        v
Route Table
        |
        v
RDS Subnet
```

Check:

- Application subnet route table
- RDS subnet association
- VPC CIDR
- Subnet CIDRs
- Any relevant peering/TGW routes
- Cross-VPC routing if applicable

For a normal RDS deployment inside the same VPC, the VPC's local route should provide connectivity between the relevant subnets.

---

# 5. Network ACLs

Network ACLs are stateless.

That means return traffic must also be permitted.

Check both directions where relevant:

```text
Application subnet NACL
        |
        v
RDS subnet NACL
        |
        v
Return traffic
```

A Security Group may be correct while an overly restrictive Network ACL still blocks the connection.

Review:

- Inbound rules
- Outbound rules
- Ephemeral ports
- Rule numbers
- Allow/deny ordering

Do not modify NACLs blindly.

First identify whether the NACL is actually involved in the failing path.

---

# 6. RDS Configuration

Once network connectivity works, investigate the database itself.

Verify:

### Database engine

```text
PostgreSQL
MySQL
MariaDB
Aurora
etc.
```

### Port

PostgreSQL:

```text
5432
```

MySQL:

```text
3306
```

### Endpoint

Make sure the application is using the correct endpoint.

### Database status

Confirm the RDS instance/cluster is in an appropriate state such as:

```text
Available
```

### Connectivity settings

Review relevant configuration such as:

- Public/private accessibility
- DB subnet group
- Parameter group
- Cluster/instance configuration
- SSL/TLS requirements

---

# 7. Database Authentication

If TCP connectivity works, the next question is authentication.

For PostgreSQL:

```bash
psql -h <rds-endpoint> -U <username> -d <database>
```

For MySQL:

```bash
mysql -h <rds-endpoint> -u <username> -p
```

At this point, errors such as:

```text
password authentication failed
```

or:

```text
Access denied
```

indicate a different problem from a network timeout.

The network path may already be working.

Now investigate:

- Username
- Password
- Database name
- Authentication method
- SSL/TLS
- IAM database authentication if configured
- Application secret/configuration

---

# 🐘 PostgreSQL Troubleshooting

PostgreSQL commonly uses:

```text
TCP 5432
```

Basic connectivity test:

```bash
nc -zv <rds-endpoint> 5432
```

Database connection:

```bash
psql \
  -h <rds-endpoint> \
  -p 5432 \
  -U <username> \
  -d <database>
```

Check DNS:

```bash
dig +short <rds-endpoint>
```

### PostgreSQL troubleshooting sequence

```text
DNS
 ↓
TCP 5432
 ↓
Security Group
 ↓
Route
 ↓
NACL
 ↓
PostgreSQL
 ↓
Username
 ↓
Password
 ↓
Database
```

---

# 🐬 MySQL Troubleshooting

MySQL commonly uses:

```text
TCP 3306
```

Test the port:

```bash
nc -zv <rds-endpoint> 3306
```

Connect:

```bash
mysql \
  -h <rds-endpoint> \
  -P 3306 \
  -u <username> \
  -p
```

The same layered troubleshooting approach applies:

```text
DNS
 ↓
TCP 3306
 ↓
Security Group
 ↓
Route
 ↓
NACL
 ↓
MySQL
 ↓
Authentication
 ↓
Database
```

---

# ⚠️ Common Failure Scenarios

## Scenario 1 — DNS does not resolve

```text
Application
    |
    X
RDS endpoint
```

### Investigate

- VPC DNS settings
- Resolver
- DNS configuration
- Endpoint spelling

---

## Scenario 2 — DNS works but TCP times out

```text
DNS       ✓
TCP       ✗
```

### Investigate

```text
Security Group
Route Table
Network ACL
Subnet
Network path
```

---

## Scenario 3 — TCP works but login fails

```text
DNS       ✓
TCP       ✓
Login     ✗
```

### Investigate

```text
Username
Password
Database
SSL/TLS
Authentication configuration
```

---

## Scenario 4 — Wrong port

For example:

```text
Application → 5432
RDS → MySQL 3306
```

The application is trying to reach the wrong service.

Verify the actual engine and port.

---

## Scenario 5 — RDS is private

A private RDS instance should not normally be accessed directly from the public internet.

Expected architecture:

```text
Application
     |
     v
Private network
     |
     v
RDS
```

If an engineer is attempting:

```text
Laptop
   |
   v
Internet
   |
   v
Private RDS
```

the network architecture itself needs to be addressed.

---

# 🧪 Useful Commands

## DNS

```bash
nslookup <rds-endpoint>
```

```bash
dig <rds-endpoint>
```

```bash
dig +short <rds-endpoint>
```

## TCP connectivity

```bash
nc -zv <rds-endpoint> 5432
```

```bash
nc -zv <rds-endpoint> 3306
```

## PostgreSQL

```bash
psql -h <rds-endpoint> -U <username> -d <database>
```

## MySQL

```bash
mysql -h <rds-endpoint> -u <username> -p
```

## Basic network information

```bash
ip addr
```

```bash
ip route
```

```bash
cat /etc/resolv.conf
```

These commands help establish:

```text
Who am I?
Where am I?
How do I route traffic?
Which DNS resolver am I using?
Can I reach the destination?
```

---

# 🔐 Security Best Practices

## 1. Do not expose RDS unnecessarily

Avoid:

```text
0.0.0.0/0
```

for database access unless there is a very specific, controlled reason.

## 2. Prefer Security Group references

Use:

```text
Application SG → RDS SG
```

instead of broad IP ranges where possible.

## 3. Keep databases private

For typical application architectures:

```text
Internet
   |
   v
ALB
   |
   v
Application
   |
   v
Private RDS
```

## 4. Use encryption

Consider:

- Encryption at rest
- TLS in transit
- Secrets Manager
- IAM authentication where appropriate

## 5. Do not put passwords in commands or source code

Avoid committing:

```text
DB_PASSWORD=...
```

to Git repositories.

Use an appropriate secrets-management mechanism.

---

# ✅ Troubleshooting Checklist

```text
[ ] Confirm the exact error message
[ ] Confirm the application host
[ ] Confirm the RDS endpoint
[ ] Confirm the database engine
[ ] Confirm the database port
[ ] Test DNS resolution
[ ] Test TCP connectivity
[ ] Check application Security Group
[ ] Check RDS Security Group
[ ] Verify RDS inbound rule
[ ] Verify route tables
[ ] Check Network ACLs
[ ] Verify subnet configuration
[ ] Confirm RDS status
[ ] Verify RDS endpoint
[ ] Test database login
[ ] Verify username
[ ] Verify password
[ ] Verify database name
[ ] Check SSL/TLS requirements
[ ] Check application configuration
[ ] Check recent network changes
[ ] Validate the fix
[ ] Document the root cause
```

---

# 🧠 Key Takeaways

### 1. RDS availability is not application connectivity

An RDS instance can be:

```text
Available
```

while the application cannot reach it.

### 2. Troubleshoot from the bottom up

Use:

```text
DNS
 ↓
Network
 ↓
Security Group
 ↓
Route
 ↓
NACL
 ↓
Database port
 ↓
Database
 ↓
Authentication
```

### 3. Do not change everything at once

Random changes make troubleshooting harder because you lose the ability to determine which change actually fixed the issue.

### 4. Security Groups are only one layer

A correct Security Group does not guarantee connectivity if another networking layer is broken.

### 5. Connection errors tell you where to look

For example:

```text
DNS error
```

means investigate DNS.

```text
Connection timeout
```

usually means investigate network reachability.

```text
Authentication failure
```

means the network path may already be working.

---

# 🎯 Final Thoughts

RDS connectivity troubleshooting becomes much easier when the problem is treated as a layered networking problem rather than simply a database problem.

Instead of asking:

> **"Why can't my application connect to RDS?"**

ask:

> **"At which exact layer does the connection fail?"**

Use the following flow:

```text
        DNS
         ↓
      Network
         ↓
  Security Groups
         ↓
    Route Tables
         ↓
    Network ACLs
         ↓
    Database Port
         ↓
      RDS
         ↓
 Authentication
```

The objective is simple:

**Find the failing layer → fix the real problem → validate → prevent recurrence.**

---

## 📚 Related Guides

This guide is part of the **AWS Production Troubleshooting** repository.

Planned troubleshooting scenarios:

- [AWS ALB 504 Gateway Timeout](../alb/alb-504-gateway-timeout.md)
- EC2 application unresponsiveness
- Auto Scaling configuration drift
- CloudWatch investigation
- AWS Security Group troubleshooting
- EKS troubleshooting
- ECR → EKS deployment troubleshooting
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

> ⭐ **Document. Share. Help others. That's how we grow the AWS community.**
