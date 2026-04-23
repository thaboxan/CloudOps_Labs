# Lab 1: Auditing AWS Resources with Systems Manager & AWS Config

## Overview

This lab focuses on auditing, compliance, and visibility of AWS resources using:

- AWS Systems Manager
- AWS Config

---

## Objectives

- Setup Systems Manager Inventory
- Connect to EC2 using Session Manager
- Enable AWS Config
- Create compliance rules
- Audit infrastructure
- Explore instance metadata

---

## Architecture Summary

### Systems Manager
- Manage EC2 instances
- Remote access (no SSH)
- Collect inventory (metadata, apps)

### AWS Config
- Tracks resource configurations
- Evaluates compliance using rules

---

## Task 1: Setup Systems Manager Inventory

### Steps

1. Go to **Systems Manager → Inventory**
2. Choose: **Manual selection**
3. Select: **Web Server**, **App Server**
4. Enable Inventory

### Notes

- Inventory collects metadata only
- Includes: installed applications, instance details
- Works across regions
- Requires: SSM Agent, IAM roles

---

## Task 2: Connect to EC2 via Session Manager

### Steps

1. Go to **Fleet Manager**
2. Select instance (**Web Server**)
3. **Connect → Start terminal session**

### Commands

Check open connections:

```bash
netstat -a | grep http
```

Get EC2 info via CLI:

```bash
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`

AZ=`curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone`

export AWS_DEFAULT_REGION=${AZ::-1}

aws ec2 describe-instances
```

### Notes

- No SSH required — no port 22 needed
- Benefits:
  - More secure
  - No key management
  - Auditing via CloudTrail

---

## Task 3: Enable AWS Config

### Settings

| Setting | Value |
|---|---|
| Recording strategy | All resource types |
| Frequency | Continuous |
| IAM role | AWSServiceRoleForConfig |

### Notes

- Tracks configuration changes
- Stores history
- Used for compliance auditing

### Ensure Recorder is ON

Go to **Config → Settings → Recorder**

If OFF:
1. Enable recording
2. Use service-linked role

---

## Task 4: Create Config Rule — EC2 Compliance

**Rule:** `ec2-instance-managed-by-systems-manager`

### Purpose

Checks if EC2 instances are managed by SSM and reachable.

### Notes

Non-compliant if:
- No SSM agent installed
- Instance not connected to SSM

---

## Task 5: IAM Compliance Rule

**Rule:** `iam-user-no-policies-check`

### Purpose

Detect IAM users with inline policies attached.

### Notes

- Best practice: use roles instead of inline policies
- This is audit only (no remediation)

---

## Task 6: Explore Systems Manager Inventory

### Dashboard Insights
- Top installed applications
- Managed instances

### Instance Details (Fleet Manager)
- IP address
- Instance type
- SSH key
- Tags

### Inventory Checks
- Installed packages (e.g. `bind-utils`)
- Version + architecture

### Associations
- Last SSM document run
- Document name

### Patches
- Applied patches
- Errors (if any)
- Pending updates

### Compliance
- Patch compliance status

### Notes

Inventory gives deep visibility — useful for audits, troubleshooting, and compliance checks.

---

## Key Concepts

### Systems Manager
- Centralized management
- Remote access
- Inventory collection

### AWS Config
- Tracks changes over time
- Evaluates compliance
- Uses rules

### Compliance Model

```
Resource → Rule → Evaluation → COMPLIANT / NON_COMPLIANT
```

### Inventory vs Config

| Feature | Systems Manager Inventory | AWS Config |
|---|---|---|
| Purpose | Metadata collection | Compliance auditing |
| Data | Apps, OS, packages | Resource configs |
| Use case | Visibility | Governance |

---

## Summary

- Enabled Systems Manager Inventory
- Connected to EC2 without SSH
- Enabled AWS Config
- Created compliance rules
- Audited EC2 + IAM resources
- Explored instance metadata and inventory
