# Lab 5: Automating with AWS Backup for Archiving and Recovery

## Overview

As a CloudOps engineer, a key responsibility is backing up and restoring data from block storage. Archives and backups are critical for business continuity and disaster recovery (DR).

This lab covers using **AWS Backup** to:
- Create a backup plan and vault
- Automate snapshots of an Amazon EBS volume
- Restore an EBS volume from a snapshot
- Push backup job notifications through **Amazon SNS**
- Validate backups using **AWS Lambda**

---

## Objectives

- Subscribe to an existing Amazon SNS topic
- Create a backup plan in AWS Backup
- Create a backup vault in AWS Backup
- Create an on-demand backup job of an Amazon EBS volume
- Test a backup using a Lambda function to automatically start a restore job
- Examine AWS CloudWatch logs

---

## Architecture Summary

```
EBS Volume
    └── AWS Backup (scheduled/on-demand)
            └── Backup Vault (snapshot stored)
                    └── SNS Topic (BACKUP_JOB_COMPLETED / RESTORE_JOB_COMPLETED)
                            ├── Email (subscribed CloudOps engineer)
                            └── Lambda Function
                                    ├── Starts Restore Job (validates backup)
                                    ├── Deletes restored volume (cleanup)
                                    └── Sends SNS notification
```

---

## Task 1: Subscribe to an AWS SNS Topic

1. In the AWS Console, search for and open **Simple Notification Service**
2. Go to **Topics** in the navigation bar
3. Select **BackupNotificationTopic**
4. Choose **Create subscription**
5. Set **Protocol** to `Email`
6. Enter a valid email address in the **Endpoint** field
7. Choose **Create subscription**
8. Open your email inbox and confirm the subscription via the link from `no-reply@sns.amazonaws.com`

> **Note:** It may take up to 5 minutes to receive the confirmation email.

---

## Task 2: Create a Backup Plan

1. Search for and open **AWS Backup**
2. Select **Dashboard → Create backup plan**
3. Choose **Build a new plan**

### Backup Plan Settings

| Field | Value |
|---|---|
| Backup plan name | `myBackupPlan` |
| Tag Key | `Name` |
| Tag Value | `myApp Backup` |

### Backup Rule Configuration

| Field | Value |
|---|---|
| Rule name | `myDailyBackupRule` |
| Frequency | Daily (default) |

### Create Backup Vault

1. Click **Create new vault** in the Backup rule configuration section
2. Configure the vault:

| Field | Value |
|---|---|
| Vault name | `myDailyBackupVault` |
| Vault type | Backup vault |
| KMS encryption key | `(default) aws/backup` |

3. Click **Create vault**, then return to the backup plan tab
4. Refresh and select `myDailyBackupVault` from the dropdown

### Recovery Point Tags

| Key | Value |
|---|---|
| `Name` | `myApp Recovery` |

5. Choose **Create plan**

---

## Task 3: Assign Resources to Backup

### Task 3.1: Assign Resources

On the **Assign resources** page, configure the following:

| Field | Value |
|---|---|
| Resource assignment name | `myEBSVolumes` |
| IAM role | `BackupRole` |
| Resource type | EBS |
| Volume | `webAppVolume` |

**Refine selection using tags:**

| Key | Condition | Value |
|---|---|---|
| `Name` | Equals | `webAppVolume` |

Click **Assign resources**.

---

### Task 3.2: Subscribe AWS Backup to the SNS Topic

Use the AWS CLI (via the Code Editor IDE) to configure vault notifications.

**Connect:** Open the `LabInstanceURL` in a new browser tab.

**Configure notifications:**

```bash
aws backup put-backup-vault-notifications \
  --endpoint-url https://backup.<LabRegion>.amazonaws.com \
  --sns-topic-arn <SNSTopicARN> \
  --backup-vault-name myDailyBackupVault \
  --backup-vault-events RESTORE_JOB_COMPLETED BACKUP_JOB_COMPLETED
```

> Replace `<LabRegion>` and `<SNSTopicARN>` with the values from the lab panel.

**Verify notifications were configured:**

```bash
aws backup get-backup-vault-notifications \
  --backup-vault-name myDailyBackupVault
```

Expected output:

