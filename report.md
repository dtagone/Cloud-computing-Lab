# Lab 6 Report — Object Storage and Data Lifecycle

Name: [Your Name]
Subject: Cloud Computing Security Essentials
Code: IKB 42603
Lecturer: Madam Adani
Date: [Current Date]

## Purpose
This lab demonstrates the secure configuration of cloud object storage, including bucket creation, access control, versioning, lifecycle management, encryption, and data immutability.

## Requirements
- AWS CLI configured or LocalStack running locally.
- Standard shell tools.
- IAM user with appropriate S3 permissions.

## Setup
Environment setup for the object storage lab, ensuring the AWS CLI is configured to interact with the target environment.

<img width="768" alt="Environment setup evidence" src="Environment setup.png" />

## Task 1 — Bucket Creation and Object Upload
In this task, a secure S3 bucket was created and an initial object was uploaded to establish a baseline for storage management.

<img width="768" alt="Task 1.1 Evidence" src="Task 1.1 Evidence.png" />
<img width="768" alt="Task 1.2 Evidence" src="Task 1.2 Evidence.png" />
<img width="768" alt="Task 1.3 Evidence" src="Task 1.3 Evidence.png" />

## Task 2 — Access Control and Bucket Policies
This task involved testing public access restrictions and applying bucket policies. The attacker view demonstrates blocked access without the proper credentials.

<img width="768" alt="Task 2 Evidence Attacker view" src="Task 2 Evidence Attacker view.png" />
<img width="768" alt="Task 2 Evidence" src="Task 2 Evidence.png" />

## Task 3 — Object Versioning
### Verification
The evidence shows that object versioning is enabled on the bucket, and multiple versions of the same file are maintained after modifications.

<img width="768" alt="Task 3.1 Evidence" src="Task 3.1 Evidence.png" />
<img width="768" alt="Task 3.2 Evidence" src="Task 3.2 Evidence.png" />

### Explanation
Enabling versioning ensures that every modification or deletion of an object creates a new version rather than overwriting or permanently deleting the original data. This provides a critical recovery mechanism against accidental deletions, application logic failures, or malicious ransomware encryption.

## Task 4 — Data Lifecycle Management
### Verification
The provided evidence displays the configuration of a lifecycle rule, defining specific transition periods for objects moving to colder storage tiers and eventual expiration.

<img width="768" alt="Task 4.1 Evidence" src="Task 4.1 Evidence.png" />
<img width="768" alt="Task 4.2 Evidence" src="Task 4.2 Evidence.png" />
<img width="768" alt="Task 4.3 Evidence" src="Task 4.3 Evidence.png" />
<img width="768" alt="Task 4.4 Evidence" src="Task 4.4 Evidence.png" />

### Explanation
Data lifecycle policies automatically manage the storage classes of objects based on their age and access patterns. By transitioning older, less frequently accessed data to cheaper storage classes (like Glacier) and automatically deleting obsolete data, organizations can optimize storage costs while adhering to data retention policies.

## Task 5 — Pre-signed URLs for Temporary Access
Generated a pre-signed URL to grant time-limited, secure access to a private object without requiring the requester to have AWS credentials.

<img width="768" alt="Task 5.1 Evidence" src="Task 5.1 Evidence.png" />
<img width="768" alt="Task 5.2 Evidence" src="Task 5.2 Evidence.png" />

## Task 6 — Data Encryption at Rest
### Verification
The evidence confirms that default Server-Side Encryption (SSE-S3 or SSE-KMS) is enabled on the bucket, ensuring all newly uploaded objects are automatically encrypted.

<img width="768" alt="Task 6.1 Evidence" src="Task 6.1 Evidence.png" />
<img width="768" alt="Task 6.2 Evidence" src="Task 6.2 Evidence.png" />

### Explanation
Server-side encryption protects data at rest by encrypting it at the storage layer. If an unauthorized actor gains access to the underlying physical storage media, they cannot read the data without the corresponding decryption keys. It is a fundamental security control for protecting sensitive information in the cloud.

## Task 7 — Access Logging and Monitoring
Enabled server access logging or CloudTrail data events to track all requests made to the bucket, providing an audit trail for security analysis.

<img width="768" alt="Task 7.1 Evidence" src="Task 7.1 Evidence.png" />
<img width="768" alt="Task 7.2 Evidence" src="Task 7.2 Evidence.png" />
<img width="768" alt="Task 7.3 Evidence" src="Task 7.3 Evidence.png" />

## Task 8 — Data Immutability (Object Lock)
### Verification
The evidence verifies the successful configuration of Object Lock, along with a verification command testing the restriction against deleting a locked object.

<img width="768" alt="Task 8.1 Evidence" src="Task 8.1 Evidence.png" />
<img width="768" alt="Task 8.2 Evidence" src="Task 8.2 Evidence.png" />
<img width="768" alt="Task 8.2 Evidence verify" src="Task 8.2 Evidence verify.png" />

### Explanation
Object Lock enforces a Write-Once-Read-Many (WORM) model. By locking objects, it prevents them from being modified or deleted by anyone (including administrators) for a specified retention period. This is essential for regulatory compliance (e.g., SEC rule 17a-4) and provides strong protection against data destruction by malicious actors.

## Verification commands
Executed final verification checks to ensure all security controls (versioning, encryption, locking) were successfully applied.

<img width="768" alt="Verification commands evidence" src="Verification commands.png" />

## Cleanup and teardown
Removed all created objects, deleted the bucket, and cleared any temporary IAM policies to avoid ongoing charges and maintain a clean environment.

<img width="768" alt="Policy remove before session B evidence" src="Policy remove before session B.png" />
<img width="768" alt="Clean up and teardown evidence" src="Clean up and teardown.png" />
<img width="768" alt="Clean up and teardown 2 evidence" src="Clean up and teardown 2.png" />

## Security best-practices checklist

- [x] S3 buckets block all public access by default.
- [x] Bucket versioning is enabled to protect against accidental overwrites.
- [x] Lifecycle policies are configured to manage storage costs efficiently.
- [x] Server-side encryption (SSE-S3 or SSE-KMS) is enforced for data at rest.
- [x] Least privilege IAM policies are used to restrict bucket access.
- [x] Pre-signed URLs are utilized for secure, time-bound access sharing.
- [x] Object Lock is employed for critical records requiring immutability.

## END OF REPORT
