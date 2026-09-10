# Lab 6 Report — Object Storage Security & the Data Security Lifecycle

Name: Muhammad A'beed bin Firdaus 52215124303  
Subject: Cloud Computing Security Essentials  
Code: IKB 42603  
Date: 10 September 2026
Lecturer: Madam Adani

## Purpose

This lab demonstrates secure object-storage operations in Amazon S3 emulated by LocalStack. It covers data classification, a deliberately public bucket, preventive access controls, IAM and bucket-policy evaluation, SSE-KMS encryption, delegated access, versioning, lifecycle retention, and cryptographic erasure. The lab bucket was `miit-patient-records-11331`.

## Environment

LocalStack Pro was started at `http://localhost:4566` with `ENFORCE_IAM=1`; the AWS CLI was configured for `us-east-1`. The `sts get-caller-identity` response confirms the LocalStack account `000000000000`.

<img width="932" height="496" alt="Environment setup" src="https://github.com/user-attachments/assets/2039f7c0-13b1-4bfd-98bf-34be93723e3d" />

## Task 1 — Classify data before storage

Three objects were created and tagged before access controls were applied. The object listing shows the three prefixes and the tag query confirms `classification=confidential` on the patient record.

<img width="545" height="197" alt="Task 1 1 Evidence" src="https://github.com/user-attachments/assets/1025ac61-1540-42e5-a342-d31da15947ef" />

<img width="755" height="702" alt="Task 1 2 Evidence" src="https://github.com/user-attachments/assets/205c11be-6bd8-4b7e-a223-e5d3f2d36d7d" />

| Classification | Who may read it | Impact if leaked | Control implemented |
|---|---|---|---|
| public | Anyone | Low; intended public information may be altered or misrepresented | `public/` prefix; no public policy remains in the final posture |
| internal | Authorised hospital staff / account | Operational information and staff privacy may be exposed | Least-privilege bucket policy scoped to `internal/*`; time-limited presigned sharing |
| confidential | Only authorised clinical personnel and system owners | Patient-data disclosure, privacy harm, and regulatory breach | `confidential/` prefix; SSE-KMS default encryption, Block Public Access, versioning, lifecycle, and cryptographic erasure capability |

<img width="742" height="392" alt="Task 1 3 Evidence" src="https://github.com/user-attachments/assets/239ba247-709e-4c85-aa5a-ae5a710652ec" />

## Task 2 — Reproduce the public-bucket breach

The bucket policy used `"Principal": "*"` with `s3:GetObject` on `arn:aws:s3:::miit-patient-records-11331/*`. An unauthenticated `curl` request returned `HTTP 200` and displayed the confidential patient record. This proves that the exposure required no exploit: the wildcard principal authorised every requester.

<img width="792" height="527" alt="Task 2 Evidence" src="https://github.com/user-attachments/assets/a2d8d03d-7590-4cbc-ba8a-48ed09ded2ec" />

<img width="547" height="162" alt="Task 2 Evidence Attacker view" src="https://github.com/user-attachments/assets/312f16c2-b311-4870-9fe6-418b6f26c213" />

## Task 3 — Remediate with Block Public Access

The public policy was removed and all four bucket-level Block Public Access flags were set to `true`: `BlockPublicAcls`, `IgnorePublicAcls`, `BlockPublicPolicy`, and `RestrictPublicBuckets`. The least-privilege replacement allows only the LocalStack account root to read `internal/*`.

**Verification and explanation.** The screenshot records all four flags as `true`, which is the required configuration evidence. It also shows that LocalStack accepted the attempted public policy and the anonymous request still returned `HTTP 200`. This is a LocalStack enforcement limitation, not evidence that the AWS control is ineffective. On AWS, `BlockPublicPolicy=true` rejects a newly attached public bucket policy such as `Principal: "*"`; `RestrictPublicBuckets=true` also restricts public bucket-policy access to AWS service principals and authorised account users. The ACL flags protect against the equivalent ACL-based route to public access.

A guardrail is preventative: it stops a risky policy from being attached in the first place. A detective control finds or reports public exposure only after a configuration has already created a disclosure window. The former is therefore stronger for a multi-engineer environment because it reduces both the chance and the blast radius of human error.

