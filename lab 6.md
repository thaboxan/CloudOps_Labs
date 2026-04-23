Lab 6: Capstone Lab for CloudOps
© 2026 Amazon Web Services, Inc. or its affiliates. All rights reserved. This work may not be reproduced or redistributed, in whole or in part, without prior written permission from Amazon Web Services, Inc. Commercial copying, lending, or selling is prohibited. All trademarks are the property of their owners.

Note: Do not include any personal, identifying, or confidential information into the lab environment. Information entered may be visible to others.

Corrections, feedback, or other questions? Contact us at AWS Training and Certification.

Lab overview
Tasks to be solved for this lab are divided into three parts. The first part requires you to create a custom mechanism that sends out an email notification when an AWS Systems Manager SSM document successfully completes. The second part requires you to use AWS CloudFormation drift detection to discover configuration changes and perform remediation actions using SSM documents. The third part requires you to use AWS Config rules to detect an Amazon Simple Storage Service (Amazon S3) storage configuration that is out of compliance with company policy and then setup an auto-remediation solution for all occurrences of the issue using AWS Config rules and SSM documents.

This lab concludes the Cloud Operations on AWS course. It is intended to provide both a summary review of monitoring tools previously covered, as well as a review of the final day’s topics including networking and storage. In this lab students are guided through three distinct monitoring and troubleshooting scenarios which are solved by using some of the tools and techniques previously covered.

Objectives
By the end of this lab, you should be able to do the following:

Use Amazon EventBridge and Amazon Simple Notification Service (Amazon SNS) to create monitoring and notification solutions.
Use AWS Config Rules to detect out of compliance configurations.
Use AWS Config and AWS Systems Manager document (SSM document) to remediate compliance issues.
Use AWS CloudFormation drift detection to identify network configuration changes.
Technical knowledge prerequisites
This lab requires the following prerequisites:

Access to a computer with internet and Microsoft Windows, macOS X, or Linux.
An internet browser, such as Google Chrome, Mozilla Firefox, or Microsoft Internet Explorer 11 (previous versions of Internet Explorer are not supported).
Basic knowledge of AWS CloudFormation, AWS Config, AWS Systems Manager, Amazon EventBridge, and Amazon Simple Notification Service.
Icon key
Various icons are used throughout this lab to call attention to different types of instructions and notes. The following list explains the purpose for each icon:

 Learn more: Where to find more information.
 Note: A hint, tip, or important guidance.
 WARNING: An action that is irreversible and could potentially impact the failure of a command or process (including warnings about configurations that cannot be changed after they are made).
 Task complete: A conclusion or summary point in the lab.
Start lab
To launch the lab, at the top of the page, choose Start Lab.

 Caution: You must wait for the provisioned AWS services to be ready before you can continue.

To open the lab, choose Open Console .

You are automatically signed in to the AWS Management Console in a new web browser tab.

 Warning: Do not change the Region unless instructed.

Common sign-in errors
Error: You must first sign out
Log out error

If you see the message, You must first log out before logging into a different AWS account:

Choose the click here link.
Close your Amazon Web Services Sign In web browser tab and return to your initial lab page.
Choose Open Console again.
Error: Choosing Start Lab has no effect
In some cases, certain pop-up or script blocker web browser extensions might prevent the Start Lab button from working as intended. If you experience an issue starting the lab:

Add the lab domain name to your pop-up or script blocker’s allow list or turn it off.
Refresh the page and try again.
Lab environment
The following diagram represents the major components used in this lab. The numbers represent the logical workflow of the architecture in this lab.

The architecture diagram of the lab 6 environment.

Image description: The preceding diagram depicts the group labeled as part 1 displays the relevant Amazon Web Services (AWS) services you use to build a notification mechanism. This notification mechanism is leveraged by the groups in the lab labeled part 2 and part 3.The group labeled as part 2 displays the relevant AWS services you use to make changes to the existing lab environment and complete part 2 of the lab. In part 2 of the lab, the drift detection tool of CloudFormation is used to locate a configuration change of the stack. The change is remediated with SSM documents. The completion of these SSM documents sends a notification out from the mechanism that was created in part 1 of the lab.The group labeled part 3 displays the relevant AWS services you use to make changes in the existing lab environment and complete part 3 of the lab. In part 3 of the lab, AWS Config Rules and AWS Systems Manager SSM documents are used together to create an auto-remediation mechanism for certain configuration AWS resources in the lab environment. When this auto-remediation mechanism completes running, a notification is sent out from mechanism created in part 1.

