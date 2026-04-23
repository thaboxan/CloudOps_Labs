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

```
EC2 Instances
    └── SSM Agent (pre-installed on Amazon Linux 2)
            ├── Systems Manager Inventory → metadata, packages, patches
            ├── Fleet Manager → instance details, connections
            └── Session Manager → browser-based terminal (no SSH needed)

AWS Config
    └── Configuration Recorder (continuous)
            └── Config Rules
                    ├── ec2-instance-managed-by-systems-manager
                    └── iam-user-no-policies-check
```

### Systems Manager
- Manage EC2 instances without opening port 22
- Remote access via browser-based Session Manager
- Collect inventory: installed apps, OS info, patch state

### AWS Config
- Tracks every configuration change over time
- Evaluates resources against compliance rules
- Stores a full configuration history in S3

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
  - More secure — no inbound firewall rules needed
  - No key pair management — no `.pem` files to rotate or lose
  - Full audit trail — all session activity logged to CloudTrail and S3
  - Works for instances in private subnets with no public IP

> **How it works:** Session Manager communicates outbound over HTTPS (port 443) via the SSM endpoint. The instance needs internet access (or a VPC endpoint) and the `AmazonSSMManagedInstanceCore` IAM policy.

---

## Task 3: Enable AWS Config

### Settings

| Setting | Value |
|---|---|
| Recording strategy | All resource types |
| Frequency | Continuous |
| IAM role | AWSServiceRoleForConfig |

### Notes

- Tracks configuration changes and stores the full history
- You can query "what did this resource look like on [date]?"
- Used for compliance auditing and incident investigation
- The service-linked role (`AWSServiceRoleForConfig`) grants Config read access to all supported resource types

> **Important:** Make sure the recorder is ON. Config won't detect anything while paused.

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
- No SSM agent installed or outdated
- Instance not connected to SSM (check: Fleet Manager shows "Connection status: Offline")
- Missing required IAM permissions (`AmazonSSMManagedInstanceCore`)

> **Tip:** Use this rule as a quick health check — any non-compliant EC2 instance can't be managed remotely via SSM.

---

## Task 5: IAM Compliance Rule

**Rule:** `iam-user-no-policies-check`

### Purpose

Detect IAM users with inline policies attached.

### Notes

- Best practice: **never** attach inline policies directly to users — use groups and roles instead
- Inline policies are harder to audit because they don't appear in the global policies list
- This rule is audit-only (no auto-remediation) — a human must review and fix violations
- In production, also add `iam-password-policy` and `iam-root-access-key-check` rules

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

> **Pro tip:** You can query inventory data using the SSM Inventory Query feature or export it to Athena via S3 for SQL-based analysis across your entire fleet.

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
