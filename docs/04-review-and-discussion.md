# Module 4: Review and Discussion

In the last module we will have a short discussion and discuss exactly what occurred. We will also go over a number of questions to test your knowledge.

<!--and then provide instructions on how to clean up the workshop environment to prevent future charges in your AWS account.
-->

**Agenda**

1. Review & Discussion – 10 min
2. Questions – 10 min
3. Cleanup – 5 min

## Architecture Overview
Below is a diagram of the overall workshop setup:
![Part 1 Diagram](./images/03-diagram-attack-v2.png)

## What is Really Going On?

In **Module 1** of the workshop you setup the initial components of your infrastructure including detective controls such as GuardDuty, Inspector, SecurityHub as well as simple notification and remediation pipeline. Some of the steps required manual configuration but you also ran a CloudFormation template which setup some of the components. 

In **Module 2** you launched a second CloudFormation template that initiated the attack simulated by this workshop. The CloudFormation template created two EC2 instances. One instance (named **Malicious Host**) had an EIP attached to it that was added to your GuardDuty custom threat list. Although the **Malicious Host** is in the same VPC as the other instance, for the sake of the scenario (and to prevent the need to submit a penetration testing request) we acted as if it is on the Internet and represented the attacker's computer. The other instance (named **Compromised Instance**) was your web server and it was taken over by the **Malicious Host**. 

In **Module 3** you investigated the attack, remediated the damage, and setup some automated remediations for future attacks.  

**Here is what occurred in the attack:**

1. There are two instances created by the Module 2 CloudFormation template. They are in the same VPC but different subnets. The **Malicious Host** represents the attacker which we pretend is on the Internet. The Elastic IP on the **Malicious Host** is in a custom threat list in GuardDuty. The other instance named **Compromised Instance** represents the web server that was lifted and shifted into AWS.

2. Although company policy is that only key-based authentication should be enabled for SSH, at some point password authentication for SSH was enabled on the **Compromised Instance**.  This misconfiguration is identified in the Inspector scan that is triggered from the GuardDuty finding.

3. The **Malicious Host** performed a brute force SSH password attack against the **Compromised Instance**. The brute force attack is designed to be successful.
	
	!!! info "**GuardDuty Finding**: UnauthorizedAccess:EC2/SSHBruteForce"

4. The SSH brute force attack was successful and the attacker was able to log in to the **Compromised Instance**.
	
	!!! info "Successful login is confirmed in CloudWatch Logs (/threat-detection-wksp/var/log/secure)."


6. The Compromised Instance also has a cron job that continuously pings the Malicious Host to generate a GuardDuty finding based off the custom threat list.
	
	!!! info "**GuardDuty Finding**: UnauthorizedAccess:EC2/MaliciousIPCaller.Custom"
 
7. The API Calls that generated the API findings come from the **Malicious Host**. The calls use the temp creds from the IAM role for EC2 running on the **Malicious Host**. The GuardDuty findings are generated because the EIP attached to the **Malicious Host** is in a custom threat list. 
	
	!!! info "**GuardDuty Finding**: Recon:IAMUser/MaliciousIPCaller.Custom or **GuardDuty Finding**: UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom"

8. A number of EventBridge Rules are evoked by the GuardDuty findings and then these trigger various services.
	1.	**EventBridge Rule**: The generic GuardDuty finding invokes an EventBridge rule which triggers SNS to send an email.
	<!-- 2.	**CloudWatch Event Rule**: The generic Macie alert invokes a CloudWatch Event rule which triggers SNS to send an email. -->
	3.	**EventBridge Rule**: The SSH brute force attack finding invokes an EventBridge rule which triggers a Lambda function to block the attacker IP address of the attacker via a NACL as well as a Lambda function that runs an Inspector scan on the EC2 instance.
	4. **EventBridge Rule**: The Unauthorized Access Custom MaliciousIP finding invokes an EventBridge rule which triggers a Lambda function to block the IP address of the attacker via a NACL.

<!-- 5. The EC2 Instance that is created in the **Module** 2 CloudFormation template disabled default encryption on the **Data** bucket.  In addition the CloudFormation template made the **Data** bucket public.  This is used for the Macie part of the investigation in Module 3. We pretend that the attacker made the bucket public and removed the default encryption from the bucket.
	
	!!! info "**Macie Alert**: S3 Bucket IAM policy grants global read rights."
-->  
## Cleanup
In order to prevent charges to your account we recommend cleaning up the infrastructure that was created. If you plan to keep things running so you can examine the workshop a bit more please remember to do the cleanup when you are done. It is very easy to leave things running in an AWS account, forgot about it, and then accrue charges. 

!!! info "If you are using this in an instructor led session, with the AWS Event Engine you do not need to run the cleanup steps"

!!! info "If you are running this in your own account. You will need to manually delete some resources before you delete the CloudFormation stacks so please do the following steps in order."