Part 1: Amazon EventBridge and automated notifications from Systems Manager
In this task, you use the Amazon SNS and Amazon EventBridge services to create email notifications following the successful completion of an SSM document.

Part 1 (Challenge) Use Amazon EventBridge to forward event notifications to your Amazon SNS topic
Instructions in this challenge section are purposefully left vague to give you a chance to apply techniques learned from previous labs and resolve the compliance issue. You can skip this challenge section and continue onto to the detailed guidance steps if you choose.

For the challenge section, try to build a mechanism using Amazon EventBridge that forwards notifications to an Amazon SNS topic whenever an SSM document from the Systems Manager service successfully completes.

Create a new Amazon SNS topic named SuccessfulAutomationAction.
Subscribe your email to the Amazon SNS topic.
Create a new EventBridge Rule named AutomationSuccess that triggers when an SSM document status updates to ‘success’.
Set the target of the rule to the Amazon SNS topic named SuccessfulAutomationAction.
Test the notification system by starting the AWS-ConfigureCloudWatchOnEC2Instance SSM document, and then checking your email for the notification from Amazon SNS when the automation completes successfully.
Continue onto to the next task for detailed guidance.

Task 1: Create and subscribe to Amazon SNS notifications
In this task, you create the Amazon SNS topic necessary for setting up a notification service, and then subscribe your mail address to the topic.

Task 1.1: Create a new AWS Simple Notification Service topic
At the top of the page, in the unified search bar, search for and choose Simple Notification Service.

Choose the hamburger menu icon  to expose the left navigation menu. Then choose Topics.

Choose Create topic

For Type, choose  Standard.

For Name enter SuccessfulAutomationAction.

Scroll to the bottom and choose Create topic.

The Amazon SNS topic SuccessfulAutomationAction is created and details page for the topic is displayed.

 Task complete: You have created a new Amazon SNS topic.

Task 1.2: Subscribe to an AWS Simple Notification Service topic
Subscribe to an existing Amazon Simple Notification Service topic. The topic is used to alert all subscribers about the successful completion of Systems Manager SSM document executions.

Still in the same page, in the Subscriptions tab, choose Create subscription.

For Protocol select Email.

In the Endpoint field enter a valid email address you can access.

 Note: In your personal AWS environment, this might be an email alias for all of the CloudOps engineers. Individuals receive an email and have to confirm their subscription prior to receiving future notifications from the topic.

Choose Create subscription.

A banner message similar to the following is displayed at the top of the page, “ Subscription to SuccessfulAutomationAction created successfully.” letting you know the email address was successfully registered to the Amazon SNS topic.

Open the inbox of the email address you entered for the subscription.

Locate a recent message from AWS Notifications <no-reply@sns.amazonaws.com>.

 Note: It may take up to 5 minutes to receive the email, depending on your email server.

Select the Confirm subscription hyperlink contained in the email.

A page is opened confirming the subscription. Lab6 SNS subscription confirmation page

Close the Amazon SNS topic subscription confirmation page.

 Task complete: You have successfully subscribed to an Amazon SNS topic. With a subscription, Amazon SNS pushes new messages from this topic to your email address.

Task 2: Setup an Amazon EventBridge rule for SSM document success
In this task, you setup a new Amazon EventBridge rule to monitor for SSM document successful completion and then publish notifications through an Amazon SNS topic.

Task 2.1: Create a EventBridge rule
At the top of the page, in the unified search bar, search for and choose Amazon EventBridge.

Choose the hamburger menu icon  to expose the left navigation menu if necessary. Then choose Rules.

Verify the Event pattern rules tab is selected, then choose Create rule.

 Note: If you see a “Rule creation experience” dialog, de-select the “Visual rule builder opt in” option.

For Name enter AutomationSuccess.