<img width="857" height="472" alt="Task 3 1 Evidence" src="https://github.com/user-attachments/assets/00c5892a-3664-4569-8174-d09482e46fc7" />

<img width="865" height="515" alt="Task 3 2 Evidence" src="https://github.com/user-attachments/assets/6eb06564-7b8e-4eb2-8cda-06eb8a4e9668" />

## Task 4 — Identity policy versus resource policy

The `DataAnalyst` IAM identity policy allows `s3:GetObject` and `s3:ListBucket` on `*`. The bucket policy separately allows that principal to read `internal/*` and explicitly denies it all `s3:*` actions on `confidential/*`.

**Verification and explanation.** The internal request succeeded and printed `internal: ALLOWED`, as expected. The confidential request also returned object metadata in this LocalStack run rather than the expected denial, even though the `DenyAnalystConfidential` statement is shown in the captured policy. The required policy evaluation on AWS is:

1. Begin with an implicit/default deny.
2. Evaluate applicable explicit `Deny` statements; any matching deny overrides every allow.
3. If no deny matches, permit the request only when an applicable allow exists.

For `internal/roster.txt`, the identity policy allows `GetObject` and the resource policy's `AllowAnalystInternal` also matches, so the request is allowed. For `confidential/record.txt`, the identity policy's broad allow matches, but `DenyAnalystConfidential` matches the same analyst, action (`s3:*` includes `GetObject`), and `confidential/*` resource. The explicit bucket-policy deny decides the request, so real AWS must deny it. The observed success should be treated as LocalStack's policy-enforcement limitation, while the two policy documents and this evaluation provide the requested evidence.

<img width="683" height="493" alt="Task 4 1 Evidence" src="https://github.com/user-attachments/assets/2c89d7ac-e2cb-4692-a8da-6c60cc7d6a2f" />

<img width="721" height="352" alt="Task 4 2 Evidence" src="https://github.com/user-attachments/assets/c6c8ceb2-0191-4f49-9d11-8d5a17c46bc2" />

<img width="846" height="385" alt="Task 4 3 Evidence" src="https://github.com/user-attachments/assets/9ddd8872-70fc-425e-a336-ba84a0746624" />

<img width="893" height="626" alt="Task 4 4 Evidence" src="https://github.com/user-attachments/assets/6bf8832b-d410-42ce-9a93-a74c8e23624b" />

## Task 5 — Default encryption at rest

A dedicated KMS key (`552871ec-b3c5-4f69-929e-68d9699884c0`) was configured as the bucket default with `SSEAlgorithm: aws:kms` and `BucketKeyEnabled: true`. Uploading `confidential/record-v2.txt` without encryption flags still produced `aws:kms`, the KMS key ARN, and `true` for Bucket Key in `head-object`. This demonstrates bucket-enforced server-side encryption without relying on each uploader to request it.

<img width="582" height="342" alt="Task 5 1 Evidence" src="https://github.com/user-attachments/assets/c26370db-b6bf-49bf-a54f-a062bd7161aa" />

<img width="891" height="770" alt="Task 5 2 Evidence" src="https://github.com/user-attachments/assets/7067d4ff-a3c8-451e-a269-3728e8363fbe" />

## Task 6 — Delegated access and the condition-key trap

A presigned URL was created for only `internal/roster.txt` with a 60-second expiry. The URL contains `X-Amz-Algorithm=AWS4-HMAC-SHA256`, credential/scope data, `X-Amz-Date`, `X-Amz-Expires=60`, signed headers, and `X-Amz-Signature`. The expiry binds the delegated capability to a short validity period; the signature binds the permitted request, object, credential scope, and expiry to the signer. Anyone who possesses the URL before expiry can exercise that authority, so it must be treated as a bearer secret and not shared in logs or public channels.

