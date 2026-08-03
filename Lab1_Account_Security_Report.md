# Lab 1 Report — Cloud Account Security, Identity & Access Management

## Evidence index

| Item | Evidence |
|---|---|
| LocalStack environment and operating identity | [Docker and LocalStack setup](<One time environment setup.png.png>) and [AWS CLI caller identity](<One time environment setup aws.png.png>) |
| Task 2 — Admin group and member | [Task 2 evidence](<Task 2 Evidence.png.png>) |
| Task 3 — Analyst read-only policy | [Task 3 evidence](<Task 3 Evidence.png.png>) |
| Task 4 — Access-key rotation | [Task 4 evidence](<Task 4 Evidence.png.png>) |
| Kubernetes cluster | [Kubernetes setup](<Task B setup.png.png>) |
| Task 5 — Namespaces | [Task 5 evidence](<Task 5 Evidence.png.png>) |
| Task 6 — Role and binding | [Task 6 evidence](<Task 6 Evidence.png.png>) |
| Task 7 — Authorisation tests | [Task 7 evidence](<Task 7 Evidence.png.png>) |
| Required RBAC verification | [RoleBinding YAML](<Verification command.png>) |

## Task 1 — Cloud identity landscape

| Concept | AWS term | Purpose |
|---|---|---|
| All-powerful owner | Root user | The original account owner with complete access to all AWS services, resources, and account settings. |
| Human/app identity | IAM User | Represents a person or application and provides credentials for accessing AWS resources. |
| Permission bundle | IAM Policy | Defines what actions are allowed or denied and which AWS resources can be accessed. |
| Collection of users | IAM Group | Organises multiple IAM users so the same permissions can be assigned and managed together. |
| Temporary identity | IAM Role | Provides temporary permissions that can be assumed by users, applications, or AWS services without using permanent credentials. |

## Task 2 — Dedicated administrator

An `Admins` IAM group was created, granted the `AdministratorAccess` managed policy, and the user `CloudAdmin_abeed` was added to it. The `get-group` output confirms that membership. This moves routine administration away from the root identity.

![Task 2: Admins group, policy attachment, CloudAdmin user, and verified membership](<Task 2 Evidence.png.png>)

## Task 3 — Least-privilege analyst

The user `Analyst_abeed` was created with only `AmazonS3ReadOnlyAccess`. The attached-policy listing shows this single read-only policy. Therefore, if this account is compromised, an attacker can at most read the S3 resources permitted by that policy; they cannot use administrator capabilities to alter IAM, delete workloads, change policies, or administer the account. Limiting what one stolen identity can do reduces the incident's **blast radius**.

![Task 3: Analyst account with AmazonS3ReadOnlyAccess](<Task 3 Evidence.png.png>)

## Task 4 — Credential hygiene

The access-key evidence shows the Analyst key being set to `Inactive`, then access-key status being checked, followed by creation of an active replacement key. This demonstrates key rotation. Long-lived access keys are risky because anyone who obtains the key can make API requests as that identity until the key is disabled or deleted. In a real AWS account, keys must not be created for the root user or committed to source control; short-lived role credentials are preferred.

![Task 4: access-key deactivation, listing, and replacement](<Task 4 Evidence.png.png>)

## Task 5 — Environment separation

Separate `dev` and `prod` namespaces were created and are listed as `Active`. These namespaces create logical boundaries within the Kubernetes cluster, allowing permissions to be scoped to an environment.

![Task 5: dev and prod namespaces](<Task 5 Evidence.png.png>)

## Task 6 — Namespaced RBAC

The `dev-user` service account was created in `dev`. The `pod-reader` Role grants only `get`, `list`, and `watch` on pods in that namespace. The `dev-user-binding` RoleBinding assigns that Role to `dev-user`; it does not grant permissions by itself.

![Task 6: service account, pod-reader Role, and RoleBinding](<Task 6 Evidence.png.png>)

## Task 7 — Access-control test

For `system:serviceaccount:dev:dev-user`, the authorisation results are:

| Request | Result | Meaning |
|---|---:|---|
| List pods in `dev` | YES | The namespaced Role allows `list` on pods in `dev`. |
| Delete pods in `dev` | NO | `delete` was never granted by the Role. |
| List pods in `prod` | NO | The `dev` Role and RoleBinding have no authority in `prod`. |

The service account passes **authentication**: Kubernetes recognises the identity named by `--as`. It then performs **authorisation** against RBAC rules. Authorisation allows the first request and blocks the delete and cross-namespace requests because no matching permission exists.

![Task 7: YES / NO / NO authorisation results](<Task 7 Evidence.png.png>)

## Short-answer questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?

Groups centralise permission management. Administrators change a group policy once and the update applies consistently to every member, making access easier to audit, reducing configuration drift, and avoiding repeated per-user policy attachments. It also supports least privilege by grouping users with the same job responsibilities.

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

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
```

![Verification: dev-user-binding YAML](<Verification command.png>)

## Security best-practices checklist

- [x] Root user is not used for daily tasks; a dedicated CloudAdmin identity exists.
- [x] Administrator permissions are granted through the `Admins` group.
- [x] A least-privilege, read-only Analyst identity was created.
- [x] Access keys were listed and rotation/deactivation was demonstrated.
- [x] Kubernetes RBAC blocked unauthorised deletion and cross-namespace access.
