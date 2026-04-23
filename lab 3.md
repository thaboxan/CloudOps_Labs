# Lab 3: Operations as Code with AWS Systems Manager

## Overview

As a CloudOps engineer, manually logging into individual EC2 instances to make changes is slow, error-prone, and unauditable. This lab shows how to use **AWS Systems Manager Documents** to deploy configuration changes across multiple servers with a single command.

You will:
- Create a **CloudWatch Log Group** to capture SSM command output
- Write and run **Command Documents** to install and configure Apache across a resource group of EC2 instances
- Use an **Automation Document** to resize an EC2 instance
- Inspect **CloudWatch log streams** to audit what was done

---

## Objectives

- Create a CloudWatch Log Group for SSM document output
- Create and run a Command Document to install Apache web server
- Create and run a Command Document to update web content
- Run an Automation Document to resize an EC2 instance
- Audit changes through CloudWatch log stream inspection

---

## Architecture Summary

```
SSM Command / Automation Document
        ↓
  Resource Group (ProductionWebServers)
        ↓
  EC2 Instances (WebServer1, WebServer2, ...)
        ↓
  CloudWatch Logs (WebServerSSMAgentLogGroup)
        ↓
  Log Streams (stdout / stderr per instance)
```

---

## Task 1: Create a CloudWatch Log Group

SSM Run Command can send its output to CloudWatch Logs for near-real-time monitoring, searching, and alarming — much better than the default 48,000-character console output limit.

### Steps

1. Navigate to **CloudWatch → Log groups**
2. Choose **Create log group**
3. Enter `WebServerSSMAgentLogGroup` as the name
4. Choose **Create**

### Notes

> **IAM requirement:** For EC2 instances to push logs to CloudWatch, their IAM role must include these permissions:
> ```json
> {
>   "Effect": "Allow",
>   "Action": [
>     "logs:CreateLogGroup",
>     "logs:CreateLogStream",
>     "logs:DescribeLogGroups",
>     "logs:DescribeLogStreams",
>     "logs:PutLogEvents"
>   ],
>   "Resource": "arn:aws:logs:::log-group:/aws/ssm/*"
> }
> ```
> In this lab the instances already have the managed policy `CloudWatchAgentServerPolicy` attached.

- SSM Agent also writes its own local logs on each instance:
  - **Linux:** `/var/log/amazon/ssm/`
  - **Windows:** `%PROGRAMDATA%\Amazon\SSM\Logs\`

---

## Task 2: Create and Run SSM Command Documents

### Task 2.1: Command Document — Install Apache

Create a command document that updates packages and installs the Apache web server.

1. Navigate to **Systems Manager → Documents**
2. Choose **Create document → Command or Session**
3. Enter `myInstallApache` for the name
4. Select **YAML** for content format
5. Paste the following:

```yaml
schemaVersion: '2.2'
mainSteps:
  - action: 'aws:runShellScript'
    name: 'configureApache'
    inputs:
      runCommand:
        - 'sudo yum update -y'
        - 'sudo yum install -y httpd'
        - 'sudo service httpd start'
        - 'sudo chkconfig httpd on'
        - 'sudo chkconfig httpd --add'
```

6. Choose **Create document**

> **Warning:** If you get `InvalidDocumentContent: null`, check YAML indentation — YAML is whitespace-sensitive.

### Task 2.2: Run the Command Document

1. Open **myInstallApache** from the **Owned by me** tab
2. Choose **Run command**
3. In **Target selection**, select **Choose a resource group → ProductionWebServers**
4. In **Output options**:
   - ☐ Uncheck **Enable S3 bucket**
   - ☑ Check **Enable CloudWatch logs**
   - Log group: `WebServerSSMAgentLogGroup`
5. Choose **Run**
6. Refresh until all instances show **Success**

### Task 2.3: Verify Apache is Running

1. Copy a **WebServer public DNS** value from the lab panel
2. Paste it into a new browser tab (**without** `https://` — SSL is not configured)
3. You should see the default Apache test page ✅

> **Note:** Using HTTP (port 80) directly is expected here. Adding SSL would require an ACM certificate and load balancer — out of scope for this lab.

---

### Task 2.4: Command Document — Update Web Content

Create a second command document that writes a dynamic hostname-based `index.html`.

1. Navigate to **Systems Manager → Documents → Create document → Command or Session**
2. Enter `myIndexContentUpdate` as the name
3. Select **YAML** and paste:

```yaml
---
schemaVersion: '2.2'
mainSteps:
  - action: 'aws:runShellScript'
    name: 'configureApache'
    inputs:
      runCommand:
        - 'sudo bash -c "echo Hello World from $(hostname -f) > /var/www/html/index.html"'
```

4. Choose **Create document**

### Task 2.5: Run the Content Update Document

1. Open **myIndexContentUpdate** → **Run command**
2. Target: **ProductionWebServers** resource group
3. Enable CloudWatch Logs → `WebServerSSMAgentLogGroup`
4. Choose **Run** → wait for **Success**

### Task 2.6: Verify the Updated Content

1. Paste a WebServer public DNS into a browser tab (no `https://`)
2. The page should now display: `Hello World from <hostname>` ✅

> **Key insight:** With resource groups, you ran the exact same command across **all** WebServer instances simultaneously — no SSH, no manual steps. This scales to hundreds of instances with no additional effort.

---

## Task 3: SSM Automation Document — Resize EC2 Instance

The dev team's performance review found that `AppServer1` needs to be upgraded from its current instance type to `t2.small`.

### Task 3.1: Run AWS-ResizeInstance

1. Navigate to **Systems Manager → Automation**
2. Choose **Execute runbook**
3. Switch to the **Owned by Amazon** tab
4. Search for and select `AWS-ResizeInstance`
5. Review the **Content** tab to understand the document's steps before running
6. Choose **Execute automation → Simple execution**
7. Enable **Show interactive instance picker** → select `AppServer1`
8. Set `InstanceType` to `t2.small`
9. Choose **Execute**

**What happens:**

| Step | Outcome |
|------|---------|
| `assertInstanceType` | **Fails** — current type ≠ `t2.small` (expected, triggers resize) |
| Subsequent steps | Proceed to stop, resize, and restart the instance |
| Overall status | **Success** |

> **Note:** The `assertInstanceType` step intentionally fails when the current instance type doesn't match the target. This failure is the condition that triggers the resize steps. If you ran this document a second time (after the resize), `assertInstanceType` would succeed and no further steps would run — the document is idempotent.

### Task 3.2: Verify the Resize

1. Navigate to **EC2 → Instances**
2. Select `AppServer1`
3. Verify **Instance type** in the Details tab shows `t2.small` ✅

---

## Task 4: Review CloudWatch Log Streams

1. Navigate to **CloudWatch → Log groups → WebServerSSMAgentLogGroup**
2. Open any log stream
3. Review the timestamped entries

### What you'll see

- **stdout streams** — command output (e.g. yum install messages)
- **stderr streams** — any errors encountered during execution
- Each EC2 instance that ran the command has its own log stream

> **Audit value:** These log streams give you a full, timestamped audit trail of every command run via Systems Manager. Combined with CloudTrail, you can trace exactly who ran what, on which instances, and when.

---

## Key Concepts

### SSM Document Types

| Type | Purpose | Example |
|------|---------|---------|
| Command | Run shell/PowerShell commands on instances | Install software, edit files |
| Automation | Multi-step workflows with conditionals | Resize instance, patch AMI |
| Session | Open interactive terminal via Session Manager | Debugging, ad-hoc commands |
| Policy | Define configuration state (like Puppet/Chef) | Ensure service is running |

### Why Resource Groups Matter

Without resource groups you'd have to:
1. Manually find all `ProductionWebServers` instances
2. Run the command on each one individually
3. Repeat for every future change

With resource groups, all instances matching the group criteria are targeted **automatically** — even new ones added later.

### Command Document vs Automation Document

| Feature | Command Document | Automation Document |
|---------|-----------------|---------------------|
| Target | EC2 instances | AWS resources/APIs |
| Language | Shell / PowerShell | YAML with AWS actions |
| Conditional logic | Limited | Full step conditions |
| Use case | OS-level changes | Infrastructure changes |

---

## Summary

- ✅ Created `WebServerSSMAgentLogGroup` in CloudWatch for SSM output logging
- ✅ Created and ran `myInstallApache` — installed Apache on all WebServers via a resource group
- ✅ Created and ran `myIndexContentUpdate` — pushed custom content to all WebServers
- ✅ Ran `AWS-ResizeInstance` automation to upgrade `AppServer1` to `t2.small`
- ✅ Inspected CloudWatch log streams to audit all SSM operations