**Verification and explanation.** The first request returned `HTTP 200`; after `sleep 65`, LocalStack still returned `HTTP 200`. This is the documented LocalStack expiry-enforcement limitation, not an indication that a real S3 presigned URL remains valid after expiry. The `DenyUnencryptedTransport` policy was then applied with `aws:SecureTransport=false`. The following ordinary `http://localhost:4566` `list-objects-v2` call also succeeded. In real AWS, a request over plain HTTP makes that condition true, the explicit `Deny` matches `s3:*`, and every bucket request is refused. Here LocalStack did not enforce the condition key.

The policy itself is appropriate for AWS's HTTPS S3 endpoint but inappropriate for this plain-HTTP emulator. Condition keys must be evaluated against the actual deployment environment: blindly copying a correct TLS-only policy into LocalStack can lock out the operator in an enforcing environment, whereas assuming the emulator's successful call proves a production policy works would be unsafe.

<img width="935" height="741" alt="Task 6 1 Evidence" src="https://github.com/user-attachments/assets/f6a61a24-8906-43a1-8852-7c184406cecc" />

<img width="600" height="742" alt="Task 6 2 Evidence" src="https://github.com/user-attachments/assets/cb4099b3-1309-4d2b-b14f-cea2cd6b2f77" />

## Task 7 — Versioning, delete markers, and data remanence

Versioning was enabled. Two revisions were uploaded after the original `null` version. Deleting the key created a current delete marker, so an ordinary `get-object` failed with `NoSuchKey`; retrieving version `null` still recovered the original unredacted diagnosis. This is object-level data remanence. The original version was later explicitly deleted, but the evidence listing shows other versions still require per-version removal.

<img width="687" height="671" alt="Task 7 1 Evidence" src="https://github.com/user-attachments/assets/f66bde98-09bc-4d02-89b5-e3b0d68bc5c3" />

<img width="933" height="782" alt="Task 7 2 Evidence" src="https://github.com/user-attachments/assets/25dab1c2-298e-4972-9913-4470ac4f7996" />

<img width="801" height="315" alt="Task 7 3 Evidence" src="https://github.com/user-attachments/assets/53a016ef-190e-43f2-872f-462d11e4d8ed" />

## Task 8 — Lifecycle, retention, and cryptographic erasure

The lifecycle configuration contains two enabled rules: `RetireConfidentialRecords` expires `confidential/` current objects after 365 days and noncurrent versions after 30 days; `AbortIncompleteUploads` aborts incomplete multipart uploads after seven days. These are machine-readable, auditable retention controls rather than an ad-hoc deletion practice.

The KMS key was first enabled, then disabled and scheduled for deletion with a seven-day pending window. The final `describe-key` output reports `PendingDeletion` with a deletion date. This is the evidence that cryptographic erasure has been initiated.

**Verification and explanation.** LocalStack nevertheless returned `confidential/record-v2.txt` after the key was disabled/scheduled, so it did not reliably re-check the key state for S3 reads. The additional KMS check reports `KMSInvalidStateException` when attempting `kms encrypt` using that pending-deletion key. That is direct KMS-layer proof that the key is no longer usable. The follow-up `decrypt` failure is `InvalidCiphertextException` because no valid ciphertext was created after the preceding encryption failed; it should not be presented as an independent decrypt-after-disable test.

In AWS, destroying the customer-managed key makes ciphertext encrypted under it unrecoverable, including replicated or backed-up copies that retain only ciphertext. This provides stronger assurance than overwriting because the organisation cannot reliably locate or control every physical disk, replica, backup, or storage remnant; disabling/scheduling destruction of the key centrally removes the ability to decrypt them. Key deletion must still follow retention, legal-hold, and recovery-window requirements because it is deliberately irreversible after completion.

<img width="662" height="662" alt="Task 8 1 Evidence" src="https://github.com/user-attachments/assets/40c6ab30-6ddb-40f6-acb5-03e2d6a2d923" />

<img width="930" height="776" alt="Task 8 2 Evidence" src="https://github.com/user-attachments/assets/144200ce-2a61-4ca4-978b-bc5c0a3d91e0" />

<img width="932" height="213" alt="Task 8 2 Evidence verify" src="https://github.com/user-attachments/assets/146a48bb-3328-4cfb-a9ba-76c6683a5444" />

## Short-answer questions

### 1. What caused the Task 2 exposure, and why is it especially dangerous?