Choose Next.

For Event source choose  AWS events or EventBridge partner events.

In the Event pattern section:

Keep Creation method set to Use pattern form
Keep Event source set to AWS services.
For AWS service, select Systems Manager.
For Event type select Automation.
Under Event Type Specification 1 select  Specific detail type(s).

In the Specific detail type(s) menu, select only the EC2 Automation Execution Status-change Notification option.

Under Event Type Specification 2 select  Specific status(es).

In the Specific status(es) menu, select only the Success option.

Choose Next.

The Select target(s) page is displayed.

Under Target types, ensure AWS service is selected.

For Select a target search for and choose SNS topic.

In the Topic drop-down menu, select SuccessfulAutomationAction.

Under Permissions, uncheck (deselect) Use execution role (recommended)

Choose Next.

Choose Next.

Choose Create rule.

The event pattern rule is created. It appears in the list of rules. The newly created rule will be enabled by default.

 Task complete: You have created a new EventBridge rule and linked it to the Amazon SNS topic.

Task 2.2: Test notification system by running a Systems Manager SSM document
At the top of the page, in the unified search bar, search for and choose Systems Manager.

Choose the hamburger menu icon  if necessary to expose the left navigation menu. Then under Change Management Tools choose Documents.

At the top of the page, verify that Owned by Amazon is selected then in the Documents search area search for AWS-ConfigureCloudWatchOnEC2Instance

Select the hyperlink for the AWS-ConfigureCloudWatchOnEC2Instance document.

The Description page of the AWS-ConfigureCloudWatchOnEC2Instance document is displayed.

Select the Content tab and review the SSM document.

Choose the Execute automation button.

The Execute automation runbook page is displayed.

Leave the default choice of  Simple execution.

 Note: Simple execution is a good choice for this lab as the architecture is relatively straightforward. Read descriptions of the other execution options and think about what kinds of scenarios they would be useful for.

From the left of these lab instructions, copy the value for the InstanceID .

Back in the Systems Manager console, locate the Input parameters section and under InstanceId paste in the value you just copied.

Scroll down and choose Execute.

 WARNING: If the SSM document execution status fails instead of succeeding, double-check the values used in the Input parameters section. Whitespace used in the InstanceID field could cause an error. If you find any errors, correct them, wait one minute and run the SSM document once more. Ask your instructor if you need help retrying this SSM document.

Within a minute, the Overall status in the Execution status section returns ‘ Success’.

Return to the inbox of the email account you subscribed to the Amazon SNS topic named SuccessfulAutomationAction and wait for an email from Amazon SNS.

Amazon SNS delivers to your email a JSON formatted text string detailing the SSM document that completed.

 Note: It may take up to 5 minutes to receive the email depending on your email server.

 Task complete: You have run an SSM document, triggered the notification system and completed this task.

Part 2: AWS CloudFormation and automated remediation
In this task, you use AWS CloudFormation drift detection to detect a change to your networking security and then setup auto remediation for this specific issue using AWS Systems Manager.

 Note: Recall from lab 2 ‘Infrastructure as Code’ that drift detection is a tool in AWS CloudFormation that helps you identify what item(s) in your AWS CloudFormation stack(s) are currently out of synchronization from their original template. Automated drift detection for stacks is not a feature built-in to AWS CloudFormation.

 Learn more: Refer to additional resources to discover using multiple AWS services to build an architecture that automatically detects drift.

Part 2 (Challenge) Detect a change from AWS CloudFormation template and remediate with Systems Manager
Instructions in this challenge section are purposefully left vague to give you a chance to apply techniques learned from previous labs and resolve the compliance issue. You can skip this challenge section and continue onto to the detailed guidance steps if you choose.

For the challenge section, find the change made to your environment by comparing the original AWS CloudFormation template and the current configuration of resources. Once you find the changes made to your environment, revert the resource back to their original configuration using a Systems Manager SSM document. Then run drift detection once again to confirm resources are in synchronization with their original AWS CloudFormation template.

