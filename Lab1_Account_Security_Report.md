# Lab 1 Report — Cloud Account Security, Identity & Access Management

Name: Muhammad A'beed bin Firdaus 52215124303

Subject: Cloud computing security essentials

Code: IKB 42603

Lecturer: Madam Adani

Date: 2 Aug 2026

## Purpose
Stand up a local cloud lab using Docker and Localstack and applying principles of least privileges. Creating and testing fine grained permissions which allows or denies an identity what to do. Implementing and verifying RBAC (Role based access control) in Kubernetes and identities and reason about MFA, access keys and credential hygiene.

## Requirements
- At least 8 GB RAM and administrator rights to install software.
- Docker Desktop and GitBash for Windows.
- AWS CLI v2, kind and kubectl.
- Run all commands in GitBash.

## Session A Environment Setup (Task 1-4)
Verify Docker is running and then start Localstack, run each command in GitBash.

<img width="1463" height="356" alt="One time environment setup png" src="https://github.com/user-attachments/assets/53195b22-8258-4d0a-be61-f17b02d689ee" />

Point the AWS CLI at Localstack and verify. Run the commands in GitBash.

<img width="608" height="282" alt="One time environment setup aws png" src="https://github.com/user-attachments/assets/b5d73c19-5da9-4b8f-bc8e-62c9160b61ce" />

## Task 1 — Cloud identity landscape

| Concept | AWS term | Purpose |
|---|---|---|
| All-powerful owner | Root user | The original account owner with complete access to all AWS services, resources, and account settings. |
| Human/app identity | IAM User | Represents a person or application and provides credentials for accessing AWS resources. |
| Permission bundle | IAM Policy | Defines what actions are allowed or denied and which AWS resources can be accessed. |
| Collection of users | IAM Group | Organises multiple IAM users so that the same permissions can be assigned and managed together. |
| Temporary identity | IAM Role | Provides temporary permissions that can be assumed by users, applications, or AWS services without using permanent credentials. |

## Task 2 — Creating Least privilege Admin

An `Admins` IAM group was created, granted the `AdministratorAccess` managed policy, and the user `CloudAdmin_abeed` was added to it. The `get-group` output confirms that membership. This moves routine administration away from the root identity. 

<img width="635" height="976" alt="Task 2 Evidence png" src="https://github.com/user-attachments/assets/4222e780-298c-4699-b453-87ff180d8710" />


## Task 3 — Enforcing Least privilege with scoped policy

The user `Analyst_abeed` was created with only `AmazonS3ReadOnlyAccess`. The attached-policy listing shows this single read-only policy. Therefore, if this account is compromised, an attacker can at most read the S3 resources permitted by that policy; they cannot use administrator capabilities to alter IAM, delete workloads, change policies, or administer the account. Limiting what one stolen identity can do reduces the incident's **blast radius**.

<img width="750" height="362" alt="Task 3 Evidence png" src="https://github.com/user-attachments/assets/6091864f-686e-457f-a9f5-fe240c609574" />


## Task 4 — Credential hygiene and access keys

The access-key evidence shows the Analyst key is created then set to `Inactive`, the access-key status is then checked for confirmation. This demonstrates key rotation. Long-lived access keys are risky because anyone who obtains the key can make API requests as that identity until the key is disabled or deleted. In a real AWS account, keys must not be created for the root user or committed to source control; short-lived role credentials are preferred.

<img width="752" height="576" alt="Task 4 Evidence png" src="https://github.com/user-attachments/assets/39063270-b49a-4453-b6ca-10df2e3ad581" />

## Session B Environment setup (Task 5-7)
Create a Local Kubernetes Cluster and verify its creation.

<img width="755" height="616" alt="Task B setup png" src="https://github.com/user-attachments/assets/9de547c7-b873-4525-9ac9-0d2d6b8443c2" />

## Task 5 — Environment separation with Namespaces

`dev` and `prod` namespaces were created and are listed as `Active`. These namespaces create logical boundaries within the Kubernetes cluster, allowing permissions to be scoped to an environment.

<img width="338" height="322" alt="Task 5 Evidence png" src="https://github.com/user-attachments/assets/1ff7179c-3688-4a4d-ac82-c8f5c299a2ef" />


## Task 6 — Define a Role and Bind It (Least Privilege)

The `dev-user` service account was created in `dev`. The `pod-reader` Role grants only `get`, `list`, and `watch` on pods in that namespace. The `dev-user-binding` RoleBinding assigns that Role to `dev-user`; it does not grant permissions by itself.

<img width="578" height="237" alt="Task 6 Evidence png" src="https://github.com/user-attachments/assets/78ea12d2-ef9d-42b0-9b10-448ca9d08ae8" />


## Task 7 — Access-control test

For `system:serviceaccount:dev:dev-user`, the authorisation results are:

| Request | Result | Meaning |
|---|---:|---|
| List pods in `dev` | YES | The namespaced Role allows `list` on pods in `dev`. |
| Delete pods in `dev` | NO | `delete` was never granted by the Role. |
| List pods in `prod` | NO | The `dev` Role and RoleBinding have no authority in `prod`. |

The service account passes **authentication**: Kubernetes recognises the identity named by `--as`. It then performs **authorisation** against RBAC rules. Authorisation allows the first request and blocks the delete and cross-namespace requests because no matching permission exists.

<img width="465" height="252" alt="Task 7 Evidence png" src="https://github.com/user-attachments/assets/27c4dd9a-43ac-4b2e-a5da-32084b0e6646" />


## Short-answer questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?

Administrators change a group policy once and the update applies consistently to each member, making access easier to audit, reducing configuration drift, and avoiding repeated per-user policy attachments. It also supports least privilege by grouping users with the same job responsibilities.

### Q2. What is the difference between an IAM User and an IAM Role?

An IAM User is a persistent identity for a person or application and can have long-term credentials, such as a password or access keys. An IAM Role is an assumable identity that supplies temporary credentials and permissions to a trusted user, application, AWS service, or external identity. A role is preferable when temporary access can replace permanent credentials.

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.

The Analyst account has only `AmazonS3ReadOnlyAccess`, rather than administrative access. It can perform the limited S3 read actions necessary for analysis, but cannot modify data or manage the account. A compromised Analyst credential therefore exposes only the resources/actions within that narrow permission set, instead of allowing an attacker to control the whole environment. This containment is blast-radius reduction.

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?

A Role defines a set of allowed verbs on resources within one namespace—for example, reading pods in `dev`. A RoleBinding connects that Role to subjects such as users, groups, or service accounts. The Role says *what is allowed*; the RoleBinding says *who receives it*.

### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?

`dev-user` is bound to `pod-reader` only in the `dev` namespace. Kubernetes Roles and RoleBindings are namespaced, so that binding cannot grant `prod` access. The denial demonstrates least privilege and environment/tenant isolation: an identity receives only the permissions and scope explicitly required.

## Required verification command

The following output verifies that `dev-user-binding` exists in `dev`, refers to the `pod-reader` Role, and binds the `dev-user` service account in the `dev` namespace.

<img width="532" height="317" alt="Verification command" src="https://github.com/user-attachments/assets/e7e4ad0a-a41c-4c68-a243-1afdb05b11bc" />


## Security best-practices checklist

- [v] Root user is not used for daily tasks; a dedicated CloudAdmin identity exists.
- [v] Administrator permissions are granted through the `Admins` group.
- [v] A least-privilege, read-only Analyst identity was created.
- [v] Access keys were listed and rotation/deactivation was demonstrated.
- [v] Kubernetes RBAC blocked unauthorised deletion and cross-namespace access.

## END OF REPORT
