---
layout: post
title: "Building a Practical Privileged Access Management Architecture with Okta, AWS IAM, HashiCorp Vault, and Wazuh"
description: "A practical implementation of privileged access management using federated identity, temporary AWS roles, HashiCorp Vault, private networking, and Wazuh security monitoring."
date: 2026-09-09
categories: [cybersecurity, pam, cloud-security, aws, identity]
---

*By Captain CSEC, September 09, 2026*

**Readers who want to see this blog by Hackatron on Medium should [click here](https://medium.com/@highroller039/cybersecurity-in-aerospace-safeguarding-the-future-of-flight-and-space-24abedf0a87a).**

![Cyber Aviation Banner](/images/title_image.png)


Privileged access is one of those security problems that looks simple until you try to implement it properly.
An administrator needs access to a sensitive system. They authenticate, receive permissions, perform an administrative task, and leave. On paper, that sounds straightforward. In a real environment, however, several questions immediately appear:
Who authenticated?
How was their identity established?
Which AWS role did they receive?
Why were they given privileged access?
Which secrets were exposed?
Where was that access allowed?
What happened during the privileged operation?
And, perhaps most importantly, can the entire sequence be reconstructed afterward?
This project was built around those questions.
The implementation combines Okta, AWS IAM Identity Center, AWS IAM, HashiCorp Vault, private AWS networking, a bastion host, and Wazuh into a practical privileged-access security architecture.
The goal was not simply to deploy individual security products. The goal was to demonstrate how identity, authorization, privileged credentials, network isolation, monitoring, and audit evidence fit together as one security control plane.
![Architecture Diagram](/images/iam-pam-wazuh-architecture.png.png)
*Figure 1: Privileged Access Management Architecture*
________________________________________

1. Start with Identity, Not the Privileged Account
The architecture begins with the enterprise identity layer.
An operator authenticates through Okta, providing the initial identity assurance and MFA control. Okta represents the human identity layer; AWS should not need to become the system responsible for managing that human identity independently.
The identity then reaches AWS IAM Identity Center, which provides the federation boundary into AWS.
From there, AWS issues temporary access through an IAM role.
The important concept here is separation of responsibilities:
![](/images/1.png)
This is fundamentally different from giving an administrator a permanent AWS access key.
A long-lived credential creates a persistent authorization path. Federated temporary role access creates a much narrower window in which the identity can operate with the permissions assigned to that role.
For a junior engineer, the simplest way to think about this is:
Okta answers "Who are you?"
IAM answers "What are you allowed to do?"
Those are related questions, but they are not the same control.

![Okta](/images/okta_user.png)

![AWS](/images/okta_user_admin_thru_to_aws.png)
*Figure 2 & 3: Alice Admin using Okta to get thru to AWS*
________________________________________

2. Privileged Access Needs Another Control Layer
Normal application access and privileged access should not be treated identically.
A privileged operator may need access to infrastructure, secrets, administrative systems, or sensitive configuration. That means the architecture needs controls beyond ordinary authentication.
The project therefore models the privileged-access sequence as:
![](/images/2.png)
The PAM layer represents the security decision surrounding privileged access.
In a mature privileged-access architecture, this layer can incorporate approval, business justification, role-based authorization, Just-In-Time access, credential controls, session monitoring, and expiration.
In this implementation, the concrete privileged credential-management component was HashiCorp Vault.
That distinction matters.
The project does not claim to have implemented tools that were not actually deployed. Instead, it demonstrates the underlying PAM security principles using the tools available in the environment.

![RBAC](/images/Bob_Developer_confederated_via_Okta_started_EC2_successfully.png)

![Least Privelege](/images/Charlie_Auditor_confederated_via_Okta_denied_EC2_access.png)
*Figure 3 & 4: Bob Developer confederated via Okta and IAM Identity Center - RBAC (shown above) Successfully Start EC2 Due to Permitted Temporary Access while Charlie Auditor confederated via Okta and IAM Identity Center Failed to Start EC2 (Least Privilege)*
________________________________________

3. HashiCorp Vault Becomes the Privileged Credential Boundary
The Vault deployment was deliberately separated from the general workload environment.
A dedicated PAM/Vault VPC was created, with the Vault system placed inside a private subnet. The Vault EC2 instance had no public IP address.
This is an important architectural decision.
A secrets-management system should not require direct Internet exposure merely because administrators need to manage it.
The resulting trust boundary looks approximately like:
![](/images/3.png)
Connectivity was established using private AWS networking and VPC peering between the relevant environments.
Security-group rules were kept focused on the required administrative path rather than exposing Vault broadly.
This creates a useful principle for anyone reproducing the architecture:
Protect the secret store with network controls before relying on application-level authorization.
You want multiple independent controls.

![Vault Policy Example](/images/vault_policy_example.png)
*Figure 5: Vault Policy Example - Vault ssh-ed in via Bastion using Peer-to-Peer*
________________________________________

4. Vault Policy Turns the Secret Store into an Authorization Boundary
Deploying Vault is only half the problem.
A secrets engine without carefully defined policies simply becomes another place where sensitive information can be stored.
The project used Vault's KV secrets engine and created a PAM-specific policy controlling access to the PAM secret namespace.
The policy covered the protected paths under:
secret/data/pam/*
and the associated metadata:
secret/metadata/pam/*
The resulting model was:
![](/images/4.png)
This is where the distinction between authentication and authorization becomes especially important.
Authentication establishes the identity.
The Vault policy determines what that identity can actually access.
The test privileged record provided concrete evidence that the system was managing a versioned privileged-access object rather than simply storing an arbitrary file or password.
The evidence showed the secret at:
secret/pam/test-privileged
with a current KV version of 4 (_see Figure 5 above_).
That version history is useful because privileged-access systems need more than a binary "secret exists" state.
They need traceability.
________________________________________

5. Version History Provides Another Audit Dimension
Vault KV versioning provided evidence that the privileged record had undergone multiple tracked changes.
The metadata showed four versions, with creation and update information associated with the different operations.
That gives an investigator another useful question to answer:
What changed, and when?
Versioning should not be confused with a complete security audit trail. They serve different purposes.
Version history describes the lifecycle of the stored secret.
Audit logging describes activity performed against Vault.
Together they provide significantly stronger evidence.
Conceptually:
![](/images/5.png)
This separation is an important implementation lesson.
________________________________________

6. Audit Logging Turns Activity into Evidence
The Vault deployment had an audit device configured using the file backend.
The audit configuration pointed to:
/opt/vault/log/audit.log
This was then queried directly from the EC2 environment.
The resulting audit records demonstrated that Vault was recording API activity, including operations against the PAM secret and metadata paths.
One of the captured events showed an authorized read operation against:
secret/metadata/pam/test-privileged
with the request marked as allowed.
That is considerably more useful than simply taking a screenshot saying "Vault is running."
A good security portfolio should demonstrate behavior, not just configuration.
There is a major difference between:
Vault is configured.
and:
Vault recorded an authorized operation against a protected PAM resource.
The second statement is evidence of an actual control operating.

![Vault log Example](/images/vault_audit_log_example.png)
*Figure 6: Vault Audit Log Example - Vault ssh-ed in via Bastion using Peer-to-Peer*
________________________________________

7. The Bastion Is an Administrative Gateway, Not a Shortcut Around Security
The administrative gateway provides the controlled operational path into the private environment.
Rather than making Vault directly accessible from the public Internet, administrative connectivity passes through the bastion.
The resulting network relationship is:
![](/images/6.png)
The bastion also became an important monitoring point.
A Wazuh agent was deployed on the bastion so that activity occurring at this administrative boundary could feed into the security-monitoring layer.
This is an important distinction:
The Vault EC2 instance was not treated as the Wazuh agent.
The Wazuh agent was placed on the bastion endpoint.
That allows the architecture to monitor the administrative endpoint without incorrectly representing the Vault service itself as a monitored Wazuh agent.
________________________________________

8. Wazuh Adds Detection and Visibility
The architecture adds Wazuh as the security-monitoring layer.
The Wazuh deployment consists of the core components:
![](/images/7.png)
The bastion hosts the Wazuh agent:
![](/images/8.png)
The agent provides endpoint telemetry that can be processed by the Wazuh Manager and subsequently indexed and presented through the dashboard.
This changes the architecture from simply being a PAM/secret-management environment into a security-monitoring environment.
Vault answers questions around protected secrets and access policy.
Wazuh provides another layer of security visibility around the infrastructure and endpoint.
________________________________________

9. Why the Wazuh Network Path Matters
The Wazuh communication paths were explicitly represented in the architecture.
Agent telemetry uses TCP 1514, while agent enrollment uses TCP 1515.
The architecture therefore separates the monitoring relationship from the administrative relationship.
The conceptual flow is:
![](/images/9.png)
The Wazuh components themselves were kept private rather than being exposed as public-facing services.
This follows the same principle used for Vault:
Security infrastructure should have a smaller attack surface than the systems it protects.
________________________________________

10. Temporary Internet Access Is Not Permanent Architecture
One practical challenge during deployment was software installation.
Private EC2 instances may need temporary outbound Internet access to retrieve packages or installation dependencies.
The architecture therefore represented a temporary installation egress path:
![](/images/10.png)
This path was specifically treated as temporary.
After installation, it was removed rather than becoming a permanent requirement for Wazuh's internal monitoring architecture.
That distinction is important in infrastructure design.
A deployment dependency should not automatically become a permanent production dependency.
A good question to ask after every installation is:
Does this connectivity still need to exist after the software is installed?
If the answer is no, remove it.
________________________________________
11. The Complete Security Story
Putting the major components together gives the following architecture:
![](/images/11.png)
Around this sits the audit and monitoring layer.
AWS identity and role activity provides one source of evidence.
Vault provides secret-management policy, version history, and audit records.
Wazuh provides endpoint and security-monitoring visibility.
The controls therefore reinforce each other instead of existing as isolated products.
________________________________________

12. What a Privileged-Access Event Should Look Like
The most useful way to understand the implementation is to follow one hypothetical administrative operation.
An operator begins with an authenticated enterprise identity.
Okta establishes the identity and MFA context.
AWS IAM Identity Center federates that identity into AWS.
The operator receives temporary IAM role access.
A privileged operation requires access to a protected resource.
The request enters the privileged-access workflow.
PAM controls determine whether the requested privilege is appropriate.
Vault acts as the protected credential and secret-management boundary.
The administrative path goes through the bastion into the private environment.
The operation produces security telemetry and audit records.
After the privileged activity is complete, temporary access can expire or be revoked according to the access-control model.
The important point is that the security story does not end when the administrator gets access.
It continues through the operation and into the audit layer.
That is the difference between simply implementing access control and implementing an auditable privileged-access architecture.
________________________________________
13. Validation Was Treated as Part of the Implementation
The final phase was not simply "everything appears to work."
The architecture was validated through evidence covering several control categories.
The Vault privileged-access record was retrieved successfully.
The Vault policy was inspected to verify the configured permissions.
The KV metadata was inspected to verify version history.
The Vault audit device was confirmed as enabled.
The audit log was queried to locate activity involving the protected PAM resource.
The private infrastructure and administrative paths were also validated.
These checks provide a much stronger portfolio story than a collection of installation screenshots.
Each screenshot should answer a question.
For example:
Can the privileged record be retrieved?
The Vault KV evidence answers that.
What permissions control it?
The Vault policy evidence answers that.
Is the secret versioned?
The KV metadata evidence answers that.
Is activity being audited?
The Vault audit configuration and audit-event evidence answer that.
Is the infrastructure monitored?
The Wazuh evidence answers that.
This is the mindset I would recommend when building security portfolios:
Do not collect screenshots because they look technical. Collect screenshots because they prove a control.
________________________________________

14. What This Architecture Demonstrates
The project demonstrates several core security-engineering principles.
Identity Federation
Okta and AWS IAM Identity Center establish a centralized identity path instead of relying on unmanaged long-lived AWS credentials.
Temporary Authorization
AWS IAM roles provide temporary access rather than requiring permanent credentials for administrative activity.
Least Privilege
Vault policies restrict access to defined PAM secret and metadata paths.
Privileged Credential Protection
Sensitive privileged-access information is stored inside a dedicated Vault environment rather than being left directly on administrative hosts.
Network Isolation
Vault and Wazuh were deployed within private AWS networking, with controlled administrative connectivity.
Just-In-Time Thinking
The architecture treats privileged access as something that should be granted for a defined operational purpose and duration rather than permanently.
Auditability
Vault audit logging records activity against protected resources.
Security Monitoring
Wazuh provides endpoint and security visibility around the administrative infrastructure.
Defense in Depth
No individual component is expected to solve the entire privileged-access problem.
Identity, IAM, Vault, network controls, endpoint monitoring, and audit logging collectively create the security boundary.
________________________________________

15. The Biggest Implementation Lesson
The most important lesson from this project is that PAM is not a single product.
It is a security architecture.
A privileged-access system becomes meaningful when multiple controls work together:
![](/images/12.png)
If one layer is missing, the overall security story becomes weaker.
For example, strong identity without authorization can still produce excessive privilege.
Strong authorization without auditability makes investigations difficult.
A secure Vault without network isolation increases its attack surface.
Network isolation without monitoring makes suspicious activity harder to detect.
Monitoring without strong identity makes attribution difficult.
The strength comes from the combination.
________________________________________

16. Reproducing the Approach
An engineer looking to reproduce this architecture should think in layers rather than attempting to deploy everything at once.
Start with the identity model.
Define who the privileged operator is and how authentication reaches AWS.
Then establish the AWS authorization boundary using IAM Identity Center and temporary IAM roles.
Next, build the private PAM network.
Place Vault in a private subnet and restrict administrative connectivity.
Deploy Vault and configure TLS, storage, authentication, policies, and the KV engine.
Create a test privileged resource and verify both authorized and unauthorized behavior.
Enable Vault audit logging.
Then deploy the Wazuh monitoring layer and place the Wazuh agent on the administrative endpoint.
Finally, validate the architecture through evidence.
The sequence matters because each layer depends on the trust boundary established by the previous one.
The objective is not to create a turnkey production deployment from a blog post. The objective is to understand the security decisions well enough to reproduce the architecture safely in an appropriate lab or controlled environment.
________________________________________

17. Evidence Should Tell the Story
The final portfolio should not contain dozens of raw terminal screenshots.
A small number of carefully selected artifacts are more effective.
The architecture diagram explains the system at a glance.
The privileged-access screenshot demonstrates the protected PAM resource.
The Vault policy screenshot demonstrates authorization.
The Vault version/audit evidence demonstrates lifecycle and accountability.
The Wazuh screenshots demonstrate monitoring.
Together, these artifacts tell a coherent story:
![](/images/13.png)
That is ultimately what makes the project useful as a security-engineering portfolio.
It is not just a demonstration that several tools can be installed.
It demonstrates how those tools can be assembled into a defensible privileged-access architecture.
________________________________________

Conclusion
This project started with a simple security requirement: privileged access should be controlled, protected, observable, and auditable.
The resulting implementation connects enterprise identity through Okta and AWS IAM Identity Center, uses temporary AWS IAM roles for authorization, isolates privileged infrastructure inside private AWS networking, uses HashiCorp Vault for protected secrets and policy enforcement, routes administration through a bastion, and adds Wazuh for security monitoring and endpoint visibility.
The most important outcome is not any individual component.
It is the chain of controls:
Identity → Temporary Access → Privileged Control → Vault → Private Infrastructure → Monitoring → Audit Evidence
That chain provides a practical foundation for understanding how modern PAM architectures are built.
For engineers new to PAM, it demonstrates the fundamental concepts.
For experienced engineers, it provides a concrete implementation showing how identity, authorization, secrets management, network segmentation, endpoint monitoring, and auditability can be connected into one operational security model.
And for a portfolio, the strongest message is simple:
Privileged access should never be treated as just a login. It should be an explicitly controlled, time-bounded, monitored, and auditable security event.

---

✈️ Interested in more on this topic? Drop your thoughts or questions in the comments—let’s keep the conversation flying.