Navigate to the AWS CloudFormation console.
Perform a drift detection on the AWS CloudFormation lab stack.
Identify which stack resources have been modified and how.
Navigate to the Systems Manager console.
Browse the Systems Manager Document library and locate an appropriate SSM document to remediate the non-compliant resources found by the drift detection.
Run the SSM document.
Navigate to the AWS CloudFormation console.
Perform a drift detection on the AWS CloudFormation stack.
Verify the AWS CloudFormation stack returns an ‘in sync’ status.
Continue onto to the next task for detailed guidance.

Task 3: Perform a drift detection operation on your environment’s AWS CloudFormation stack
In this task, perform a drift detection operation.

Drift detection is a tool within the AWS CloudFormation service that lets you quickly locate stack resources which are no longer configured as defined by their original template.

 Learn more: Refer to additional resources for documentation on a current list of resources supported by drift detection as well as some notable limitations of the tool.

Task 3.1: Start an AWS CloudFormation drift detection operation
At the top of the page, in the unified search bar, search for and choose CloudFormation.

Select the hyperlink for the AWS CloudFormation stack with the description Lab 6 Capstone.

The Stack info page is displayed.

Choose the Stack actions drop down menu and select Detect drift.

An informational banner is displayed at the top of the page with text similar to, ‘Drift detection initiated for arn:aws:cloudformation:us-west-2:1234567890:stack/qls-abcdefg123’.

Wait as long as 1 minute for the drift detection job to complete before continuing to the next task.

 Task complete: You have started a drift detection operation.

Task 3.2: Review AWS CloudFormation drift detection results and identify the out of sync resource
From the Stack actions menu, select View drift results.

The Drifts page is displayed.

The Drift status for this stack is  DRIFTED.

Locate Resource drift status section.

Review which resource has a Drift status  MODIFIED.

Select the radio button next to  AppSecurityGroup.

Select View drift details.

The Drift details page is displayed.

Copy the value for the Physical ID of the Security Group from the Resource drift overview section to a notepad. The ID begins like sg-1234…. You need this value in a later task.

Select the checkbox  next to SecurityGroupIngress in the Differences section.

Review the highlighted code in the Details section and discover what configuration has been changed in this Security group.

 Task complete: You have used drift detection to identify specific stack resources that have changed from the stack template.

Task 4: Use an SSM automation document to remediate Amazon EC2 security groups that are open to the public
In this task, you use SSM documents to disable public access to individual Amazon EC2 Security groups.

Task 4.1: Locate a suitable SSM Document
At the top of the page, in the unified search bar, search for and choose Systems Manager.

Choose the hamburger menu icon  if necessary to expose the left navigation menu. Then under Change Management Tools choose Documents.

At the top of the page, verify that Owned by Amazon is selected then in the Documents search area search for AWS-DisablePublicAccessForSecurityGroup

Select the hyperlink for the SSM document named AWS-DisablePublicAccessForSecurityGroup.

The Description page of the AWS-DisablePublicAccessForSecurityGroup document is displayed.

Select the Content tab and review the SSM document.

Select the Execute automation button.

The Execute automation document page is displayed.

Leave the default choice of  Simple execution.

In the Input Parameters section, locate the GroupId and paste in the Security Group ID you copied earlier.

 Note: It is a string value that starts with sg-. The same value is also located to the left of these lab instructions.

Scroll down and choose Execute.

The Execution details page is displayed.

A banner message is displayed at the top of the page with text similar to ‘Execution has been initiated.’

Wait until the Overall status in the Execution status section of the page displays  Success.

 Note: You may notice some of the individual executed steps from the operation succeed while others fail. Recall the content of the document you examined before you ran it. There are more cases handled by the document than are applicable to the current scenario. If you investigate the reason for the individual step failure, the reason is because the rule specified by that step does not exist in the security group you specified. Overall the document succeeds with individual failed steps. This is by design.

Return to the inbox of the email account you subscribed to the Amazon SNS topic named SuccessfulAutomationAction and wait for an email from Amazon SNS.

Amazon SNS delivers to your email a JSON string detailing each of the SSM document steps that complete.

 Note: It may take up to 5 minutes to receive the email depending on your email server. You do not need to wait for the email notification as you have already tested that the system works in task 2.2.

 Task complete: You manually ran an SSM Document and removed a publicly available SSH port from a security group.

