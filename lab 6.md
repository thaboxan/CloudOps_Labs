# Lab 6: Capstone Lab for CloudOps

## Overview

This capstone lab brings together the tools and techniques from all previous labs into three integrated scenarios. Each part builds on the last, combining **EventBridge**, **SNS**, **SSM Documents**, **CloudFormation**, and **AWS Config** to create fully automated monitoring, detection, and remediation pipelines.

---

## Objectives

- Use **Amazon EventBridge** and **Amazon SNS** to build an automated notification system for SSM document completions
- Use **AWS CloudFormation drift detection** to discover unauthorized infrastructure changes
- Use **AWS Config Rules** with **SSM Documents** to auto-remediate non-compliant S3 buckets

---

## Architecture Overview

```
Part 1: Notification Mechanism
  SSM Document completes → EventBridge Rule → SNS Topic → Email

Part 2: Drift Detection + Remediation
  Security Group changed manually → CloudFormation Drift Detected
  → SSM Document (AWS-DisablePublicAccessForSecurityGroup) → Remediated
  → EventBridge → SNS → Email notification

Part 3: Config Rule + Auto-Remediation
  S3 bucket versioning disabled → Config Rule detects non-compliance
  → SSM Document (AWS-ConfigureS3BucketVersioning) → Versioning enabled
  → EventBridge → SNS → Email notification
```

---

## Part 1: Automated Notifications from SSM Documents

### Task 1.1: Create an SNS Topic

1. Navigate to **Simple Notification Service → Topics**
2. Choose **Create topic**
3. Select type: **Standard**
4. Enter name: `SuccessfulAutomationAction`
5. Choose **Create topic**

### Task 1.2: Subscribe Your Email

1. On the topic page, go to the **Subscriptions** tab
2. Choose **Create subscription**
3. Protocol: **Email**
4. Endpoint: enter your email address
5. Choose **Create subscription**
6. Open your inbox and click the **Confirm subscription** link

> **Note:** It may take up to 5 minutes to receive the confirmation email.

### Notes

- SNS topics support multiple subscribers simultaneously (email, SMS, Lambda, SQS, HTTP)
- In a real environment, this topic would be subscribed to by the whole CloudOps team
- Always confirm subscriptions before testing — unconfirmed endpoints receive nothing

---

### Task 2.1: Create an EventBridge Rule

This rule fires whenever an SSM automation document completes with status `Success`.

1. Navigate to **Amazon EventBridge → Rules**
2. Choose **Create rule**
3. Enter name: `AutomationSuccess`
4. Choose **Next**
5. Configure the **Event pattern**:
   - Event source: **AWS services**
   - AWS service: **Systems Manager**
   - Event type: **Automation**
   - Detail type: **EC2 Automation Execution Status-change Notification**
   - Status: **Success**
6. Choose **Next**
7. Target: **SNS topic → SuccessfulAutomationAction**
8. Under Permissions: **uncheck** Use execution role
9. Choose **Next → Next → Create rule**

### Notes

> **How EventBridge works:** EventBridge continuously monitors the AWS event stream. When a matching event occurs (in this case: SSM automation reaches `Success`), EventBridge routes it to the configured target (SNS topic) within milliseconds.

**Event pattern (what it looks like internally):**
```json
{
  "source": ["aws.ssm"],
  "detail-type": ["EC2 Automation Execution Status-change Notification"],
  "detail": {
    "Status": ["Success"]
  }
}
```

---

### Task 2.2: Test the Notification System

1. Navigate to **Systems Manager → Documents → Owned by Amazon**
2. Search for and open `AWS-ConfigureCloudWatchOnEC2Instance`
3. Choose **Execute automation → Simple execution**
4. Paste the **InstanceId** from the lab panel into Input Parameters
5. Choose **Execute**
6. Wait for **Overall status: Success**
7. Check your email — SNS delivers a JSON payload detailing the execution ✅

---

## Part 2: CloudFormation Drift Detection and Remediation

**The scenario:** Someone manually added an inbound SSH rule (`port 22`) to a security group, making it publicly accessible. You need to detect and revert this change.

### Task 3.1: Run Drift Detection

1. Navigate to **CloudFormation**
2. Select the **Lab 6 Capstone** stack
3. Choose **Stack actions → Detect drift**
4. Wait up to 1 minute for the operation to complete
5. Choose **Stack actions → View drift results**

### Task 3.2: Identify the Drifted Resource

1. In **Resource drift status**, find the resource with `DRIFTED` status
2. Select the radio button next to **AppSecurityGroup**
3. Choose **View drift details**
4. In the **Differences** section, select **SecurityGroupIngress**
5. Review the code diff — you'll see the unauthorized inbound rule ⚠️
6. **Copy the Physical ID** of the security group (format: `sg-...`) — you need it next

> **Key insight:** Drift detection shows you the exact JSON diff between the template definition and the current configuration, making it easy to identify exactly what changed and who may have done it (check CloudTrail for the API call).

---

