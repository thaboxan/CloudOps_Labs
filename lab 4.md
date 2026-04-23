# Lab 4: Monitoring Applications & Infrastructure

## Overview

This lab focuses on building a monitoring solution for an EC2 application server using AWS services.

**Main components:**
- Amazon CloudWatch
- AWS Systems Manager
- Amazon Simple Notification Service
- AWS Lambda
- Amazon EventBridge

## Objectives

- Configure telemetry on EC2 using CloudWatch Agent
- Create dashboards for metrics visualization
- Set up notifications using SNS
- Create and test CloudWatch alarms
- Implement Lambda Canary monitoring

## Task 1: Install CloudWatch Agent

### 1.1 Update SSM Agent

- Use SSM document: `AWS-UpdateSSMAgent`
- Target: EC2 instance (AppServer)
- Purpose: Ensure latest version

### 1.2 Install CloudWatch Agent

- Use SSM document: `AWS-ConfigureAWSPackage`
- Package: `AmazonCloudWatchAgent`
- Target: AppServer

### Notes

- SSM allows remote management without SSH
- Requires IAM roles + network access (ports 80/443 outbound to SSM endpoints)
- CloudWatch Agent collects **OS-level metrics** not available natively:
  - Memory utilization (AWS doesn't expose this by default)
  - Disk space usage per mount point
  - Custom application logs
- The agent is separate from the SSM Agent — both must be installed and running

## Task 2: Configure & Start CloudWatch Agent

### 2.1 Configuration File

- Stored in SSM Parameter Store
- Name: `AgentConfigFile`
- Format: JSON
- Defines:
  - Metrics to collect
  - Logs to monitor

### 2.2 Start Agent

- Use SSM document: `AmazonCloudWatch-ManageAgent`
- Parameters:
  - Action: configure
  - Source: ssm
  - Config location: AgentConfigFile
  - Restart: yes

### 2.3 Check Agent Status

Use SSM document:
- Action: status

OR manually:

```bash
amazon-cloudwatch-agent-ctl -a status
```

Expected output:

```json
{
  "status": "running"
}
```

### 2.4 Verify Metrics

- Go to CloudWatch → Metrics
- Check namespace: `CWAgent`

### Notes

- Install ≠ Configure ≠ Start — three separate steps, each can fail independently
- Parameter Store enables:
  - **Reuse**: same config file pushed to many instances via SSM
  - **Versioning**: roll back to a previous config if the new one causes issues
  - **Centralized config**: update in one place, propagates everywhere

> **Namespace note:** CloudWatch Agent metrics appear under the `CWAgent` namespace, separate from the built-in EC2 metrics under `AWS/EC2`. When building dashboards, you'll need to combine both namespaces.

## Task 3: Create CloudWatch Dashboard

**Dashboard:** `myDashboard`

**Widget 1:**
- Type: Line graph
- Metric: `mem_used_percent` (CWAgent)

**Widget 2:**
- Type: Stacked area
- Metrics:
  - NetworkIn
  - NetworkOut

### Notes

- Dashboards provide customized monitoring view
- Combine:
  - Agent metrics
  - Native AWS metrics

## Task 4: SNS Subscription

**Setup:**
- Topic: `SysOpsTeamPager`
- Protocol: Email
- Confirm via email link

### Notes

- SNS enables alert notifications across multiple channels simultaneously
- Multiple alarms can share the same topic — one topic can notify the entire team
- Always **confirm the subscription** before testing alarms — unconfirmed subscriptions receive nothing
- In production, use a team email alias or PagerDuty/Slack integration as the endpoint

## Task 5: Create CloudWatch Alarm

**Alarm Details:**
- Name: `AppServerSystemsCheckAlarm`
- Metric: `StatusCheckFailed_System`
- Threshold: ≥ 1
  - Send notification to SNS topic

### Notes

- `StatusCheckFailed_System` detects hardware/hypervisor failures on the underlying AWS host
- Use `StatusCheckFailed_Instance` for OS-level failures within your instance
- Good alarms always include:
  - A clear description (what broke)
  - A runbook link (how to fix it)
  - Remediation steps in the alarm description
- Set the evaluation period carefully — 1 data point in 1 minute is fine for critical system checks

## Task 6: Test Alarm Using CLI

**Start Session:**
- Use Systems Manager → Session Manager

**Trigger Alarm:**

```bash
aws cloudwatch set-alarm-state \
--alarm-name "AppServerSystemsCheckAlarm" \
--state-value ALARM \
--state-reason "testing purposes"
```

**Reset Alarm:**

```bash
--state-value OK
```

### Notes

- Manually setting alarm state is much faster than waiting for a real failure in a lab/test environment
- Use this technique in CI/CD pipelines to validate that alarm → SNS → Lambda workflows are wired correctly before going to production
- Always reset to `OK` after testing so the alarm doesn't stay in `ALARM` and suppress real future alerts

## Task 7: Lambda Canary Monitoring

### 7.1 Create Function

- Name: `lambda-canary`
- Blueprint: URL check

**Trigger:**
- EventBridge rule: `rate(1 minute)`

**Environment Variables:**
- `site` → target URL
- `expected` → expected string

### 7.2 Test Function

- Use built-in test event
- Check logs/output

### 7.3 Create Alarm

- Metric: Lambda Errors
- Threshold: ≥ 1
- Notification: SNS

### 7.4 Test Failure

- Change expected value (e.g. → 404)
- Trigger error
- Alarm fires → email sent

### Notes

- **Canary = synthetic monitoring** — simulates a real user request from outside your infrastructure
- Detects:
  - Complete downtime (connection refused / timeout)
  - Wrong content (site returns 200 but shows an error page)
  - Regression bugs (expected string no longer present after a deployment)
- EventBridge rate expressions: `rate(1 minute)`, `rate(5 minutes)`, `cron(0 8 * * ? *)` (8 AM daily)
- Lambda canaries are much cheaper than AWS Synthetics Canaries but less feature-rich

## Key Concepts

### Monitoring Components

| Component | What it measures | Example |
|-----------|-----------------|---------|
| **Metrics** | Numeric time-series data | CPU %, memory %, error count |
| **Logs** | Text event records | Apache access log, app errors |
| **Events** | State change notifications | Instance stopped, deployment finished |
| **Alarms** | Threshold breach detection | CPU > 80% for 5 minutes |
| **Notifications** | Human/system alerts | Email via SNS, PagerDuty |

### CloudWatch Agent

- Provides OS-level metrics AWS cannot see from outside the instance
- Required for:
  - Memory utilization (`mem_used_percent`)
  - Disk usage (`disk_used_percent`)
  - Custom application metrics
- Agent config is stored in Parameter Store for centralized management

### Parameter Store

- Stores config centrally — one place to update, all instances pick it up
- Supports versioning — roll back if a config causes issues
- Hierarchical naming (e.g. `/app/prod/cloudwatch-config`) for organization

### Alarm Flow

```
EC2 Metric → CloudWatch Alarm → ALARM state → SNS Topic → Email/SMS/Lambda
```

### Event-driven Monitoring

```
EventBridge rule (rate: 1 min) → Lambda Canary → HTTP check
                                      ↓ (on error)
                               CloudWatch Error metric
                                      ↓
                               CloudWatch Alarm → SNS → Email
```

## Final Summary

- ✅ Installed and configured CloudWatch Agent using SSM
- ✅ Created dashboards for visualization
- ✅ Configured SNS notifications
- ✅ Created and tested CloudWatch alarms
- ✅ Implemented Lambda Canary for website monitoring