Task 4.2: Run drift detection operation on the AWS CloudFormation stack to verify the stack resources are in sync with the template
Run the CloudFormation drift detection operation to verify stack resources are in sync with the template that created them.

Search for CloudFormation using the Unified Search Bar at the top of the AWS Console, and choose the service from the listed results.

Select the hyperlink for the AWS CloudFormation stack with the description of Lab 6 Capstone lab.

The Stack info page is displayed.

Choose the Stack actions drop down menu and select Detect drift.

An informational banner is displayed at the top of the page with text similar to ‘Drift detection initiated for arn:aws:cloudformation:us-west-2:1234567890:stack/qls-abcdefg123’

Wait as long as 1 minute for the drift detection job to complete before continuing to the next task.

From the Stack actions menu, select View drift results.

The Drifts page is displayed.

Confirm that the Drift status now displays  IN_SYNC.

 Task complete: After making a configuration change to a security group using an SSM document, you then ran the AWS CloudFormation drift detection tool and verified that the stack is now in sync with the original template.

Part 3: AWS Config Rules and automated remediation
In this task, you use AWS Config to detect non-compliance violations with your company’s data security policy regarding Amazon S3 Buckets that do not have bucket versioning enabled. Next, you institute an automated remediation action to correct this particular violation, as well as future potential violations of this policy.

Task 5: Using the AWS Config Dashboard to identify noncompliant resource
Use the AWS Config service to identify non-compliant storage resource that currently exist in the infrastructure.

Task 5.1: Set up AWS Config and create a new rule
Search for AWS Config using the Unified Search Bar at the top of the AWS Console, and choose the service from the listed results.

Choose the Get started button.

The settings page of the AWS Config setup is displayed.

For the Settings page, in Recording method section configure the following:

Choose the  All resource types with customizable overrides option from the Recording Strategy section.

Choose the Continuous recording option from the Recording Frequency section, under Default settings.

In the Data governance section configure the following:

Choose the  Choose a role from your account option from the IAM role for AWS Config section.
Choose AWSServiceRoleForConfig from Existing roles.

In the Delivery channel section configure the following:

Choose the  Create a bucket option from the Amazon S3 bucket section.
Choose the Next button.

The Rules page of the AWS Config setup is displayed.

Locate the AWS Managed Rule named S3-bucket-versioning-enabled.

Fill the checkbox next to the rule titled  S3-bucket-versioning-enabled.

Choose the Next button.

The Review page of the AWS Config setup is displayed.

Choose the Confirm button.

AWS Config finishes its initial setup process.

The AWS Config dashboard is displayed.

Choose Rules from the AWS Config navigation bar.

Choose the hyperlink for the rule named s3-bucket-versioning-enabled from the list.

Select the All  option from the drop-down menu in the Resources in Scope section.

 Important: If your AWS Config rule returns a result of No resources in scope you need to wait as long as 5 minutes for other AWS resources to synchronize with AWS Config and re-evaluate the AWS Config rule.

In the Resources in scope section, the Compliance column displays a status of Evaluating… for the rule while the scan is in progress. A result of ‘Non-compliant resource(s)’ appears after a few minutes when the compliance scan is finished.

For this step of the lab you do not need to wait for the status to change from ‘Evaluating…’ and is Noncompliant. Continue to the next step.

In the Resources in scope list, locate the bucket which has the string labbucket in the name and select the hyperlink for it. You may need to expand the column width to view the full bucket names.

The resource details page is displayed.

Choose the Manage Resource  button.

The details page for the Amazon S3 bucket is displayed.

Select the Properties tab.

Verify that Bucket Versioning is Disabled for this bucket.

Part 3 (Challenge): Edit the AWS Config rule for Auto-remediation
Instructions in this challenge section are purposefully left vague to give you a chance to apply techniques learned from previous labs and resolve the compliance issue. You can skip this challenge section and continue onto to the detailed guidance steps if you choose.

For the challenge section, identify the appropriate SSM document to use. Then find a way to link the deployment status of the SSM document to the AWS Config rule findings of non-compliant resources.

