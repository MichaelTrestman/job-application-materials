# Technical Writing Samples

Michael Trestman

michael.a.trestman@gmail.com



Contents:

- [AWS-managed SecurityAudit IAM Policy risk assessment and remediation](./writing_samples.md#aws-managed-securityaudit-iam-policy-risk-assessment-and-remediation)
- [Problem and remediation 1: You aren't monitoring load balancer response for server errors](./writing_samples.md#problem-you-arent-monitoring-load-balancer-response-for-server-errors)
- [Problem and remediation 2: You aren't storing application and service credentials and other secrets in a secure way.](./writing_samples.md#problem-2-you-arent-storing-application-and-service-credentials-and-other-secrets-in-a-secure-way)



## AWS-managed SecurityAudit IAM Policy risk assessment and remediation

### Introduction

Amazon offers a managed Identity and Access Management (IAM) policy, named [**SecurityAudit**](https://console.aws.amazon.com/iam/home?#/policies/arn:aws:iam::aws:policy/SecurityAudit$actionLevelSummary?service=EC2&effect=Allow&section=permissions), designed to grant a security auditor privileges sufficient to perform security auditing functions. However, granting infrastructure access privileges should always be carefully considered, especially when the user would be someone outside of the organization or company. As an organization performing audits, is it appropriate to ask for these permissions? Does granting the privileges contained in the SecurityAudit policy expose a client organization to increased risk of leaking their customers' personally identifiable information (PII), or otherwise endanger their compute environment or information assets in the case of irresponsible or malicious use of the permissions by the granted user, i.e. the security auditor?

The set of permissions included in this policy is generally designed to allow a privileged user to access configuration metadata without reading any system or application data, or accessing any credentials that could later be used to access sensitive data or resources. However, several of the permissions granted in the policy do allow a user to view application data in specific scenarios, which might include PII for the customer’s users. For this reason, it is advisable to customize the permissions granted to auditors to a more restricted set. This procedure is documented below, following discussions of the permissions contained in the policy that are considered problematic.

Other than the **Problematic Permissions** detailed below, permissions granted by the SecurityAudit policy are considered safe. Many of them look a bit scary at first glance, as they include a long list of ‘read’ permissions, however these allow only for the reading of metadata. Nonetheless, it is advisable to remove or deny any permissions not specifically required for an audit use case. Further, permissions granted for the purpose of a security audit should be granted only to users who specifically require them, and for the minimum required duration. Permissions should be granted only when needed and revoked immediately when no longer needed, in keeping with the general [Principle of Least Privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege). 



### Risk Assessment Rubric

The following questions are to be asked of each permission in the policy:

- Does granting a user a given permission allow them to read and decrypt data from any form of encrypted database, bucket or other datastore?

- Does granting a user a given permission allow them to read logs?

- Does granting a user a given permission allow them to read or send emails or other notifications?

- Does granting a user a given permission allow them to read, write, modify, create or delete any of the following resources:

  - compute instances
  - storage volumes
  - databases, buckets or other datastores
  - load balancers
  - messaging queues

- Does granting a user a given permission allow them to escalate permissions by creating, editing, or granting IAM policies, permissions, roles or group memberships?

    


### Problematic permissions in the SecurityAudit policy

The following permissions are considered problematic in the sense that the answer to some of the questions in the risk assessment rubric is "yes", at least in specific scenarios.



#### 1) RDS:[DownloadDBLogFilePortion](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_DownloadDBLogFilePortion.html)

This action allows a user to download log files for RDS databases. Available log files can be listed with the also-allowed [DescribeDBLogFiles](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_DescribeDBLogFiles.html) action.

Why is this dangerous? By default, RDS retains only error logs for Postgres and MySQL databases. However, query logs can be, and are often are, enabled by engineers, as these logs are useful for identifying slow queries and tuning database performance. If query logs are enabled, the raw database query is displayed in the log lines. Depending on application logic, database queries may contain PII for application end users, i.e. a client organization's customers. For example, if a client's website has users fill out a form with their name, address and social security number in order to access their records, the database log would contain a line like this:


    LOG:  execute … SELECT(*) FROM records WHERE (social_security_number LIKE YOUR_SOCIAL_SECURITY_NUMBER) AND (full_name LIKE YOUR_NAME) AND (personal_address LIKE YOUR_ADDRESS)



#### 2) DataPipeline:[EvaluateExpression](https://docs.aws.amazon.com/datapipeline/latest/APIReference/API_EvaluateExpression.html) and [QueryObjects](https://docs.aws.amazon.com/datapipeline/latest/APIReference/API_QueryObjects.html)

Both of these permissions allow users to evaluate queries against pipeline data, which could potentially contain PII. This depends on the existence of sensitive data and AWS Data Pipelines operating on it, so it may not pose a risk to all customers, but this permission is better removed or explicitly denied.



#### 3) CloudFormation:[GetTemplate](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_GetTemplate.html)

This allows an auditor to read CloudFormation templates. If developer best practices are followed, CloudFormation templates should not include any compromising credentials, and they will likely never contain user PII. However, a shortcut practice that is unfortunately common among engineers is to place service credentials and other secrets (for example, AWS IAM access key pairs for service accounts) directly in CloudFormation templates. If the templates contain secrets, then an auditor with `CloudFormation:GetTemplate` privileges will have access to those secrets, as well as a road-map of where those secrets can be used. This could allow a malicious actor to wreak havoc in a variety of ways, including accessing PII.



### Remediation: denying the problematic permissions

There are two easy ways to remediate the problem of the policy including unsafe, unnecessary permissions:

1. Create a policy which denies the permissions. Add this policy together with Amazon's SecurityAudity policy to an IAM group, and grant permissions to auditors by adding them as IAM users to the group, insuring that the permissions are applied together. In AWS IAM, an [explicit deny overrides any allow statement](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html).
2. Create a custom security audit policy by cloning Amazon’s Security Audit Policy and removing the problematic allow statements. Add this policy to a group, and grant permissions to auditors by adding them as IAM users to the group.

These two solutions produce the same result, and each is described below.

One additional issue raised by the permissions above deserves further consideration: the possibility of secrets included in CloudFormation templates. Rather than removing the `CloudFormation:GetTemplate` permission from the permissions granted to the auditor (as this permission may be required for legitimate audit-related purposes), it is advisable that customers audit their CloudFormation templates for secrets, and sanitize and rotate any secrets discovered to have been previously exposed in templates. However, this must be done *before* granting permissions to the auditor, and removing or denying the permission is a possible temporary stop-gap measure until the remediation of the underlying problem is complete.

As a matter of best practices, ensure that teams using CloudFormation *never* put secrets directly into templates. Instead, secrets should be are stored securely and under IAM-scoped encryption, for example in AWS Parameter Store using AWS Key Management Service (KMS) for encryption, and [interpolated at deploy-time as dynamic references](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html) in templates.



#### Solution 1: Explicitly denying the problematic permissions

First, create a custom IAM policy to explicitly deny permissions considered dangerous. In AWS IAM, deny rules always take precedence over allow rules. Second, attach this policy together with the Amazon-managed SecurityAudit Policy to an IAM group in order to easily manage permissions for security auditors by adding them to and removing them from the group.



**Step 1**: Create a policy which will deny the problematic permissions granted by the Security Audit Policy. This can be done using either the AWS Console or the AWS CLI.



To use the AWS Console:

1. Navigate to the [create IAM policy page](https://console.aws.amazon.com/iam/home?#/policies$new?step=edit).
2. Open **Select a service** and click **Choose a service.**
3. Find **RDS.**
4. Under **Actions**, in the search bar, enter “download”, and select both **DownloadCompleteDBLogFile** and  **DownloadDBLogFilePortion.** (These can also be found under the list of **Read** actions.)
5. Click **Switch to deny permissions.**
6. Confirm that the policy segment now displays the red **Deny** symbol in the upper left had corner.
   ![](https://paper-attachments.dropbox.com/s_A29DBF44CAFEE3FE9F998E521AB34223FBADD39EE55890A92A3D50C794839E0A_1569911555497_Screen+Shot+2019-10-01+at+8.31.31+AM.png)

7. Under **Resources,** select **All resources.**
8. Click **Add additional permissions.**
9. Open **Select a service** and click **Choose a service.**
10. Find **Data Pipeline.**
11. Under **Actions,** in the search bar, enter “query” and select **QueryObjects.** (This can also be found under the list of **Read** actions.)
12. In the **Actions** search bar, remove “query” and enter “evaluate”, and select **Evaluate Expressions.** (This can also be found under the list of **Read** actions.)
13. Click **Switch to deny permissions.**
14. Confirm that the policy segment now displays the red **Deny** symbol in the upper left had corner.
    ![](https://paper-attachments.dropbox.com/s_A29DBF44CAFEE3FE9F998E521AB34223FBADD39EE55890A92A3D50C794839E0A_1569911555497_Screen+Shot+2019-10-01+at+8.31.31+AM.png)

15. Click **Review policy.**
16. Enter a name, such as **block-problematic-security-audit-permissions,** and a description, such as “Block problematic security audit permissions”.



To use the AWS CLI:

Pre-requisite: you must have the AWS CLI configured with credentials for an IAM user with  “IAM create-policy” permissions.

Copy the below policy JSON to your clipboard:

```json
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

**Step 2:** Create a group with both the **block-problematic-security-audit-permissions** policy created above, and the Security Audit policy. Access privileges can be safely granted to auditors by adding them to this group, which will add the policies together, blocking the problematic permissions but granting the needed ones.


1. Navigate to the [IAM Groups](https://console.aws.amazon.com/iam/home?#/groups) page.
2. Click **Create New group.**
3. Enter a name, such as "security-auditors”, and click **Next Step.**
4. Under the **Attach Policy** pane, search for and select both the “**block-problematic-security-audit-permissions”** and “**Security Audit”** policies, then click **Next Step.**
5. Review the policies to be attached to the new group, confirming that both the deny policy created above and the Security Audit policy are attached.
6. Click **Create Group.**



#### Solution 2: Creating a more secure CustomSecurityAudit policy without the problematic permissions

Create a custom IAM policy with a restricted subset of permissions from the Amazon-managed SecurityAudit Policy. Attach this policy to an IAM group in order to easily manage permissions for security auditors by adding them to and removing them from the group.



**Step 1:** Create a custom IAM permissions policy:


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
12. Custom policies have a size limit smaller than that for AWS managed policies, so you will need to remove some more permissions in order to create this custom policy. Remove unnecessary permissions, such as those that are not used by your company. For example, if you do not use AWS Internet of Things (IoT) service, click remove in the **IoT** and **IoT Greengrass** rows. Most companies also do not use services such as **RoboMaker,** **Comprehend, Gamelift, Sagemaker,** or **Chime,** so those can likely be removed**.** If you do not need **CodeDeploy, CodeCommit,** and **CodeStar,** remove those as well. If you need more permissions than can fit in a single policy, copy the JSON into two separate **Create policy** forms and remove complementary permissions.
13. Click **Review policy.**
14. Enter a name for your policy, such as **CustomSecurityAudit.**
15. Click **Create policy.**



**Step 2:** Create an IAM group with the **CustomSecurityAudit** policy. Permissions can be safely granted to auditors by adding them to this group, which will add only the desired permissions.


1. Navigate to the [IAM Groups](https://console.aws.amazon.com/iam/home?#/groups) page.
2. Click **Create New group.**
3. Enter a name, such as "security-auditors”, and click **Next Step.**
4. Under the **Attach Policy** pane, search for and select the **CustomSecurityAudit** policy, then click **Next Step.**
5. Review the policy to be attached to the new group.
6. Click **Create Group.**





## Cloud Operations Problems and Remediations

The below two writing samples are instructions on how to fix common shortcomings for small organizations with cloud-based compute infrastructure. Each problem has remediation instructions for both Amazon Web Service (AWS) and Google Cloud Platform (GCP).



### Problem 1: You aren't monitoring load balancer response for server errors

Server error responses at the load balancer level represent errors that are not handled by your application code and propagate out to the web page or other client using this service, potentially impacting user experience or offering malicious actors information about your system. Even a *low rate* of such unhandled errors is unacceptable and should be remediated. However, remediation of such errors occurring at a low rate may be low urgency and involve developers fixing a bug or adding error handling code as part of their normal work flow.

On the other hand, a *high rate* of unhandled server errors may immediately impact the experience of users of your service, violating your service-level agreement (SLA). It may also offer an exploitable vector for malicious actors to take your service offline or introduce data corruption. This represents a *high urgency* problem that should be handled immediately by incident response procedures, which may involve rolling back production code to a version that does not raise the unhandled error, or by pushing a hot fix to handle the errors.

Therefore, best practice will probably be to have not one but two different alerts for server error response:

- One which alerts at very low count threshold and sends out a low-urgency alert, such as a slack message or email to the development team responsible for the service;

- One which alerts at a higher rate threshold and sends out a high-urgency alert, such as creating a PagerDuty Incident to immediately notify on-call first-responders.

  

#### AWS Remediation

Create alert on your load balancer for server error responses:

1. Navigate to your AWS [Elastic Compute (EC2) console](console.aws.amazon.com/ec2/).

2. Select **Load Balancers** under the **Load Balancing** dropdown  on the left side menu.

3. Select the load balancer for which you want to create alerts.

4. In the details pane at the bottom of the page, select the **Monitoring** tab.

5. Click **Create Alarm**.

6. Fill in the alarm condition as:

   **Whenever:** **Minimum** of **HTTP 5XXs** is **>=** YOUR_THRESHOLD.

7. Click **Create Alarm**.

8. In the **Alarm Created Successfully** modal which opens up, click the name of your alarm to navigate to its details page in the CloudWatch console. 

9. Click **Edit.**

10. Click **Next** to skip the metric details, which are already configured.

11. Configure **Notifications:**

    By default, AWS sends CloudWatch alarm notifications a Simple Notification Service (SNS) topic called **Default_CloudWatch_Alarms_Topic**.

    However, it is better to create separate notifications topics for separate alarms, for example a high-urgency topic for the high urgency alarm for high rates of 5xx errors reported by the load balancer, and a low-urgency topic for the low urgency alarm for low rates of 5xx errors reported by the load balancer.

12. Under **Select an SNS topic**, check  **Create a new topic**.

13. Enter a name approprate to the topic's function, such as "example-application-elb-5XXs-HIGH-URGENCY".

14. Enter a comma-separated list of the email addresses of subscribers. This can include employees and notifications endpoint, for example email integration endpoints for incident response management services such as PagerDuty.

15. Click **Create topic**.

16. Scroll to the bottom and click **Update alarm.**



#### GCP Remediation

Use Stackdriver monitoring to send server error response alerts for your Google Cloud SQL database instances:

1. Navigate to the  [Stackdriver monitoring dashboard]([https://app.google.stackdriver.com](https://app.google.stackdriver.com/)).

2. Hover over **Alerting** on the left-hand navigation menu and click **Create a Policy**.

3. Click **Add Condition**.

4. Enter a title for the condition, such as "Load Balancer: Server Error Response"

5. Under the **Target** header, click the **Find resource type and metric** search bar, and select first **Google Cloud HTTP/S Load Balancing Rule** and then **Request Count**.

6. Under **Filters** click **+ Add a filter**.

7. Find and select **Response code class**.

8. In the details modal that opens up for the filter, select **500** as the **value**, and leave the default **=** as the operator.

9. If you have multiple Load Balancers and only want to alert on a subset, add additional filters for instance name or ID.

   Note that by default the data is aggregated and charted by *rate*. It may make more sense to monitor *count*, as even a very low non-zero rate of such errors is unacceptable.

   To aggregate by count instead:

   ​	a. click **SHOW ADVANCED OPTIONS**.

   ​	b. Under **Aligner**, which is set to **rate** by default, click the drop down, and select **count** instead.

10. Enter **Configuration details** for the alert, customized to your particular application. For example, you might want to alert if, for **any time series**, the Server Error Response count **is above 1** for **the most recent value**.

11. Click **Save** to save your alert condition.

12. Enter a **Notification channel**, such as emailing your administrator, sending a slack message to your "on call" channel, or creating a PagerDuty incident to notify your on-call team.

13. Enter **Documentation** details describing the alert and including a suggested incident response play, or linking to your organization's on-call incident response playbook or wiki.

14. Enter a policy name.

15. Click the **Save** button.



### Problem 2: You aren't storing application and service credentials and other secrets in a secure way.



Secrets such as credentials for databases, external services and APIs, RSA keys, and AWS IAM key pairs are essential for modern applications. However, they are potentially dangerous vulnerabilities for exploitation, as malicious actors who gain access to secrets can wreak havoc. Secrets should be stored securely, such that they are accessible only by service roles or users that need them for specific engineering functions. They should also be encrypted with keys that are only accessible for encryption or decryption by service roles and users that specifically need to encrypt or decrypt them.



#### AWS Remediation

Store secrets as encrypted text, aka "Secure Strings" in Amazon Parameter Store. Use Amazon encryption key management service (KMS) to encrypt secrets for storage and decrypt them for use. Use Amazon Web Services (AWS) Identity Access Management (IAM) roles to limit access to secrets and encryption keys to those who need the secrets for job functions.

As an IAM administrator, allow access to encrypted secrets for select users by creating an IAM policy for name-spaced access to the Parameter Store. Attach this policy to a group and add users to the group as needed, always keeping access as restricted as possible, in keeping with the [Principle of Least Privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege).

To create an IAM policy for Parameter Store access:

1. Navigate to the [AWS IAM console policies page.](https://console.aws.amazon.com/iam/home#/policies).

2. Click **Create policy**.

3. Open the **Service** drop-down, and find **Systems Manager**, which includes the Parameter Store.

4. Open the **Actions** dropdown.

   a. Under **Read**, check: `GetParameter`, `GetParameterHistory`, `GetParameters`, and `GetParameterByPath`.

   b. Under **Tagging,** check: `PutParameter`.

   c. Under **Write,** check: `DeleteParameter`.

5. To restrict access under this policy to a namespace in the Parameter Store:

   a. click **Resources**.

   b. Select **Specific**.

   c. Click **Add ARN** to add the Amazon Resource Name.

   d. Customize the ARN by adding a **Region** (e.g. us-east-1).

   e. Add the name-space as **Fully qualified parameter name**, followed by a wildcard operator. For example, to grant IAM users with this policy access to all parameters in a name space for a team called "Cloud Operations", you could enter `cloud-operations/*`, or to grant access to a name space for production database parameters, you could enter `production-db/*`

6. Close the drop-down next to **Systems Manager**.

7. Click **Add additional permissions**.

8. For service, again select **Systems Manager**.

9. For **Actions**, enter "describe parameters", and check `DescribeParameters` when it comes up.

   **DO NOT** add the ARN restriction as above, or users will not be able to list parameters.

10. Scroll down and click **Review policy**.

11. Enter a name and description. In our first example this might be "cloud-operations-parameter-store-user", and "Grants access to the 'cloud-operations/\*\' namespace in the parameter store". In the second example it might be "production-db-parameter-store-user", and "Grants access to production database params in the 'production-db/*' namespace in the parameter store."  

12. Click **Create policy**.



Attach the policy to a group to manage access:

**Either** attach the policy to an existing group:

1. Navigate to the [policies page](https://console.aws.amazon.com/iam/home?#/policies).
2. Select the proper policy and then click **Attach** from the **Policy actions** dropdown.
3. Search for the group, select it and click **Attach policy**.

**Or** attach it to a new group:

1. Navigate to the [IAM Groups page](https://console.aws.amazon.com/iam/home#/groups), and add the policy to a group, for example a "Cloud Operations Team" group or a "Production Database Operator" group:

2. Click **Create New Group**.

3. Enter the group name and click **Next Step.**

4. Attach the appropriate policy and click **Next Step.**

5. Review and click **Create Group**.

   

Add users to the group to grant them access:

1. Navigate to the [groups page](https://console.aws.amazon.com/iam/home?#/groups).
2. Select the proper group and then click **Add Users to a Group** from the **Group Actions** dropdown.
3. Select the users you want to add to the group and then click **Add Users**.



Create an encryption key in KMS and grant access to select Users:

NOTE: Unfortunately, key usage permissions must be added directly to AWS Users or Roles rather than groups.

1. Navigate to the [AWS key management service (KMS) console](https://console.aws.amazon.com/kms/).
2. Click **Create key.**
3. Enter a name and description for the key, such as "cloud-ops-params-encryption-key" and "For encrypting and decrypting secrets for the Cloud Operations Team".
4. Click **Next.**
5. Add any desired tags and click **Next**.
6. Under **Define key administrative permissions**, add users who will have admin permissions for this key, including the ability to add and revoke encryption/decryption access for the key. Restrict admin permissions as much as possible.
7. Under **Key Deletion**, uncheck **allow key administrators to delete this key.**
8. Click **Next.**
9. Add AWS Users that will need access to the encrypted secrets in the namespace of the parameter store that will make use of this encryption key. In this example, that would be members of the Cloud Operations Team.
10. Review the key policy and click **Finish**.



As an authorized user, to put encrypted secrets to the parameter store in the appropriate namespace, run:

```bash
aws ssm put-parameter \
--name /cloud-operations/team-mascot \
--value "honey badger"
--key-id KSM_KEY_ID
--type SecureString
```



To fetch and decrypt secrets, run:

```bash
aws ssm get-parameter \
--name /cloud-operations/most-secret-of-secrets \
--with-decryption
```



#### GCP Remediation

Store secrets as encrypted text in Google Storage buckets. Use encryption keys from Google Cloud Key Management Service (KMS) to encrypt secrets for storage and decrypt them for use. Use IAM roles to limit access to the storage buckets and encryption keys to those who need the secrets for job functions, in keeping with the [Principle of Least Privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege).

As a Project owner, grant your company's administrators the Cloud KMS Admin role, so they can manage storage buckets to hold secrets, manage encryption keys, and manage user access to the buckets and encryption keys.

1. As a Project Owner, or other user with the **IAM Security Admin** role, navigate to the [IAM dashboard](https://console.cloud.google.com/iam-admin/). Note that this powerful role should be limited to as few people as possible.

2. Click on the **Add** button.

3. Enter the Google user ID (gmail account) of the user to be responsible for managing access to encrypted secrets storage.

4. Under **Role**, enter **Cloud KMS Admin**.

5. Click **Add Another Role.** 

6. Select **Storage Admin**.

7. Click **Save**.

   

Secrets administrators should create Cloud Storage buckets to contain secrets for specific teams or applications as encrypted text. They should also create Cloud KMS encryption/decryption keyrings for use by those specific teams. Access to these storage buckets and keys should be restricted to those who need access to the contained secrets for engineering functions.

Note: IAM roles at the level of keys and key-rings cannot be administered through the console.

As a secrets storage administrator (a user, as created above, with the **Cloud KMS Admin** and **Storage Admin** roles), first use the Google storage utilities (gsutil) CLI to create and grant access to your secrets bucket, then create and grant access to a KMS symmetric encryption key:

1. If you do not already have it, install the [Google cloud SDK](<https://cloud.google.com/sdk/docs/downloads-interactive>), which will include both CLIs to be used in the following steps.

2. Create a bucket by running `gsutil mb gs://YOUR_SECRETS_BUCKET_NAME/`

3. Give object admin permissions to a user who will make use of the secrets stored in the bucket by granting them the `storage.objectAdmin` by running:

   `gsutil iam ch  user:USER_NAME@gmail.com:objectAdmin  gs://YOUR_SECRETS_BUCKET_NAME/`

4. Use the Google cloud CLI to create a KMS keyring:

   `gcloud kms keyrings create YOUR_KEYRING_NAME  --location global` 

5. Then create a key:

	```bash
	gcloud kms keys create YOUR_KEY_NAME \
	--keyring YOUR_KEYRING_NAME \
	--purpose encryption \
	--location global
	```


6. Add the encrypter/decrypter permissions role for the user who will encrypt and decrypt secrets with the key:
	```bash
   gcloud kms keys add-iam-policy-binding YOUR_KEY_NAME \
   --keyring YOUR_KEYRING_NAME \
   --location global \
   --member user:USER_NAME@gmail.com \
   --role roles/cloudkms.cryptoKeyEncrypterDecrypter 
	```

7. View permissions for the key by running:

	```bash
	gcloud kms keys get-iam-policy YOUR_KEY_NAME \
	--keyring YOUR_KEYRING_NAME \
	--location global 
	```



As a Google user (such as the one created above) with Cloud KMS encryption/decryption privileges for the specific key, and read/write privileges on the particular Google Cloud Storage bucket, use the gcloud CLI to encrypt and push files to the bucket, and pull files from the bucket and decrypt them.

To encrypt a file, for example a YAML file containing a secrets manifest named secrets.yml, run:
```bash
gcloud kms encrypt --location KEYRING_LOCATION \
--keyring KEYRING_NAME --key KEY_NAME \
--plaintext-file .yml \
--ciphertext-file secrets.yml.encrypted
```

To push the encrypted file to the bucket, run:

`gsutil cp secrets.yml.encrypted gs://BUCKET_PATH`