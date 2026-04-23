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
- Requires IAM roles + network access (ports 80/443)
- CloudWatch Agent collects:
  - System metrics (CPU, memory, disk)
  - Logs

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

- Install ≠ Configure ≠ Start
- Parameter Store enables:
  - Reuse
  - Versioning
  - Centralized config

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

- SNS enables alert notifications
- Multiple alarms can use same topic

## Task 5: Create CloudWatch Alarm

**Alarm Details:**
- Name: `AppServerSystemsCheckAlarm`
- Metric: `StatusCheckFailed_System`
- Threshold: ≥ 1
  - Send notification to SNS topic

### Notes

- Detects EC2 system failures
- Include:
  - Description
  - Runbook link
  - Remediation steps

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

- Faster than waiting for real failure
- Useful for testing:
  - SNS
  - Lambda
  - Alerts

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

- Canary = synthetic monitoring
- Simulates real user request
- Detects:
  - downtime
  - incorrect responses

## Key Concepts

### Monitoring Components

- Metrics
- Logs
- Events
- Alarms
- Notifications

### CloudWatch Agent

- Provides OS-level metrics
- Needed for:
  - Memory
  - Disk

### Parameter Store

- Stores config centrally
- Supports:
  - Versioning
  - Reusability

### Alarm Flow

Metric → Threshold → Alarm → SNS → Notification

### Event-driven Monitoring

EventBridge → Lambda → CloudWatch → SNS

## Final Summary

- ✅ Installed and configured CloudWatch Agent using SSM
- ✅ Created dashboards for visualization
- ✅ Configured SNS notifications
- ✅ Created and tested CloudWatch alarms
- ✅ Implemented Lambda Canary for website monitoring