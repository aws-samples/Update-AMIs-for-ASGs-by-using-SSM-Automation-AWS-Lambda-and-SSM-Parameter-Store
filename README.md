**Manage AMI updates for AWS Auto Scaling groups with AWS Lambda and AWS Systems Manager**
--------------------------------------------------------------------------------------------------

Keeping [AmazonMachine Image](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html) (AMI) up-to-date with the latest patches and updatesis a critical task for organizations using AWS [Auto Scaling group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html) . However, manually patching AMIs and updating Auto Scaling groups can be time-consuming and error-prone. This blog post presents a solution to automate the process ofupdating AMIs for Auto Scaling groups using AWS services like [AWS Systems Manager](https://aws.amazon.com/systems-manager/), [AWSLambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html), and [ Parameter Store, a capability of AWS Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html).

The key challenge is ensuring that Auto Scaling groups always launch new instances from the latest, patchedAMI. This solution leverages Systems Manager Automation to patch the current"golden" AMI, create a new AMI from the patched instance, and updatethe AMI ID stored in Parameter Store. Auto Scaling groups can then referencethis updated AMI ID parameter to launch new instances from the latest patchedAMI.

By combining Automation runbooks, Lambda functions, and Parameter Store, you can establish an automated pipeline to keep your AMIs consistently up-to-date with minimal manual effort. This solution provides a scalable and repeatable approach to maintaining a secure and compliant AWS infrastructure.

### **Solution Overview:**

The architecture of the solution can be broken down into six steps wich are outlined in Figure 1.

1.    Launch an instance from Source AMI mentionedin SSM Parameter Store

2.    Executes SSM Run Command that applies the vendor updates to the instance

3.    Stops the instance

4.    Creates a new AMI

5.    Terminates the original instance

6.    Update the parameter store using Lambda

![_Figure 1 – Architecture Diagram for updating AMI ID using SSM and lambda_](images/arch.png)



_Figure 1 – Architecture Diagram for updating AMI ID using SSM and lambda_

### **Prerequisites:**

*   You need to have an [AWS account](https://aws.amazon.com/free/?gclid=CjwKCAjwko21BhAPEiwAwfaQCMJb0CRuEuLKqMMdrXkR_YSdGoLCHh5wT1BwwmU--LrmLP122KxEsRoC6QcQAvD_BwE&trk=7541ebd3-552d-4f98-9357-b542436aa66c&sc_channel=ps&ef_id=CjwKCAjwko21BhAPEiwAwfaQCMJb0CRuEuLKqMMdrXkR_YSdGoLCHh5wT1BwwmU--LrmLP122KxEsRoC6QcQAvD_BwE:G:s&s_kwcid=AL!4422!3!651751058790!e!!g!!aws%20account%20creation%20free!19852662149!145019243897&all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all).
    

*   You should have an existing Auto Scaling group that you want to update with new AMIs.
    

*  Your EC2 instnaces must be configured with the AWS Systems Manager Agent (SSM Agent). For more information, see [Setting up AWS Systems Manager] (https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-setting-up.html)
    

*  AWS Identity and Access Management (IAM) roles for Lambda and Automation.
    

*   (Optional) SSM VPC endpoints for managing private EC2 instances without internet access
    

**Note:**

The IAM roles for automation will be deployed by the AWS Cloudformation template.

### **Solution** **Setup Steps**

#### **Task 1: Deploy Cloudformation template.**

*   Download the [UpdateMyLatestASGAmi.yml CloudFormation template](https://github.com/aws-samples/Update-AMIs-for-ASGs-by-using-SSM-Automation-AWS-Lambda-and-SSM-Parameter-Store/tree/main/templates) from the GitHub repository.
    

*   Sign in to the AWS Management Console and open the CloudFormation service
    

*   Click "Create stack" and then choose "With new resources (standard)".
    

*   Under "Prerequisite - Prepare template", select "Template is ready".
    

*   Under "Specify template", select "Upload a template file".
    

*   Click "Choose file" and select the UpdateMyLatestASGAmi.yml file you downloaded from GitHub.
    

*   Click "Next".
    

*   On the "Specify stack details" page, provide a Stack name (e.g., UpdateMyLatestASGAmi).
    

*   Click "Next".
    

*   On the "Configure stack options" page, leave the defaults and click "Next".
    

*   On the "Review" page, scroll down, check the box to acknowledge that CloudFormation might create IAM resources, and click "Create stack".
    

Wait for the CloudFormation stack to reach the "CREATE\_COMPLETE" status, which may take a few minutes.

### **IMPORTANT:**

The CloudFormation template uses the Amazon Linux 2023 image as an example in the us-east-1 region. To find the AMI ID for this image in another region, please follow [these](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html) steps.

#### **Task 2: Create a Parameter in SSM Parameter Store for the AMI ID**

Create a string parameter in Parameter Store that uses the following information:

·       Name: latestAmi

·       Value: An AMI ID. For example:ami-0f9fc25dd2506cf6d

·       Data Type: aws:ec2:image

An example is displayed, in Figure 2.
![_Figure 2: Create aparameter in SSM parameter store_](images/param.png)
_Figure 2: Create aparameter in SSM parameter store_

#### **Task 3: **Create an AWS Lambda function**

1.    Sign in to the AWS Management Console and open the AWS Lambda console at [https://console.aws.amazon.com/lambda/](https://console.aws.amazon.com/lambda/).

2.    Choose **Createfunction**.

3.    On the **Create function** page, choose **Author from scratch**.

4.    For **Function name**, enter **Automation-UpdateSsmParam**.

5.    For **Runtime**, choose **Python3.12**.[\[TT3\]](#_msocom_3) 

6.    For **Architecture**,select the type of computer processor for Lambda to use to run thefunction, **x86\_64** or **arm64**,

7.    In the **Permissions** section, expand **Change defaultexecution role**.

8.    Choose **Usean existing role**, and then choose the service role for Lambda that was created in the CloudFormation template.

9.    Choose **Createfunction**.

10.  In the **Code source** area, on the **lambda\_function** tab, delete the pre-populated code in the field, and then paste the code sample from [this](https://github.com/aws-samples/Update-AMIs-for-ASGs-by-using-SSM-Automation-AWS-Lambda-and-SSM-Parameter-Store/tree/main/lambda) GitHub link.

11.  Choose **File, Save**.

12.  To test the Lambda function,from the **Test** menu, choose **Configure test event**.

13.  For **Event name**,enter a name for the test event, such as **MyTestEvent**.

14.  Replace the existing text withthe following JSON. Replace _AMI ID_ with your own informationto set your latestAmi parameter value.

{

   "parameterName":"latestAmi",

   "parameterValue":"_AMIID_"

}

15.  Choose **Save**.

16.  Choose **Test** to test the function. On the **Execution result** tab, the status should be reported as **Succeeded**, along with other details about the update.

#### **Task 4: Update the Launch Template for the ASG to point to SSM parameter store**

To update a [launch template](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-launch-template.html) that specifies a parameter for the AMI, use one of the following method:

1.    Open the Amazon EC2 console at [https://console.aws.amazon.com/ec2/](https://console.aws.amazon.com/ec2/).

2.    In the navigation pane, choose Launch Templates, and then choose the Launch Template ID that was deployed by the CloudFormation template and select **Actions > Modifytemplate**.

3.    Under Applicationand OS Images (Amazon Machine Image), choose **Browse more AMIs**.
![__Figure 3– Browse more AMIs__](images/ami.png)

              _Figure 3– Browse more AMIs_

4.    Choose the arrow button to the right of the search bar, and then choose **Specify custom value/Systems Manager parameter.**
![__Figure 4 – chooseSpecify custom value/Systems Manager parameter__](images/value.png)

_Figure 4 – chooseSpecify custom value/Systems Manager parameter_**.**

5.    In the Specify custom value or Systems Managerparameter, enter **“resolve:ssm:latestAmi”**
![__Specify custom value or Systems Manager parameter__](images/resolve.png)
_Figure 5 –_ _Specify custom value or Systems Manager parameter_

6.    Save the changes and select **“Create templateversion”.**

7.    Ensure you have the new version is the **“defaultversion”.**
![__Set the template to Default Version__](images/default.png)

_Figure 6 –_ _Set the template to Default Version_

#### **Task 5: Create an Automation** **Runbook**

Use the following procedure to create and run a runbook that patches the AMI you specified for the latestAmi parameter. After the automation completes, the value of latestAmi is updated with the ID of the newly-patched AMI. This approach ensures that new images are automatically made available to different computing environments that use Auto Scaling groups.

**To create and run the runbook**

1.    Navigate to the AWS Systems Manager console by visiting [https://console.aws.amazon.com/systems-manager/](https://console.aws.amazon.com/systems-manager/).

2.    In the left-hand navigation pane, select"Documents".

3.    Click on "Create document" and then choose "Automation" from the options presented.

4.    In the visual editor that appears, locate thedefault name in the top left corner. Click on it and change it to "UpdateMyLatestAmi".

5.    Look for a toggle switch that allows you to switch between "Design" and "Code" views. Switch it to "Code" view.

6.   In the code editor that appears, you can input run book code from [this](https://github.com/aws-samples/Update-AMIs-for-ASGs-by-using-SSM-Automation-AWS-Lambda-and-SSM-Parameter-Store/blob/main/runbooks/updatelatestAMI.yml) link 

7.    Choose **Create automation**.

8.    In the navigation pane, choose **Automation**,and then choose **Execute automation**.

9.    In the **Choose document** page, choose the **Owned by me** tab.

10.    Search for the **UpdateMyLatestAmi** runbook, and select the button in the **UpdateMyLatestAmi** card.

11.    Choose **Next**.

12.    Choose **Simple execution.**

13.    Specify values for the input parameters.

14.    Choose **Execute**.

15.    After the automation completes, choose **ParameterStore** in the navigation pane and confirm that the new value for latestAmimatches the value returned by the automation. You can also verify the new AMIID matches the Automation output in the **AMIs** section of the Amazon EC2 console.

### **Note:**

When you specify aws:ec2:image as the data type for a parameter, Systems Manager doesn't create the parameter immediately. It instead performs an asynchronousvalidation operation to ensure that the parameter value meets the formattingrequirements for an AMI ID, and that the specified AMI is available in your AWSaccount.

A parameter version number might be generated before the validation operation is complete. The operation might not be complete even if a parameter version number is generated.

### ** Optional:Setting up Notifications Based on Parameter Store Events**

You can set up notifications to be alerted when a Parameter Store parameter is updated, deleted, or created.This can be useful for monitoring changes to the latestAmi parameter and receiving notifications when it is updated with a new AMI ID.

To set up notifications, you can configure AWS CloudWatch Events to capture Parameter Store events and send notifications to an Amazon SNS topic. Here's a high-level overview of the process:

2.  Create an Amazon SNS topic that will receive the notifications.
    

4.  In the CloudWatch console, create a new event rule to capture Parameter Store events for the specific parameter(s) you want to monitor (e.g., latestAmi).
    

6.  Add the Amazon SNS topic you created as a target for the event rule.
    

8.  Configure subscribers (e.g., email addresses) to the SNS topic to receive notifications.
    

You can follow the detailed instructions in the AWS Systems Manager User Guide: [MonitoringAWS Systems Manager Parameter Store Events](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-cwe.html).

You can automate and schedule the monitoring process using [SSM Maintenance windows](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-maintenance.html) to ensure that checks are performed consistently without manual intervention. This setup provides a reliable way to keep track of changes in the Parameter Store, enhancing security and operational visibility. It also enhances security and operational visibility.

### **Cleanup**
If you decide that you no longer want to keep the Lambda and associated resources, you can navigate to CloudFormation in the AWS Console, choose the stack (you will have named it when you deployed it), and choose Delete. All of the resources will be deleted expect the S3 bucket which has deletion policy set to retain.

### **Summary**

TThis blog demonstrates how to automatically keep AMIs up-to-date for Auto Scaling groups by leveraging AWS Systems Manager Automation, Lambda, and Parameter Store. You can schedule this automation to run routinely using AWS Systems Manager Maintenance Windows, ensuring your infrastructure consistently uses the latest, fully patched AMIs without manual effort. Instead of manually patching AMIs, you can set up an automated workflow to patch the current golden AMI, create a new patched AMI version, and update the AMI parameter referenced by your Auto Scaling groups.

While this use case focused on patching AMIs, the automation capabilities of AWS Systems Manager can be extended to consistently maintain other aspects of your AWS environment. Try out this solution yourself and experience how Automation runbooks can streamline your operations.