`"Principal": "*"` caused the exposure because it authorises every principal, including unauthenticated internet users, to perform `s3:GetObject` on every object under the bucket wildcard. An over-broad IAM policy is attached to one known identity and is limited by that identity's credentials and other applicable controls. A public bucket policy makes the resource reachable by unknown, unauthenticated principals, greatly expanding the attack surface.

### 2. Identity-based versus resource-based policy; which decided Task 4?

An identity-based policy is attached to an IAM principal and states what that identity may do. A resource-based policy is attached to a resource and states which principals may access that resource. The internal request was permitted because both the analyst IAM allow and `AllowAnalystInternal` applied. The confidential request must be denied on AWS because the resource policy's explicit `DenyAnalystConfidential` overrides the identity allow.

### 3. Why is Block Public Access a guardrail?

It is an account or bucket-level preventive boundary that overrides or rejects public policies and ACLs even when an engineer makes a mistake. A normal access-control policy grants or denies a particular request; a guardrail constrains which risky configurations can be created. This distinction matters at scale because it prevents accidental internet exposure before it occurs rather than requiring a later scan, alert, and remediation.

### 4. Does default SSE-KMS protect the record from the analyst?

Not by itself. SSE-KMS encrypts object data at rest and S3 decrypts it transparently for a request that S3 authorises and that has the required KMS permission. It protects stored media and reduces the impact of storage-layer compromise; it does not override an authorised `GetObject`, fix a public bucket policy, or implement least privilege. Access policy and KMS permissions remain the controls that stop an analyst from reading the data.

### 5. Why is `delete-object` alone not compliant with erasure, and what makes it provable?

With versioning, `delete-object` creates a delete marker rather than destroying historical versions. Task 7 recovered the original confidential record using version ID `null` after the normal key appeared absent. Provable deletion requires deleting every object version and delete marker by version ID, and/or cryptographically erasing all copies by destroying the KMS key. Lifecycle expiration can automate version deletion according to the documented retention schedule.

### 6. Three commands an auditor would collect

| Command | Control evidenced |
|---|---|
| `aws $EP s3api get-public-access-block --bucket $BUCKET` | All four public-access guardrail flags are enabled |
| `aws $EP s3api head-object --bucket $BUCKET --key confidential/record-v2.txt` | The object has SSE-KMS encryption and the assigned customer-managed key |
| `aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET` | Enabled lifecycle/retention rules for confidential data and incomplete uploads |

## Final verification

The supplied verification output confirms the final security posture: all four Block Public Access flags are `True`, versioning is `Enabled`, default encryption is `aws:kms` using key `552871ec-b3c5-4f69-929e-68d9699884c0`, both lifecycle rules are `Enabled`, and the key state is `PendingDeletion`.

<img width="930" height="577" alt="Verification commands" src="https://github.com/user-attachments/assets/2c4b876d-553e-424b-91c1-42b5e3416f6c" />

## Cleanup and teardown

The bucket policy was removed, all object versions and delete markers were deleted explicitly, and the LocalStack container was removed. Explicit deletion of both versions and delete markers is necessary because a versioned bucket is not empty after an ordinary delete.

<img width="761" height="827" alt="Clean up and teardown" src="https://github.com/user-attachments/assets/3151ad6a-7966-4d42-a4f9-485c2653c38f" />

<img width="340" height="95" alt="Clean up and teardown 2" src="https://github.com/user-attachments/assets/20d19dda-2399-4d2d-9078-030ac62612e4" />

## Security best-practices checklist

- [x] Every object was classified and tagged before access decisions.
- [x] The public-bucket risk was demonstrated, removed, and replaced with a scoped least-privilege policy.
- [x] All four Block Public Access flags are enabled.
- [x] Access policy is scoped by identity and object prefix; explicit deny precedence is documented.
- [x] Default bucket encryption is SSE-KMS with a customer-managed key.
- [x] File sharing uses a time-bounded presigned URL rather than permanent public access.
- [x] Versioning and delete-marker remanence were demonstrated.
- [x] Lifecycle rules and KMS cryptographic erasure provide auditable retention and deletion controls.

## END OF REPORT