1.	Disable Inspector (if you didn't already have this enabled).
	* Go to the <a href="https://us-west-2.console.aws.amazon.com/inspector/v2/home" target="_blank">Amazon Inspector</a> console.
	* Click on **General** under **Settings** in the navigation pane on the left.
	* Click on **Disable Inspector**.
	* Enter **disable** in the text input field.
	* Click **Disable Inspector**.
	* Expand the left navigation menu.
	* Click **Switch to Inspector Classic**.
	* Click on **Assessment targets** in the navigation pane on the left.
	* Delete all targets that start with **threat-detection-wksp**.

2.	Delete the IAM Role for the compromised EC2 instance and the Service-Linked Role for Inspector (if you didn't already have this Role created).
	* Go to <a href="https://console.aws.amazon.com/iam/" target="_blank">AWS IAM</a> console.
	* Click on **Roles**
	* Search for the role named **threat-detection-wksp-compromised-ec2**.
	* Click the check box next to it and click **Delete** enter the role name into the text input field and Click **Delete**.
	* Repeat the steps above for the role named **AWSServiceRoleForAmazonInspector**.

3.	Delete all three S3 buckets created by the Module 1 CloudFormation template (the buckets that start with **threat-detection-wksp** and end with **-data**, **-threatlist** and **-logs**)

    !!! info "You must first empty the objects in each bucket before they can be deleted"

	* Go to <a href="https://s3.console.aws.amazon.com/s3/home?region=us-west-2" target="_blank">Amazon S3</a> console.
	* Click on the appropriate bucket.
	* Click **Empty**.
	* Confirm deletion by typing *permanently delete* in the text input field.
	* Click **Empty**.
	* Click **Exit**.
	* Click **Delete**.
	* Confirm deletion of the bucket by typing the name of the bucket in the text input field (this is an extra verification that you actually want to delete the bucket).
	* Click **Delete bucket**.
	* Repeat the steps above for all three buckets.

4.	Delete Module 1 and 2 CloudFormation stacks (**ThreatDetectionWksp-Env-Setup** and **ThreatDetectionWksp-Attacks**).
	* Go to the <a href="https://us-west-2.console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks?filter=active" target="_blank">AWS CloudFormation</a> console.
	* Select the appropriate stack.
	* Click **Delete**.
	* Repeat the steps above for each stack.

	!!! info "You do not need to wait for the first stack to delete before you delete the second one."

5.	Delete the GuardDuty custom threat list and disable GuardDuty (if you didn't already have it configured before the workshop)
	* Go to the <a href="https://us-west-2.console.aws.amazon.com/guardduty/" target="_blank">Amazon GuardDuty</a> console.
	* Click on **Lists** on the left navigation.
	* Click the **X** next to the threat list that starts with **Custom-Threat-List**.
	* Click **Delete**

6.	Disable AWS Security Hub
	* Go to the <a href="https://us-west-2.console.aws.amazon.com/securityhub/home?region=us-west-2#/findings" target="_blank">AWS Security Hub</a> console.
	* Click on **Settings** on the left navigation.
	* Click the **General** on the top navigation.
	* Click **Disable AWS Security Hub**.

6.	Delete the manual EventBridge Rule you created and the CloudWatch Logs that were generated.
	* Go to the <a href="https://us-west-2.console.aws.amazon.com/events" target="_blank">EventBridge</a> console.
	* Click on **Rules** in the navigation pane on the left.
	* Click the radio button next **threat-detection-wksp-guardduty-finding-maliciousip**.
	* Click **Delete**.
	* Go to the <a href="https://us-west-2.console.aws.amazon.com/cloudwatch" target="_blank">CloudWatch</a> console.
	* Click on **Log groups** in the navigation pane on the left.
	* Select the Log groups containing **threat-detection-wksp**
	* Select **Action** and click **Delete log group(s)** and then click **Delete**.

7.	Delete the SNS subscription that was created when you subscribed to SNS Topic.
	* Go to the <a href="https://us-west-2.console.aws.amazon.com/sns" target="_blank">AWS SNS</a> console.
	* Click on **Subscriptions** on the left navigation.
	* Select the radio field next to the subscription that shows your e-mail as the Endpoint and has **threat-detection-wksp** in the **Subscription ARN**.
	* Click **Delete**

<!--8.	Disable Macie (if you didn't already have Macie enabled before the workshop).
	* Go the <a href="https://mt.us-west-2.macie.aws.amazon.com/" target="_blank">Amazon Macie</a> console.
	* In the upper right-hand corner select the down arrow to the left of the Region and select **Macie General Settings**.
	* Check the two boxes and click **Disable Amazon Macie**-->

## Finished!

Congratulations on completing this workshop! This is the workshop's permanent home, so feel free to revisit as often as you'd like.


