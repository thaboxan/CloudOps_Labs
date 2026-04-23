# Lab 2: Infrastructure as Code with AWS CloudFormation

## Overview

Infrastructure as Code (IaC) is the practice of managing and provisioning cloud resources through machine-readable configuration files rather than manual processes. This lab introduces **AWS CloudFormation** — AWS's native IaC service — and **Resource Groups**, which allow you to organize and manage related AWS resources as a single unit.

---

## Objectives

- Understand how CloudFormation templates define infrastructure
- Deploy a CloudFormation stack from a template
- Inspect created resources in the stack
- Organize resources using AWS Resource Groups
- Understand CloudFormation drift detection concepts

---

## Architecture Summary

```
CloudFormation Template (.yaml / .json)
    └── CloudFormation Stack
            ├── VPC + Subnets
            ├── Security Groups
            ├── EC2 Instances (WebServer, AppServer)
            └── IAM Roles

Resource Group
    └── Groups resources by tag or CloudFormation stack
```

---

## Key Concepts

### CloudFormation Template Structure

A CloudFormation template is a YAML or JSON document with these main sections:

| Section | Purpose |
|---------|---------|
| `AWSTemplateFormatVersion` | Declares the template format version |
| `Description` | Human-readable description of the stack |
| `Parameters` | Input values that customize the stack at deploy time |
| `Resources` | **Required** — defines all AWS resources to create |
| `Outputs` | Values returned after stack creation (e.g. URLs, IDs) |
| `Mappings` | Static lookup tables (e.g. AMI IDs per region) |
| `Conditions` | Logic to conditionally create resources |

### Example: Minimal EC2 Resource

```yaml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0abcdef1234567890
      Tags:
        - Key: Name
          Value: WebServer
```

> **Note:** Every resource in CloudFormation has a `Type` (the AWS resource type) and `Properties` (its configuration). Changes to the template are applied by updating the stack — CloudFormation calculates and applies only the diff.

---

## Task 1: Review and Deploy a CloudFormation Stack

### Steps

1. In the AWS Console, navigate to **CloudFormation**
2. Choose **Create stack → With new resources (standard)**
3. Upload or specify the template file (`.yaml`)
4. On the **Specify stack details** page:
   - Enter a **Stack name** (e.g. `Lab2Stack`)
   - Fill in any required **Parameters**
5. On the **Configure stack options** page — add tags if needed
6. Review and choose **Create stack**
7. Monitor the **Events** tab until stack status shows `CREATE_COMPLETE`

### Notes

- CloudFormation creates resources in dependency order automatically
- If creation fails, the stack **rolls back** to its previous state by default
- Always check the **Events** tab to diagnose failures — it shows each resource and its status
- The **Resources** tab lists every resource created with its physical ID

---

## Task 2: Inspect Stack Resources

After the stack reaches `CREATE_COMPLETE`:

1. Open the stack and choose the **Resources** tab
2. Verify all resources were created (EC2 instances, Security Groups, etc.)
3. Click on a resource's **Physical ID** to navigate directly to that resource in the console

### Notes

- The **Logical ID** is the name defined in the template (`WebServer`)
- The **Physical ID** is the actual AWS resource ID (`i-0abc123...`)
- CloudFormation tracks this mapping so it can update or delete the right resource

---

## Task 3: Create a Resource Group

Resource Groups let you view and manage all resources associated with a project or environment together.

### Steps

1. Navigate to **Resource Groups & Tag Editor**
2. Choose **Create Resource Group**
3. Select **Tag-based** group type
4. Set the filter:
   - **Resource type:** All supported resource types
   - **Tag key:** `Name`  |  **Tag value:** *(e.g. `WebServer`)*
5. Enter a **Group name** (e.g. `ProductionWebServers`)
6. Choose **Create group**

### Notes

- Resource Groups are used in Lab 3 to target SSM command documents at multiple instances simultaneously
- A CloudFormation stack itself can also be used as a resource group target
- Tagging resources consistently at creation time makes grouping much easier

---

## Task 4: CloudFormation Drift Detection

**Drift** occurs when a resource's actual configuration no longer matches what was defined in the CloudFormation template (e.g. someone manually edited a security group in the console).

### Steps

1. Open your stack in **CloudFormation**
2. Choose **Stack actions → Detect drift**
3. Wait for the operation to complete (up to 1 minute)
4. Choose **Stack actions → View drift results**
5. Review the **Drift status**:
   - `IN_SYNC` — all resources match the template
   - `DRIFTED` — one or more resources have changed

### Drift Detection Table

| Status | Meaning |
|--------|---------|
| `IN_SYNC` | Resource matches template |
| `MODIFIED` | Resource exists but has been changed |
| `DELETED` | Resource was deleted outside CloudFormation |
| `NOT_CHECKED` | Drift detection not supported for this resource type |

> **Note:** CloudFormation drift detection is a manual operation — it does not run automatically. To build automated drift detection, combine it with Amazon EventBridge and Lambda (covered in Lab 6).

---

## Key Concepts Summary

### Benefits of Infrastructure as Code

| Manual Provisioning | Infrastructure as Code |
|---------------------|----------------------|
| Error-prone, inconsistent | Repeatable, consistent |
| Hard to audit | Version-controlled |
| Slow to replicate | Deploy in minutes |
| Difficult to roll back | Rollback built-in |

### CloudFormation vs Manual Changes

- **Never** manually edit resources managed by CloudFormation — always update the template
- Manual changes cause **drift** and can break future stack updates
- Use **Change Sets** to preview what will change before applying an update

### Stack Lifecycle

```
Template → Create Stack → CREATE_COMPLETE
                              ↓
                        Update Stack (via Change Set)
                              ↓
                        UPDATE_COMPLETE
                              ↓
                        Delete Stack → all resources removed
```

---

## Summary

- Deployed infrastructure from a CloudFormation template
- Inspected stack resources and their physical/logical IDs
- Created a Resource Group (`ProductionWebServers`) using tags
- Ran CloudFormation drift detection to verify stack synchronization
- Learned why IaC is preferred over manual resource management
