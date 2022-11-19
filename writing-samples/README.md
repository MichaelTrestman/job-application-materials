# Writing Samples



## [Some Cloud Operations Problems and Remediations](untitled.md)

The below two writing samples are instructions on how to fix common shortcomings for small organizations with cloud-based compute infrastructure. Each problem has remediation instructions for both Amazon Web Service \(AWS\) and Google Cloud Platform \(GCP\).

[Problem 1: You aren't monitoring load balancer response for server errors](untitled.md#problem-1-you-arent-monitoring-load-balancer-response-for-server-errors)

[Problem 2: You aren't storing application and service credentials and other secrets in a secure way.](untitled.md#problem-2-you-arent-storing-application-and-service-credentials-and-other-secrets-in-a-secure-way)



## [Risk Assessment and Remediation for AWS-managed SecurityAudit IAM Policy](risk-assessment-and-remediation-for-aws-managed-securityaudit-iam-policy.md)

Amazon offers a managed Identity and Access Management \(IAM\) policy, named [**SecurityAudit**](https://console.aws.amazon.com/iam/home?#/policies/arn:aws:iam::aws:policy/SecurityAudit$actionLevelSummary?service=EC2&effect=Allow&section=permissions), designed to grant a security auditor privileges sufficient to perform security auditing functions. However, granting infrastructure access privileges should always be carefully considered, especially when the user would be someone outside of the organization or company. As an organization performing audits, is it appropriate to ask for these permissions? Does granting the privileges contained in the SecurityAudit policy expose a client organization to increased risk of leaking their customers' personally identifiable information \(PII\), or otherwise endanger their compute environment or information assets in the case of irresponsible or malicious use of the permissions by the granted user, i.e. the security auditor?

This document addresses those questions and recommends ways of fine-tuning the security policy to protect against possible threats.

## [Kubes for Noobs: An Introduction to Kubernetes](kubes-for-noobs-an-introduction-to-kubernetes.md)

This tutorial offers a conceptual introduction to Kubernetes, and then guides the user through a hands-on, beginner-level exercise using Kubernetes and Google Cloud to create a compute cluster and deploy a simple web application.






