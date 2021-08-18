# Some Common Cloud Operations Issues and Remediations

The below two writing samples are instructions on how to fix common shortcomings for small organizations with cloud-based compute infrastructure. Each problem has remediation instructions for both Amazon Web Service \(AWS\) and Google Cloud Platform \(GCP\).

#### Problem 1: You aren't monitoring load balancer response for server errors

Server error responses at the load balancer level represent errors that are not handled by your application code and propagate out to the web page or other client using this service, potentially impacting user experience or offering malicious actors information about your system. Even a _low rate_ of such unhandled errors is unacceptable and should be remediated. However, remediation of such errors occurring at a low rate may be low urgency and involve developers fixing a bug or adding error handling code as part of their normal work flow.

On the other hand, a _high rate_ of unhandled server errors may immediately impact the experience of users of your service, violating your service-level agreement \(SLA\). It may also offer an exploitable vector for malicious actors to take your service offline or introduce data corruption. This represents a _high urgency_ problem that should be handled immediately by incident response procedures, which may involve rolling back production code to a version that does not raise the unhandled error, or by pushing a hot fix to handle the errors.

Therefore, best practice will probably be to have not one but two different alerts for server error response:

* One which alerts at very low count threshold and sends out a low-urgency alert, such as a slack message or email to the development team responsible for the service;
* One which alerts at a higher rate threshold and sends out a high-urgency alert, such as creating a PagerDuty Incident to immediately notify on-call first-responders.

**AWS Remediation**

Create alert on your load balancer for server error responses:

