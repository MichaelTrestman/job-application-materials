# AWS-managed SecurityAudit IAM Policy risk assessment and remediation

## Introduction

Amazon offers a managed Identity and Access Management \(IAM\) policy, named [**SecurityAudit**](https://console.aws.amazon.com/iam/home?#/policies/arn:aws:iam::aws:policy/SecurityAudit$actionLevelSummary?service=EC2&effect=Allow&section=permissions), designed to grant a security auditor privileges sufficient to perform security auditing functions. However, granting infrastructure access privileges should always be carefully considered, especially when the user would be someone outside of the organization or company. As an organization performing audits, is it appropriate to ask for these permissions? Does granting the privileges contained in the SecurityAudit policy expose a client organization to increased risk of leaking their customers' personally identifiable information \(PII\), or otherwise endanger their compute environment or information assets in the case of irresponsible or malicious use of the permissions by the granted user, i.e. the security auditor?

The set of permissions included in this policy is generally designed to allow a privileged user to access configuration metadata without reading any system or application data, or accessing any credentials that could later be used to access sensitive data or resources. However, several of the permissions granted in the policy do allow a user to view application data in specific scenarios, which might include PII for the customer’s users. For this reason, it is advisable to customize the permissions granted to auditors to a more restricted set. This procedure is documented below, following discussions of the permissions contained in the policy that are considered problematic.

Other than the **Problematic Permissions** detailed below, permissions granted by the SecurityAudit policy are considered safe. Many of them look a bit scary at first glance, as they include a long list of ‘read’ permissions, however these allow only for the reading of metadata. Nonetheless, it is advisable to remove or deny any permissions not specifically required for an audit use case. Further, permissions granted for the purpose of a security audit should be granted only to users who specifically require them, and for the minimum required duration. Permissions should be granted only when needed and revoked immediately when no longer needed, in keeping with the general [Principle of Least Privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege).

## Risk Assessment Rubric

The following questions are to be asked of each permission in the policy:

* Does granting a user a given permission allow them to read and decrypt data from any form of encrypted database, bucket or other datastore?
* Does granting a user a given permission allow them to read logs?
* Does granting a user a given permission allow them to read or send emails or other notifications?
* Does granting a user a given permission allow them to read, write, modify, create or delete any of the following resources:
  * compute instances
  * storage volumes
  * databases, buckets or other datastores
  * load balancers
  * messaging queues
* Does granting a user a given permission allow them to escalate permissions by creating, editing, or granting IAM policies, permissions, roles or group memberships?

## Problematic permissions in the SecurityAudit policy

The following permissions are considered problematic in the sense that the answer to some of the questions in the risk assessment rubric is "yes", at least in specific scenarios.

### **1\) RDS:DownloadDBLogFilePortion**

This action allows a user to download log files for RDS databases. Available log files can be listed with the also-allowed [DescribeDBLogFiles](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_DescribeDBLogFiles.html) action.

Why is this dangerous? By default, RDS retains only error logs for Postgres and MySQL databases. However, query logs can be, and often are, enabled by engineers, as these logs are useful for identifying slow queries and tuning database performance. If query logs are enabled, the raw database query is displayed in the log lines. Depending on application logic, database queries may contain PII for application end users, i.e. a client organization's customers. For example, if a client's website has users fill out a form with their name, address and social security number in order to access their records, the database log would contain a line like this:

```text
LOG:  execute … SELECT(*) FROM records WHERE (social_security_number LIKE YOUR_SOCIAL_SECURITY_NUMBER) AND (full_name LIKE YOUR_NAME) AND (personal_address LIKE YOUR_ADDRESS)
```

### **2\) DataPipeline:EvaluateExpression and QueryObjects**

Both of these permissions allow users to evaluate queries against pipeline data, which could potentially contain PII. This depends on the existence of sensitive data and AWS Data Pipelines operating on it, so it may not pose a risk to all customers, but this permission is better removed or explicitly denied.

### **3\) CloudFormation:GetTemplate**

This allows an auditor to read CloudFormation templates. If developer best practices are followed, CloudFormation templates should not include any compromising credentials, and they will likely never contain user PII. However, a shortcut practice that is unfortunately common among engineers is to place service credentials and other secrets \(for example, AWS IAM access key pairs for service accounts\) directly in CloudFormation templates. If the templates contain secrets, then an auditor with `CloudFormation:GetTemplate` privileges will have access to those secrets, as well as a road-map of where those secrets can be used. This could allow a malicious actor to wreak havoc in a variety of ways, including accessing PII.

## Remediation: denying the problematic permissions

There are two easy ways to remediate the problem of the policy including unsafe, unnecessary permissions:

1. Create a policy which denies the permissions. Add this policy together with Amazon's SecurityAudity policy to an IAM group, and grant permissions to auditors by adding them as IAM users to the group, insuring that the permissions are applied together. In AWS IAM, an [explicit deny overrides any allow statement](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html).
2. Create a custom security audit policy by cloning Amazon’s Security Audit Policy and removing the problematic allow statements. Add this policy to a group, and grant permissions to auditors by adding them as IAM users to the group.

These two solutions produce the same result, and each is described below.

One additional issue raised by the permissions above deserves further consideration: the possibility of secrets included in CloudFormation templates. Rather than removing the `CloudFormation:GetTemplate` permission from the permissions granted to the auditor \(as this permission may be required for legitimate audit-related purposes\), it is advisable that customers audit their CloudFormation templates for secrets, and sanitize and rotate any secrets discovered to have been previously exposed in templates. However, this must be done _before_ granting permissions to the auditor, and removing or denying the permission is a possible temporary stop-gap measure until the remediation of the underlying problem is complete.

As a matter of best practices, ensure that teams using CloudFormation _never_ put secrets directly into templates. Instead, secrets should be are stored securely and under IAM-scoped encryption, for example in AWS Parameter Store using AWS Key Management Service \(KMS\) for encryption, and [interpolated at deploy-time as dynamic references](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html) in templates.

### **Solution 1: Explicitly denying the problematic permissions**

First, create a custom IAM policy to explicitly deny permissions considered dangerous. In AWS IAM, deny rules always take precedence over allow rules. Second, attach this policy together with the Amazon-managed SecurityAudit Policy to an IAM group in order to easily manage permissions for security auditors by adding them to and removing them from the group.

#### **Step 1**: Create a policy which will deny the problematic permissions granted by the Security Audit Policy. This can be done using either the AWS Console or the AWS CLI.

To use the AWS Console:

1. Navigate to the [create IAM policy page](https://console.aws.amazon.com/iam/home?#/policies$new?step=edit).
2. Open **Select a service** and click **Choose a service.**
3. Find **RDS.**
4. Under **Actions**, in the search bar, enter “download”, and select both **DownloadCompleteDBLogFile** and  **DownloadDBLogFilePortion.** \(These can also be found under the list of **Read** actions.\)
5. Click **Switch to deny permissions.**
6. Confirm that the policy segment now displays the red **Deny** symbol in the upper left had corner. ![](https://paper-attachments.dropbox.com/s_A29DBF44CAFEE3FE9F998E521AB34223FBADD39EE55890A92A3D50C794839E0A_1569911555497_Screen+Shot+2019-10-01+at+8.31.31+AM.png)
7. Under **Resources,** select **All resources.**
8. Click **Add additional permissions.**
9. Open **Select a service** and click **Choose a service.**
10. Find **Data Pipeline.**
11. Under **Actions,** in the search bar, enter “query” and select **QueryObjects.** \(This can also be found under the list of **Read** actions.\)
12. In the **Actions** search bar, remove “query” and enter “evaluate”, and select **Evaluate Expressions.** \(This can also be found under the list of **Read** actions.\)
13. Click **Switch to deny permissions.**
14. Confirm that the policy segment now displays the red **Deny** symbol in the upper left had corner. ![](https://paper-attachments.dropbox.com/s_A29DBF44CAFEE3FE9F998E521AB34223FBADD39EE55890A92A3D50C794839E0A_1569911555497_Screen+Shot+2019-10-01+at+8.31.31+AM.png)
15. Click **Review policy.**
16. Enter a name, such as **block-problematic-security-audit-permissions,** and a description, such as “Block problematic security audit permissions”.

To use the AWS CLI:

Pre-requisite: you must have the AWS CLI configured with credentials for an IAM user with “IAM create-policy” permissions.

Copy the below policy JSON to your clipboard:

```javascript
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Deny",
                "Action": [
                    "datapipeline:EvaluateExpression",
                    "datapipeline:QueryObjects",
                    "rds:DownloadDBLogFilePortion",
                    "rds:DownloadCompleteDBLogFile"
                ],
                "Resource": "*"
            }
        ]
    }
```

Run the following CLI command:

```bash
aws iam create-policy \
--policy-name block-problematic-security-audit-permissions \
--policy-document file://<(pbpaste)
```

#### **Step 2:** Create a group with both the **block-problematic-security-audit-permissions** policy created above, and the Security Audit policy.

Access privileges can be safely granted to auditors by adding them to this group, which will add the policies together, blocking the problematic permissions but granting the needed ones.

1. Navigate to the [IAM Groups](https://console.aws.amazon.com/iam/home?#/groups) page.
2. Click **Create New group.**
3. Enter a name, such as "security-auditors”, and click **Next Step.**
4. Under the **Attach Policy** pane, search for and select both the “**block-problematic-security-audit-permissions”** and “**Security Audit”** policies, then click **Next Step.**
5. Review the policies to be attached to the new group, confirming that both the deny policy created above and the Security Audit policy are attached.
6. Click **Create Group.**

### **Solution 2: Creating a more secure CustomSecurityAudit policy without the problematic permissions**

Create a custom IAM policy with a restricted subset of permissions from the Amazon-managed SecurityAudit Policy. Attach this policy to an IAM group in order to easily manage permissions for security auditors by adding them to and removing them from the group.

#### **Step 1:** Create a custom IAM permissions policy

1. As an AWS user with permissions allowing creation of IAM policies, navigate to the [list of IAM policies](https://console.aws.amazon.com/iam/home?#/policies) in the AWS console.
2. Enter “securityaudit” into the search field to find the security audit template.
3. Click the policy name to enter the details page.
4. Click **{} JSON** to display the JSON code for the policy.
5. In a separate browser tab, navigate again to the [policies page](https://console.aws.amazon.com/iam/home?#/policies) in the AWS console.
6. Click **Create policy.**
7. Click the **JSON** tab.
8. Select all and copy the policy JSON from the existing security audit policy to the **JSON** field for the policy being created.
9. Under **Create policy**, switch from the **JSON** tab to the **Visual editor** tab.
10. If your company does not use RDS, remove the **RDS** row. If you do use RDS:

    a. Find and open the **RDS** dropdown tab.

    b. Open the **Actions** tab under **RDS**.

    c. Under **Access level**, open **Read.**

    d. Uncheck **DownloadDBLogFilePortion.**

    e. Close the **RDS** tab.

11. If your company does not use AWS DataPipeline, remove the **DataPipeline** row. If you do use it:

    a. Find and open the **DataPipeline** dropdown tab.

    b. Under **Access level**, open **Read.**

    c. Under **Access level**, open **Read.**

    d. Uncheck **EvaluateExpressions** and **QueryObjects.**

    e. Close the **DataPipeline** tab.

12. Custom policies have a size limit smaller than that for AWS managed policies, so you will need to remove some more permissions in order to create this custom policy. Remove unnecessary permissions, such as those that are not used by your company. For example, if you do not use AWS Internet of Things \(IoT\) service, click remove in the **IoT** and **IoT Greengrass** rows. Most companies also do not use services such as **RoboMaker,** **Comprehend, Gamelift, Sagemaker,** or **Chime,** so those can likely be removed**.** If you do not need **CodeDeploy, CodeCommit,** and **CodeStar,** remove those as well. If you need more permissions than can fit in a single policy, copy the JSON into two separate **Create policy** forms and remove complementary permissions.
13. Click **Review policy.**
14. Enter a name for your policy, such as **CustomSecurityAudit.**
15. Click **Create policy.**

#### **Step 2:** Create an IAM group with the **CustomSecurityAudit** policy.

Permissions can be safely granted to auditors by adding them to this group, which will add only the desired permissions.

1. Navigate to the [IAM Groups](https://console.aws.amazon.com/iam/home?#/groups) page.
2. Click **Create New group.**
3. Enter a name, such as "security-auditors”, and click **Next Step.**
4. Under the **Attach Policy** pane, search for and select the **CustomSecurityAudit** policy, then click **Next Step.**
5. Review the policy to be attached to the new group.
6. Click **Create Group.**