### Task 4.1: Remediate with SSM Document

1. Navigate to **Systems Manager → Documents → Owned by Amazon**
2. Search for `AWS-DisablePublicAccessForSecurityGroup`
3. Review the **Content** tab — the document removes public inbound rules for SSH (22), RDP (3389), and other common ports
4. Choose **Execute automation → Simple execution**
5. In **GroupId**, paste the security group ID you copied
6. Choose **Execute**
7. Wait for **Overall status: Success**

> **Note:** Some individual steps may show as failed — this is expected. The document handles multiple port types (SSH, RDP, etc.), but only the SSH rule exists in this environment. Steps for non-existent rules fail gracefully, but the overall execution succeeds.

### Task 4.2: Verify with Drift Detection

1. Back in **CloudFormation**, select the stack
2. Choose **Stack actions → Detect drift**
3. Wait up to 1 minute
4. View drift results — **Drift status should now show `IN_SYNC`** ✅

> Check your email — the SSM document completion triggered the EventBridge rule, and SNS sent a notification.

---

## Part 3: AWS Config Rules and Auto-Remediation

**The scenario:** An S3 bucket does not have versioning enabled, violating company policy. You'll configure AWS Config to detect this and automatically remediate it using an SSM document.

### Task 5.1: Set Up AWS Config

1. Navigate to **AWS Config → Get started**
2. Configure **Recording**:
   - Recording strategy: **All resource types with customizable overrides**
   - Recording frequency: **Continuous**
3. Configure **Data governance**:
   - IAM role: **AWSServiceRoleForConfig** (choose from existing roles)
4. Configure **Delivery channel**:
   - S3 bucket: **Create a bucket**
5. Choose **Next**

### Task 5.2: Add the S3 Versioning Rule

1. On the **Rules** page, find and check `S3-bucket-versioning-enabled`
2. Choose **Next → Confirm**
3. After setup, go to **Rules → s3-bucket-versioning-enabled**
4. Set scope to **All** resources
5. Wait for evaluation — the bucket with `labbucket` in its name will show as **Non-compliant** ⚠️

> **Note:** Config evaluation can take up to 5 minutes after initial setup. If you see "No resources in scope", wait and refresh.

---

### Task 6: Configure Auto-Remediation

1. Navigate to the `s3-bucket-versioning-enabled` rule
2. Choose **Actions → Manage remediation**
3. Select **Automatic remediation**
4. Remediation action: `AWS-ConfigureS3BucketVersioning`
5. In **Resource ID parameter**: select `BucketName`
6. In **Parameters**:
   - `VersioningState`: `Enabled`
   - `AutomationAssumeRole`: paste the `LabConfigRoleARN` from the lab panel
7. Choose **Save changes**

**What happens automatically:**
1. Config detects the non-compliant S3 bucket
2. Config triggers the SSM document `AWS-ConfigureS3BucketVersioning`
3. SSM enables versioning on the bucket
4. EventBridge detects SSM document `Success`
5. SNS delivers email notification ✅

> **Optional manual trigger:** Select the non-compliant bucket → choose **Remediate** to trigger immediately rather than waiting for the next auto-remediation cycle.

---

## Key Concepts

### EventBridge as the Glue

EventBridge is a serverless event bus. In this lab it connects:
- **SSM document completion** → SNS notifications
- Any AWS service that emits events can be a source
- Any AWS service that accepts invocations can be a target

### Config Rule Lifecycle

```
Resource Created/Changed
    → Config evaluates against rules
    → Non-compliant? → Trigger remediation action (SSM document)
    → SSM completes → EventBridge → SNS → Email
```

### Drift Detection vs Config Rules

| Feature | CloudFormation Drift Detection | AWS Config Rules |
|---------|-------------------------------|-----------------|
| Scope | Resources in a stack | Any AWS resource |
| Trigger | Manual (or automated via EventBridge) | Continuous or periodic |
| Remediation | Manual (run SSM doc) | Automatic |
| Best for | Detecting unauthorized IaC changes | Ongoing compliance enforcement |

### The Three Automation Pillars

| Tool | Role |
|------|------|
| **AWS Config** | Detect non-compliance |
| **SSM Documents** | Remediate (fix) non-compliance |
| **EventBridge + SNS** | Notify the team when remediation occurs |

---

## Summary

- ✅ Created `SuccessfulAutomationAction` SNS topic and subscribed via email
- ✅ Created `AutomationSuccess` EventBridge rule to notify on SSM completion
- ✅ Tested the notification pipeline with `AWS-ConfigureCloudWatchOnEC2Instance`
- ✅ Used CloudFormation drift detection to find an unauthorized security group rule
- ✅ Remediated the drifted security group using `AWS-DisablePublicAccessForSecurityGroup`
- ✅ Configured AWS Config rule `s3-bucket-versioning-enabled` with auto-remediation
- ✅ Auto-remediation enabled versioning on the non-compliant S3 bucket via `AWS-ConfigureS3BucketVersioning`