Navigate to the Systems Manager console.
Browse to the SSM document library.
Locate an appropriate SSM document to remediate the non-compliant resources as found by the AWSConfig rule named s3-bucket-versioning-enabled.
Return to the AWS Config console.
Edit the s3-bucket-versioning-enabled AWS Config rule to appropriately to remediate the compliance issue for the bucket that does not have bucket versioning enabled using an SSM document named AWS-ConfigureS3BucketVersioning.
Continue onto to the next task for detailed guidance.

Task 6: Configure auto-remediation for your AWS Config rule
In this task, you set up the auto-remediation for the AWS Config rule and verify the results.

Search for AWS Config using the Unified Search Bar at the top of the AWS Console, and choose the service from the listed results.

Choose Rules from the AWS Config navigation bar.

Choose the hyperlink for the rule named s3-bucket-versioning-enabled.

The details page for the rule is displayed.

Choose the Actions  button.

Select the Manage remediation option from the list.

The Edit: Remediation action page is displayed.

Choose the  Automatic remediation option from the Select remediation method section.

Locate the Remediation action details section.

Choose the Choose remediation action drop-down menu.

Search for and select AWS-ConfigureS3BucketVersioning from the list.

Scroll down to the Resource ID parameter section.

Select BucketName from the drop-down menu.

Locate the Parameters section and configure the following:

For VersioningState: Enter Enabled
For AutomationAssumeRole: Paste the value of LabConfigRoleARN located to the left of these lab instructions.
Choose the Save changes button.

The details page for the rule is displayed.

With auto-remediation now set up for the AWS Config rule, the resources found to be noncompliant by the Config rule are fixed without user intervention.

Scroll down the page to the Resources in scope section. Change the drop-down list value to All. If you wait long enough, you will see that the compliance status of the S3 bucket that contains labbucket in the bucket name changes to ‘Compliant’.

However, you do not need to wait from the status to change to compliant. Continue onto the next step.

 Note: From this point on, if more S3 buckets without Bucket Versioning are added to the environment, they would also be detected by the Rule scan and auto-remediated without the need of manual action.

(Three Optional steps) Rather than waiting for auto-remediation actions, the following three steps detail manual remediation.

From the Rules page of the AWS Config console select the hyperlink for the rule named s3-bucket-versioning-enabled.

Select the radio switch  next to the S3 bucket which has string labbucket in the name.

Choose the Remediate button.

The Action Status column for the resource changes from ‘n/a’ to 'Action execution queued… ’ and finally to ‘ Action executed successfully’ when the SSM document specified in the Remediation settings of the rule completes.

 Note: It takes a few minutes for the resource to be re-classified as compliant by AWS Config. It is not necessary to wait for this. Continue onto the next task.

In part 1 of the lab you created a notification mechanism. The email account you subscribed to the Amazon SNS topic named SuccessfulAutomationAction receives a new email from Amazon SNS as each time an SSM document completes. The email is a JSON string detailing each of the steps that SSM document completes. In this task each time a resource is auto-remediated by Config, a new email is be sent out. This is because the auto-remediation is occurring through use of SSM documents.

 Note: It may take up to 5 minutes to receive the email, depending on your email server. You do not need to wait for email notifications about all the non-compliant Amazon S3 buckets to continue with the lab.

Locate the Resources in scope section.

Select All  from the drop-down menu in the Resources in Scope section.

Select the hyperlink for the bucket with the string labbucket in the name.

The details page of the resource is displayed.

Choose the Manage Resource  button.

You are taken to the S3 console where the details of the selected bucket are displayed.

Choose the Properties tab.

Verify the Bucket Versioning is Enabled for this Amazon S3 bucket.

 Task complete: You have combined Systems Manager Command Documents, with AWS Config rule, and auto-remediation features. Non-compliant resources were located and remediated.

Conclusion
You have successfully done the following:

Used Amazon EventBridge and Amazon Simple Notification Service (Amazon SNS) to create monitoring and notification solutions.
Used AWS Config Rules to detect out of compliance configurations.
Used AWS Config and AWS Systems Manager document (SSM document) to remediate compliance issues.
Used AWS CloudFormation drift detection to identify network configuration changes.