1. Navigate to your AWS [Elastic Compute \(EC2\) console](https://github.com/MichaelTrestman/job-application-materials/tree/ce8eae8f844a61047aa49bd970b50caa84b4004f/console.aws.amazon.com/ec2/README.md).
2. Select **Load Balancers** under the **Load Balancing** dropdown on the left side menu.
3. Select the load balancer for which you want to create alerts.
4. In the details pane at the bottom of the page, select the **Monitoring** tab.
5. Click **Create Alarm**.
6. Fill in the alarm condition as:

   **Whenever:** **Minimum** of **HTTP 5XXs** is **&gt;=** YOUR\_THRESHOLD.

7. Click **Create Alarm**.
8. In the **Alarm Created Successfully** modal which opens up, click the name of your alarm to navigate to its details page in the CloudWatch console.
9. Click **Edit.**
10. Click **Next** to skip the metric details, which are already configured.
11. Configure **Notifications:**

    By default, AWS sends CloudWatch alarm notifications a Simple Notification Service \(SNS\) topic called **Default\_CloudWatch\_Alarms\_Topic**.

    However, it is better to create separate notifications topics for separate alarms, for example a high-urgency topic for the high urgency alarm for high rates of 5xx errors reported by the load balancer, and a low-urgency topic for the low urgency alarm for low rates of 5xx errors reported by the load balancer.

12. Under **Select an SNS topic**, check **Create a new topic**.
13. Enter a name appropriate to the topic's function, such as "example-application-elb-5XXs-HIGH-URGENCY".
14. Enter a comma-separated list of the email addresses of subscribers. This can include employees and notifications endpoint, for example email integration endpoints for incident response management services such as PagerDuty.
15. Click **Create topic**.
16. Scroll to the bottom and click **Update alarm.**

**GCP Remediation**

Use Stackdriver monitoring to send server error response alerts for your Google Cloud SQL database instances:

1. Navigate to the \[Stackdriver monitoring dashboard\]\([https://app.google.stackdriver.com](https://app.google.stackdriver.com/)\).
2. Hover over **Alerting** on the left-hand navigation menu and click **Create a Policy**.
3. Click **Add Condition**.
4. Enter a title for the condition, such as "Load Balancer: Server Error Response"
5. Under the **Target** header, click the **Find resource type and metric** search bar, and select first **Google Cloud HTTP/S Load Balancing Rule** and then **Request Count**.
6. Under **Filters** click **+ Add a filter**.
7. Find and select **Response code class**.
8. In the details modal that opens up for the filter, select **500** as the **value**, and leave the default **=** as the operator.
9. If you have multiple Load Balancers and only want to alert on a subset, add additional filters for instance name or ID.

   Note that by default the data is aggregated and charted by _rate_. It may make more sense to monitor _count_, as even a very low non-zero rate of such errors is unacceptable.

   To aggregate by count instead:

   ​ a. click **SHOW ADVANCED OPTIONS**.

   ​ b. Under **Aligner**, which is set to **rate** by default, click the drop down, and select **count** instead.

10. Enter **Configuration details** for the alert, customized to your particular application. For example, you might want to alert if, for **any time series**, the Server Error Response count **is above 1** for **the most recent value**.
11. Click **Save** to save your alert condition.
12. Enter a **Notification channel**, such as emailing your administrator, sending a slack message to your "on call" channel, or creating a PagerDuty incident to notify your on-call team.
13. Enter **Documentation** details describing the alert and including a suggested incident response play, or linking to your organization's on-call incident response playbook or wiki.
14. Enter a policy name.
15. Click the **Save** button.

#### Problem 2: You aren't storing application and service credentials and other secrets in a secure way.

Secrets such as credentials for databases, external services and APIs, RSA keys, and AWS IAM key pairs are essential for modern applications. However, they are potentially dangerous vulnerabilities for exploitation, as malicious actors who gain access to secrets can wreak havoc. Secrets should be stored securely, such that they are accessible only by service roles or users that need them for specific engineering functions. They should also be encrypted with keys that are only accessible for encryption or decryption by service roles and users that specifically need to encrypt or decrypt them.

**AWS Remediation**

Store secrets as encrypted text, aka "Secure Strings" in Amazon Parameter Store. Use Amazon encryption key management service \(KMS\) to encrypt secrets for storage and decrypt them for use. Use Amazon Web Services \(AWS\) Identity Access Management \(IAM\) roles to limit access to secrets and encryption keys to those who need the secrets for job functions.

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

   d. Customize the ARN by adding a **Region** \(e.g. us-east-1\).

   e. Add the name-space as **Fully qualified parameter name**, followed by a wildcard operator. For example, to grant IAM users with this policy access to all parameters in a name space for a team called "Cloud Operations", you could enter `cloud-operations/*`, or to grant access to a name space for production database parameters, you could enter `production-db/*`

6. Close the drop-down next to **Systems Manager**.
7. Click **Add additional permissions**.
8. For service, again select **Systems Manager**.
9. For **Actions**, enter "describe parameters", and check `DescribeParameters` when it comes up.

   **DO NOT** add the ARN restriction as above, or users will not be able to list parameters.

10. Scroll down and click **Review policy**.
11. Enter a name and description. In our first example this might be "cloud-operations-parameter-store-user", and "Grants access to the 'cloud-operations/\*\' namespace in the parameter store". In the second example it might be "production-db-parameter-store-user", and "Grants access to production database params in the 'production-db/\*' namespace in the parameter store."
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

1. Navigate to the [AWS key management service \(KMS\) console](https://console.aws.amazon.com/kms/).
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

**GCP Remediation**

Store secrets as encrypted text in Google Storage buckets. Use encryption keys from Google Cloud Key Management Service \(KMS\) to encrypt secrets for storage and decrypt them for use. Use IAM roles to limit access to the storage buckets and encryption keys to those who need the secrets for job functions, in keeping with the [Principle of Least Privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege).

As a Project owner, grant your company's administrators the Cloud KMS Admin role, so they can manage storage buckets to hold secrets, manage encryption keys, and manage user access to the buckets and encryption keys.

1. As a Project Owner, or other user with the **IAM Security Admin** role, navigate to the [IAM dashboard](https://console.cloud.google.com/iam-admin/). Note that this powerful role should be limited to as few people as possible.
2. Click on the **Add** button.
3. Enter the Google user ID \(gmail account\) of the user to be responsible for managing access to encrypted secrets storage.
4. Under **Role**, enter **Cloud KMS Admin**.
5. Click **Add Another Role.**
6. Select **Storage Admin**.
7. Click **Save**.

Secrets administrators should create Cloud Storage buckets to contain secrets for specific teams or applications as encrypted text. They should also create Cloud KMS encryption/decryption keyrings for use by those specific teams. Access to these storage buckets and keys should be restricted to those who need access to the contained secrets for engineering functions.

Note: IAM roles at the level of keys and key-rings cannot be administered through the console.

As a secrets storage administrator \(a user, as created above, with the **Cloud KMS Admin** and **Storage Admin** roles\), first use the Google storage utilities \(gsutil\) CLI to create and grant access to your secrets bucket, then create and grant access to a KMS symmetric encryption key:

1. If you do not already have it, install the [Google Cloud SDK](https://cloud.google.com/sdk/docs/downloads-interactive>), which will include both CLIs to be used in the following steps.
2. Create a bucket by running `gsutil mb gs://YOUR_SECRETS_BUCKET_NAME/`
3. Give object admin permissions to a user who will make use of the secrets stored in the bucket by granting them the `storage.objectAdmin` by running:

   `gsutil iam ch user:USER_NAME@gmail.com:objectAdmin gs://YOUR_SECRETS_BUCKET_NAME/`

4. Use the Google Cloud CLI to create a KMS keyring:

   `gcloud kms keyrings create YOUR_KEYRING_NAME --location global`

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

As a Google user \(such as the one created above\) with Cloud KMS encryption/decryption privileges for the specific key, and read/write privileges on the particular Google Cloud Storage bucket, use the gcloud CLI to encrypt and push files to the bucket, and pull files from the bucket and decrypt them.

To encrypt a file, for example a YAML file containing a secrets manifest named secrets.yml, run:

```bash
gcloud kms encrypt --location KEYRING_LOCATION \
--keyring KEYRING_NAME --key KEY_NAME \
--plaintext-file .yml \
--ciphertext-file secrets.yml.encrypted
```

To push the encrypted file to the bucket, run:

`gsutil cp secrets.yml.encrypted gs://BUCKET_PATH`

