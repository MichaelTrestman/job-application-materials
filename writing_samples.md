# Technical Writing Samples



* [AWS-managed SecurityAudit IAM Policy risk assessment and remediation](writing_samples.md#aws-managed-securityaudit-iam-policy-risk-assessment-and-remediation)
* [Problem and remediation 1: You aren't monitoring load balancer response for server errors](writing_samples.md#problem-1-you-arent-monitoring-load-balancer-response-for-server-errors)
* [Problem and remediation 2: You aren't storing application and service credentials and other secrets in a secure way.](writing_samples.md#problem-2-you-arent-storing-application-and-service-credentials-and-other-secrets-in-a-secure-way)
* [Kubes for Noobs: An introduction to Kubernetes](writing_samples.md#kubes-for-noobs-an-introduction-to-kubernetes)

### 

### Cloud Operations Problems and Remediations

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

## Kubes for Noobs: An Introduction to Kubernetes

### Summary

This tutorial offers a conceptual introduction to Kubernetes, and then guides the user through a hands-on, beginner-level exercise using Kubernetes and Google Cloud to create a compute cluster and deploy a simple web application.

#### Learning Objectives:

* Understand the advantages of using Kubernetes to manage distributed computing systems.
* Set up the tools for using Kubernetes, on your machine and in Google Cloud.
* Create your first Kubernetes cluster.
* Deploy a sample app and load balancer.
* Route traffic through the load balancer to the application using a Kubernetes service.

#### Outline:

1. The Kubernetes philosophy: a conceptual introduction
2. Getting a handle on Kubernetes: setting up your tools
   1. Set up your Google Cloud
   2. Set up kubectl
3. Cloud Operations with Kubernetes
   1. Create a cluster in the cloud
   2. Deploy a sample app
   3. Route traffic to the application
   4. Test your system
   5. Clean up by destroying your cluster

### The Kubernetes philosophy: a conceptual introduction

Kubernetes is a tool for orchestrating distributed computing systems. In recent years it has been widely adopted for applications both small and large due to its ease of use and its power as a force multiplier in operations, allowing a small number of engineers to deploy, reliably maintain, and seamlessly upgrade distributed computing systems, work that previously would have occupied a much larger team.

In large part the power of Kubernetes to ease operations work lies in the philosophy of _declarative configuration_ which it embodies. Essentially this means that as a user, you command Kubernetes to bring about a particular state of your computing system by describing or “declaring” the desired end state, rather than by issuing a sequence of commands to perform specific operations \(what is known as _imperative configuration_, by contrast\). As an analogy, consider helping your friend navigate to a cafe to meet you for lunch. An _imperative_ way to “configure” their location to the coordinates of the cafe would be to issue a sequence of concrete instructions, such as “walk to the corner of Blah Street and turn left”, “continue down Whatever Avenue for 2 miles”, or “take the next right after the little red school house”. A _declarative_ approach would be to simply _declare_ the address or GPS coordinates of the cafe, and have them use an automated system capable of continuously nudging their position toward the destination from wherever they currently are.

The declarative approach has obvious advantages over the imperative approach. Firstly, the imperative approach is more _fragile_ in that it depends on knowing the person's precise location at the start. If they are not where you think they are when you start issuing commands, they will be rather confused. A recorded sequence of instructions \(for example, a printed out sheet of navigational instructions\) contains no provisions for any unexpected surprises that might cause your friend to have to alter their route. Unless your friend is a capable navigator already familiar with the area, they will be lost if everything doesn't go according to plan. Even a missing street sign could cause them to end up on the other side of town, while you sit at the cafe by yourself.

The declarative approach clearly requires sophisticated technology, in this case a GPS navigation system. But, given that one does have access to such a system, the advantages are great indeed. Kubernetes offers comparable advantages in the domain of distributed computing systems operations. Rather than having to follow a series of specific and independently fragile procedural steps such as “download the binary from [https://whatever.url”](https://whatever.url”), “copy the file into /some/specific/directory”, etc., one simply provides Kubernetes with a _manifest_, a human-readable description of the desired end state of the system. The powerful technology Kubernetes encapsulates then guides the system to the correct state. Similar to an automated GPS navigator, Kubernetes can bring a system to a desired state from a wide range of starting positions, and can correct course along the way if things go wrong.

Declarative configuration has the further benefit that system states are easily _reproducible_. By writing down the simple address of the cafe, your friend can later remember where the two of you had lunch or even recommend the place to their other friends. Similarly, the manifest that one uses to declare a desired state of a system to Kubernetes can later serve as a record of what that state _was_, given that Kubernetes was able to successfully achieve it. Therefore, by saving these declarations in a version control system such as Git, one maintains the ability to _return_ the system to a prior state by commanding Kubernetes to apply the old manifest. In contrast, even if one records the exact steps involved in imperative configuration, it can be extremely difficult to reverse them and return the system to a previous state. Most operations one can perform on a distributed computing system lack anything like a simple 'undo' function.

Another great advantage of Kubernetes is its ability to “self-heal” if the components of your system fail for any reason. Kubernetes does this particularly effectively by using the concept of a _pod_, which is a collection of containerized applications that need to be located together in a single compute environment in order to function \(for example, if they need to share a file system\). If your manifest says there should be 147 replicas of a pod, hence 147 independently well-functioning instances of application, and someone in a data center accidentally unplugs the machine running 20 of them, Kubernetes will quickly detect this disparity and bring the system back to the state described in the manifest by allocating 20 replicas to other available machines. Another great advantage of pods is that they allow for smart, efficient scaling of distributed applications by separating the vertical scaling factor \(the compute resources available\) for specific containers, from the horizontal scaling factor \(the number of instances\) for a collection of colocated containers.

In summary, Kubernetes allows a small team of engineers to easily do what otherwise might be difficult for a much larger team: to precisely guide a distributed system into an understandable and reproducible state, and to reliably maintain that state until they intend to change it.

### Getting a Handle on Kubernetes: setting up our tools

In this section we will prepare the infrastructure and tools to use Kubernetes. We will use Google cloud, the cloud computing environment hosted by Google. It is an easy way to start, although it is also powerful enough to handle enterprise scale industrial computing applications.

#### Set up your Google Cloud

1. Create a Google Cloud account for yourself, if you do not already have one, at [https://cloud.google.com](https://cloud.google.com). Google offers a free trial account with plenty of credits. You will need to enter a credit card to verify your identity, but you will not be charged.
2. Create a project in Google Cloud at [https://console.cloud.google.com/projectcreate](https://console.cloud.google.com/projectcreate). Record the project name or keep the browser window open so you can find it easily.
3. Enable Kubernetes Engine API. Find it by searching at [https://console.cloud.google.com/apis/](https://console.cloud.google.com/apis/) and then clicking 'enable'.
4. Install 'gcloud', the Google Cloud command line interface \(CLI\) tool [https://cloud.google.com/sdk/docs/downloads-interactive](https://cloud.google.com/sdk/docs/downloads-interactive).
5. Open iterm \(on a mac\) or your terminal of choice, and type `gcloud version` to ensure that gcloud is correctly installed.
6. Configure the gcloud CLI to point to your new project by running `gcloud config set project YOUR_PROJECT_NAME`

   **Note**: You can always check which project gcloud is targetting by running `gcloud config get-value project`.

#### Set up kubectl

Kubectl is the command line interface \(CLI\) for Kubernetes. On a mac computer, the easiest way to manage kubectl is using homebrew.

1. In iterm or your terminal of choice, run `brew update` to make sure your local homebrew is up to date and knows about the latest version of kubectl.
2. Run `kubectl` to check whether kubectl is installed. 
   * If it is installed, you should see a summary of the manual page, including a list of top level commands. In this case, run `brew upgrade kubectl` to have homebrew bring your kubectl up to date.
   * If kubectl is not installed, your shell will tell you that the kubectl command cannot be found. In this case, run `brew install kubectl`. Now running the `kubectl` command should give you a summary of the manual page.

### Cloud operations with Kubernetes

#### Part 1: Create a cluster in the cloud

1. \(Optional\) If you prefer to create your cluster in a specific zone, you can target the zone by running `gcloud config set compute/zone PREFERRED_ZONE`, for example, you could use 'us-west1-a' as the value of `PREFERRED_ZONE`.
2. Create a cluster in your Google Cloud by running:

   `gcloud container clusters create YOUR_CLUSTER_NAME`

   The output should confirm that your cluster is up and running \(the output below is for creation of a cluster called 'noobcluster' in a Google Cloud project called 'kubes4noobs'\). This can take a few minutes.

```text
Creating cluster noobcluster in us-west1-a... Cluster is being health-checked (master is

 healthy)...done.

Created [https://container.googleapis.com/v1/projects/kubes4noobs/zones/us-west1-a/clusters/noobcluster].

To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-west1-a/noobcluster?project=kubes4noobs

kubeconfig entry generated for noobcluster.

NAME        LOCATION    MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS

noobcluster  us-west1-a  1.11.7-gke.12   35.197.16.123  n1-standard-1  1.11.7-gke.12  3          RUNNING
```

#### Part 2: Deploy a sample app

Now we will deploy a very simple app with a load balancer in front of a microservice that responds to http requests by saying "hello".

We'll do this by running the `kubectl apply` command, pointing to a manifest that describes a deployment with the pod containing our app. The manifest will look like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  selector:
    matchLabels:
      app: hello
      tier: backend
      track: stable
  replicas: 7
  template:
    metadata:
      labels:
        app: hello
        tier: backend
        track: stable
    spec:
      containers:
        - name: hello
          image: "gcr.io/google-samples/hello-go-gke:1.0"
          ports:
            - name: http
              containerPort: 80
```

The manifest includes metadata and labels that the kubectl CLI uses to find the app in order to execute commands. It also includes the template metadata labels that define the labels for the _pod_ that the app will exist in in our deployment, as well as the selector matchLabels that Kubernetes will use to identify the pod in which to place the app. This is a bit complicated, but for now just be aware that these must match for Kubernetes to find the correct pod which will contain the app. If you are curious, try changing these labels so they don't match and see what happens. Don't worry, nothing terrible will happen.

The `containers` section at the bottom defines a single container that will run your app. We give it a name and tell Kubernetes where to find the container image--in this case we will borrow it from the sample apps provided by Google on its public registry, gcr.io. We also tell it on which port to listen for http requests, in this case the standard default port 80.

Create the deployment by telling Kubernetes to apply the manifest at the given file path, with the command: `kubectl apply -f PATH_TO_MANIFEST`. To fill in "PATH\_TO\_MANIFEST" with the correct path, you can either a\) copy the manifest above into a file on your local file system, and use the path to that file on your local file system, or b\) use the following path where the manifest is hosted by k8s.io as an example: [https://k8s.io/examples/service/access/hello.yaml](https://k8s.io/examples/service/access/hello.yaml).

You should see confirmation that the deployment has been created:

​ `deployment.apps/hello created`

If you ask Kubernetes to list your deployments, you should see it listed:

```bash
:> kubectl get deployments
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello   7         7         7            7           1m
```

You can get detailed information about the deployment by asking Kubernetes to describe it, using the name we defined in the manifest, "hello":

`kubectl describe deployments hello`

The manifest we used to create the deployment asks for seven replicas of the pod containing our application. This is more than we need for this simple example, so let's scale it back by editing the manifest.

Run `kubectl edit deployments hello` to edit the manifest in your default text editor. Under the `spec` key, edit the value of the `replicas` key to be 3, rather than seven. Save and close the file, and Kubernetes will apply your changes.

Alternately, if you have copied the manifest to a local file, you can edit the value of the `replicas` key in this local file, and tell Kubernetes to redeploy by using same command again: `kubectl apply -f PATH_TO_MANIFEST`.

If you run `kubectl describe deployments hello` again, you should see that there are now only 3 replicas.

```bash
:> kubectl describe deployments hello | grep Replicas:
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
```

You can also see that the `Events` section will show when the pod containing your app scaled to seven replicas as originally deployed, and then scaled down to three after you redeployed with the edited manifest:

```bash
:> kubectl describe deployments hello | grep -A4 Events
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  14m    deployment-controller  Scaled up replica set hello-7d7d777c6b to 7
  Normal  ScalingReplicaSet  4m36s  deployment-controller  Scaled down replica set hello-7d7d777c6b to 3
```

#### Part 3: Route traffic to the application

Currently, the application is not exposed to the internet. Do this, we will have Kubernetes create a load balancer with a public internet IP address to which we can make requests using a web browser or the `curl` command. However, first, we need to define a Kubernetes 'service' object to make our app backend discoverable to the load balancer. We will do this by applying the following manifest:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: hello
spec:
  selector:
    app: hello
    tier: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: http
```

Note that the manifest uses the 'app: hello' and 'tier: backend' selectors that we defined in the manifest we used to deploy the app, in order to locate the app.

As before we will command Kubernetes to make the desired changes using `kubernetes apply -f PATH_TO_MANIFEST`. Either a\) copy the manifest above into a file on your local file system, and use the path to the manifest on your local file system, or b\) use the following path where it is hosted by k8s.io as an example: [https://k8s.io/examples/service/access/hello-service.yaml](https://k8s.io/examples/service/access/hello-service.yaml).

Terminal output should confirm that the service has been created.

Finally, we will deploy a simple nginx load balancer and create a corresponding service. We will do this in one swoop by applying the following manifest, which contains one section for the service and one for the deployment of the load balancer itself:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: hello
    tier: frontend
  ports:
  - protocol: "TCP"
    port: 80
    targetPort: 80
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: hello
      tier: frontend
      track: stable
  replicas: 1
  template:
    metadata:
      labels:
        app: hello
        tier: frontend
        track: stable
    spec:
      containers:
      - name: nginx
        image: "gcr.io/google-samples/hello-frontend:1.0"
        lifecycle:
          preStop:
            exec:
              command: ["/usr/sbin/nginx","-s","quit"]
```

Once more, command Kubernetes to make the desired changes using `kubernetes apply -f PATH_TO_MANIFEST`. Either a\) copy the manifest above into a file on your local file system, and use the path to the manifest on your local file system, or b\) use the following path where it is hosted by k8s.io as an example: [https://k8s.io/examples/service/access/frontend.yaml](https://k8s.io/examples/service/access/frontend.yaml).

Terminal output should confirm that both the deployment and the service have been created.

#### Part 4: Test your system

To test the system you have created, make a request through the internet to the public IP address of the load balancer. The load balancer will then route the request to the app container in one of the replicas of your 'hello' pods.

To find this public IP address, run `kubectl get services`, to display basic information about your running services.

You should see both your 'frontend' and 'hello' services listed, as will as a service called 'kubernetes', which is created automatically and used to send commands to Kubernetes from inside the cluster.

**Optional:** if you want to watch Kubernetes self-heal in a rather minor way, run `kubectl delete service kubernetes`. This will indeed delete the `kubernetes` service, as you can confirm by running `kubectl get services` immediately \(it will be gone\). However, in a matter of seconds, Kubernetes will recreate this important service object.

Only the 'frontend' service will have an "external-ip" that can be reached through the internet. When you first check, it may be listed as "pending". Simply take a break to stretch or dance around for a minute or so to get your blood pumping for the excitement to come, then check again.

Once `kubectl get services`, or the more specific command `kubectl get service frontend`, displays an external IP for the frontend service, we are ready to make a request to our application. In the terminal, type: `curl http://YOUR_EXTERNAL_IP`, using the displayed external IP. You should see a simple yet friendly message in return:

```text
{"message":"Hello"}
```

You can also confirm that the app is running by navigating to `http://YOUR_EXTERNAL_IP` in your web browser of choice. Your browser should display the same simple 'hello' message.

#### Part 5: Clean up by destroying your cluster

When you are ready to clean everything up, tell Google Cloud to delete your cluster. If you don't remember the name of your cluster, ask Google Cloud by running `gcloud container clusters list`.

When you are ready, run `gcloud container clusters delete YOUR_CLUSTER_NAME`. This will destroy the deployments and services and associated Kubernetes objects.

To confirm that everything has been deleted, you can return to the Google Cloud console at [https://console.cloud.google.com/](https://console.cloud.google.com/). The console may several minutes \(perhaps up to half an hour\) to update, but eventually it will show that you have no active resources listed under the 'resources' tab. If you click on the 'Billing' tab in the left hand navigation menu, you should see that you have plenty of credits left over from your free trial to experiment with Google Cloud and kubernetes. There are lots of good tutorials to explore at [https://kubernetes.io/docs/tutorials/](https://kubernetes.io/docs/tutorials/).

