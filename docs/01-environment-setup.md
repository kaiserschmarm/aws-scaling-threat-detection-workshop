# Module 1: Environment build and configuration

In this first module you will be configuring detective and responsive controls for your environment.  You will be running the first of two CloudFormation templates which will automate the creation of some of these controls and then you will manually configure the rest. Log into the AWS Console if you have not done so already.

**Agenda**
 
1. Run the initial CloudFormation Template – 5 min
2. Confirm SNS subscription in your email - 1 min
3. Create a CloudWatch Rule - 5 min
4. Manually Enable detective controls - 5 min

## Enable Amazon GuardDuty

Our first step is to enable Amazon GuardDuty, which will continuously monitor your environment for malicious or unauthorized behavior.

!!! warning "If these enablement steps are missed, some resources deployed during the next steps will not configure properly such as the custom threat list within GuardDuty" 

1.	Go to the <a href="https://us-west-2.console.aws.amazon.com/guardduty/home?region=us-west-2" target="_blank">Amazon GuardDuty</a> console (us-west-2).
2.	If the **Get Started** button is available, click it. If not GuardDuty is enabled and skip step three.
3.	On the next screen click the **Enable GuardDuty** button.

GuardDuty is now enabled and continuously monitoring your CloudTrail logs, VPC flow logs, DNS Query logs, and EKS audit logs for threats in your environment.

## Enable Amazon Inspector

Our next step is to enable Amazon Inspector, which will continually scan workloads for software vulnerabilities and unintended network exposure.

1. Go to the <a href="https://us-west-2.console.aws.amazon.com/inspector/v2/home?region=us-west-2" target="_blank">Amazon Inspector</a> console (us-west-2).
2. If the **Get Started** button is available, click it. If not Inspector is enabled and skip step three.
3. On the next screen click the **Enable Inspector** button.

## Deploy the AWS CloudFormation template

To initiate the scenario and configure your environment you will need to run the module 1 CloudFormation template: 

!!! info "Before you deploy the CloudFormation template feel free to view it <a href="https://github.com/awsrossw/aws-scaling-threat-detection-workshop/blob/EventEngine/templates/01-environment-setup-nom.yml" target="_blank">here</a href>."

Region| Deploy
------|-----
US West 2 (Oregon) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=ThreatDetectionWksp-Env-Setup&templateURL=https://s3-us-west-2.amazonaws.com/sa-security-specialist-workshops-us-west-2/threat-detect-workshop/staging/01-environment-setup-nom.yml" target="_blank">![Deploy Module 1 in us-west-2](./images/deploy-to-aws.png)</a>

1. Click the **Deploy to AWS** button above.  This will automatically take you to the console to run the template, click Next to get to the Specify Details page. 

2. On the **Specify Details** section enter the necessary parameters as shown below. 

	| Parameter | Value  |
	|---|---|
	| Stack name | ThreatDetectionWksp-Env-Setup  |
	| Email Address | Any valid email address you have access to  |
	
3. Once you have entered your parameters click **Next**, 
4. Click **Next** again. \(leave everything on this page at the default\)

5. Finally, scroll down and check the box to acknowledge that the template will create IAM roles and click **Create stack**.

![IAM Capabilities](./images/iam-capabilities.png)

This will bring you back to the CloudFormation console. You can refresh the page to see the stack starting to create. Before moving on, make sure the stack is in a **CREATE_COMPLETE** status as shown below.

![Stack Complete](./images/01-stack-complete.png)

!!! info "Do not forget to check your email!"

 You will get an email from SNS asking you to confirm the Subscription. **Confirm the subscription** so you can receive email alerts from AWS services during the workshop. The email may take 2-3 minutes to arrive, check your spam/junk folder if it doesn’t arrive within that timeframe.

## Setup Amazon EventBridge rules and automatic response

The CloudFormation template you just ran created <a href="https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html" target="_blank">EventBridge Rules</a> for alerting and response purposes. The steps below will walk you through creating the final rule.  After this you'll have rules in place to receive email notifications and trigger AWS Lambda functions to respond to threats.

Below are steps to create a rule through the console but you can also find out more about doing it programmatically by reviewing the <a href="http://docs.aws.amazon.com/guardduty/latest/ug/guardduty_findings_cloudwatch.html" target="_blank">Amazon GuardDuty Documentation</a>.