```json
{
    "BackupVaultName": "myDailyBackupVault",
    "BackupVaultArn": "arn:aws:backup:us-west-2:123456789:backup-vault:myDailyBackupVault",
    "SNSTopicArn": "arn:aws:sns:us-west-2:123456789:BackupNotificationTopic",
    "BackupVaultEvents": [
        "RESTORE_JOB_COMPLETED",
        "BACKUP_JOB_COMPLETED"
    ]
}
```

> **Note:** Backup vault notifications can only be configured via the AWS CLI or SDK — not the AWS Console.

---

## Task 4: On-Demand Backup of the Amazon EBS Volume

1. Open **AWS Backup → Protected resources**
2. Choose **Create on-demand backup**

### Settings

| Field | Value |
|---|---|
| Resource type | EBS |
| Volume ID | `webAppVolume` |
| Total retention period | 1 Day |
| Backup vault | `myDailyBackupVault` |
| IAM role | `BackupRole` |

3. Choose **Create on-demand backup**

> **Warning:** The initial EBS snapshot can take up to 10 minutes. You can continue to Task 5 while it runs.

---

## Task 5: Examine the Lambda Function

1. Search for and open **Lambda**
2. Select **RestoreTestFunction**
3. Open **LambdaCode.py** in the Code Source section

### What the Function Does

| Incoming Event | Action |
|---|---|
| `BACKUP_JOB_COMPLETED` | Retrieves backup details, starts a Restore job, sends SNS notification |
| `RESTORE_JOB_COMPLETED` | Deletes the restored volume (cleanup), sends SNS notification |

This architecture demonstrates an automated backup validation pattern:

```
Backup completes → SNS → Lambda → Restore job starts
Restore completes → SNS → Lambda → Restored volume deleted + notification sent
```

---

## Task 6: Test That the Backups Can Be Restored

1. Open **AWS Backup → Jobs → Backup jobs**
2. Wait for the backup job status to change to **Completed**
3. Open the **Restore Jobs** tab
4. Wait for the restore job status to change from **Running** to **Completed**
5. Note the EBS Volume ID (`vol-...`) in the Resource ID column

### Email Notifications to Expect

| Email Subject | Meaning |
|---|---|
| `An AWS Backup Restore job completed successfully` | Restore validated — contains restored volume ARN |
| `Restore from ARN was successful` | Cleanup complete — restored volume has been deleted |

> **Warning:** Both the backup and restore jobs can each take up to 10 minutes to complete.

---

## Task 7: Review CloudWatch Logs

1. Search for and open **CloudWatch**
2. Go to **Logs → Log Groups**
3. Select `/aws/lambda/RestoreTestFunction`
4. Open a log stream and review the entries

### What to Look For

- Entries starting with `Incoming Event: {"Records"}` — shows the event the Lambda received
- The entry immediately after reveals which action was triggered (restore or delete)
- Two log streams should exist: one for the backup job event, one for the restore job event

---

## Key Concepts

### AWS Backup
- Centralized backup service for AWS resources
- Supports scheduled (plan-based) and on-demand backups
- Stores backups in encrypted vaults

### Backup Vault
- Secure, organized storage for backups (recovery points)
- Can emit events to SNS on job completion
- Vault notifications only configurable via CLI/SDK

### Backup Validation Pattern

```
Backup Job → SNS → Lambda → Restore Job → SNS → Lambda → Cleanup
```

### SNS + Lambda Integration
- SNS fans out notifications to multiple subscribers simultaneously
- Lambda automates restore testing without manual intervention
- This pattern (backup → SNS → Lambda → restore → SNS → Lambda → cleanup) is a fully automated **backup validation pipeline** — a best practice for any business continuity plan

### Recovery Point vs Backup Job

| Term | Meaning |
|------|---------|
| **Backup job** | The process of taking the snapshot |
| **Recovery point** | The stored snapshot itself (what you restore from) |
| **Restore job** | The process of creating a new resource from a recovery point |

### Retention Policy

- Recovery points are automatically deleted after the retention period expires
- In this lab: 1-day retention (for cost/speed)
- In production: typically 30–90 days depending on compliance requirements (e.g. HIPAA, PCI-DSS)

---

## Summary

- Subscribed to an Amazon SNS topic for backup notifications
- Created a backup plan (`myBackupPlan`) with a daily rule
- Created a backup vault (`myDailyBackupVault`) with KMS encryption
- Assigned the `webAppVolume` EBS volume to the backup plan
- Configured vault event notifications via the AWS CLI
- Created an on-demand backup job and validated it via Lambda
- Reviewed CloudWatch logs from the Lambda function invocation