1.	Open the <a href="https://us-west-2.console.aws.amazon.com/events/home?region=us-west-2" target="_blank">EventBridge console</a> (us-west-2)
2.	In the navigation pane on the left, under **Events**, click **Rules**

	!!! question "What are the current Rules in place setup to do?"
	
3.	Click **Create rule**

4.	On the **Define rule detail** screen fill out the **Name** and **Description** (suggestions below).
    * Name: **threat-detection-wksp-guardduty-finding-ec2-maliciousip**
    * Description: **GuardDuty Finding: UnauthorizedAccess:EC2/MaliciousIPCaller.Custom**

5.  Select **Rule with an event pattern** under **Rule type** if not already selected

6.  Click **Next**

7.  Under **Event pattern** Click **Edit pattern**

Copy and paste in the custom event pattern below:
	
```json
{
  "source": [
	"aws.guardduty"
  ],
  "detail": {
	"type": [
	  "UnauthorizedAccess:EC2/MaliciousIPCaller.Custom"
	]
  }
}
```
	
8.  Click **Next**

9.  For **Target 1**, click **AWS service** under *Target types*, select **Lambda Function** from the dropdown under *Select a target*, and then select **threat-detection-wksp-remediation-nacl** from the dropdown under *Function*.

10.  Select **Add another target**

11.  For **Target 2**, click **AWS service** under *Target types*, select **SNS topic** from the dropdown under *Select a target*, and then select **threat-detection-wksp** from the dropdown under *Topic*

12.  Click **Next**

13.  Click **Next** again

14.  Click **Create rule**

**Optional:** Consider examining the Lambda function to see what it does.  Open the <a href="https://us-west-2.console.aws.amazon.com/lambda/home?region=us-west-2" target="_blank">Lambda console</a>. Click on the function named **threat-detection-wksp-remediation-nacl**

    !!! question "What will the function do when invoked?"


<!-- ## Enable Amazon Macie

Since you plan on storing sensitive data in S3, let’s quickly enable Amazon Macie.  Macie is a security service that will continuously monitor data access activity for anomalies and generate alerts when it detects risk of unauthorized access or inadvertent data leaks.

1.	Go to the <a href="https://us-west-2.redirection.macie.aws.amazon.com/" target="_blank">Amazon Macie</a> console (us-west-2).

2.	Click **Get Started**.

3.	Macie will create a service-linked role when you enable it. If you would like to see the permissions that the role will have you can click the **View service role permissions**.

4.	Click **Enable Macie**.

## Setup Amazon Macie for data discovery & classification

Macie is also used for automatically discovering and classifying sensitive data.  Now that Macie is enabled, setup an integration to classify data in your S3 bucket.

1.	In the Amazon Macie console click on **Integrations** on the left navigation.

3.	Find your AWS account ID (there should be only one) and click **Select** 

4.	Click **Add** then on the next screen click the check box next to the S3 bucket that ends with **“-data”**. Click **Add**

5. Leave the options here at the default, click **Review**.

6. On the next screen click **Start Classification**. 

6. Finally click **Done**. Macie is now enabled and has begun to discover, classify and protect your data.
-->
## Enable AWS Security Hub


Now that all of your detective controls have been configured you need to enable <a href="https://aws.amazon.com/security-hub/" target="_blank">AWS Security Hub</a>, which will provide you with a comprehensive view of the security and compliance of your AWS environment.

1.	Go to the <a href="https://us-west-2.console.aws.amazon.com/securityhub/home?region=us-west-2#" target="_blank">AWS Security Hub</a> console.

2.	Click the **Go to Security Hub** button.

3.	On the next screen click the **Enable AWS Security Hub** button.

!!! note "If you see red text ```AWS Config is not enabled on some accounts``` in the Security Hub Console, you can safely ignore for this workshop."

AWS Security Hub is now enabled and will begin collecting and aggregating findings from the security services we have enabled so far.

## Architecture overview

Your environment is now configured and ready for operations.  Below is a diagram to depict the detective controls you now have in place.

![Detective Controls](./images/01-diagram-modulev2.png)

After you have successfully setup your environment, you can proceed to the next module.
