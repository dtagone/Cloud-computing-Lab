# Lab 3 Report — Encryption and Key Management

Name: Muhammad A'beed bin Firdaus 52215124303

Subject: Cloud Computing Security Essentials

Code: IKB 42603

Lecturer: Madam Adani

Date: 20 August 2026

## Purpose

This lab applies encryption and key-management controls to protect a confidential patient record. The activities demonstrate symmetric encryption, public-key cryptography and digital signatures, TLS in transit, KMS-managed keys, envelope encryption, tenant key separation and secure key lifecycle, and integrity verification.

## Requirements

- Docker Desktop running locally.
- Git Bash for Windows with OpenSSL, Docker, AWS CLI v2, `awk`, `base64`, `sha256sum`, and `curl`.
- LocalStack running and reachable through the AWS CLI KMS endpoint.
- Run the commands in Git Bash.

## Task 1 — Symmetric encryption at rest

The confidential record was encrypted with AES-256-CBC using PBKDF2-derived key material:

```bash
openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc
```

Viewing `record.enc` shows ciphertext rather than the patient data. Decrypting it with the correct password created `record.dec.txt`; `diff record.txt record.dec.txt` produced `MATCH: decryption successful`. This demonstrates confidentiality at rest: the plaintext cannot be recovered without the password/key.

![Task 1 evidence](Task%201%20evidence.png)

## Task 2 — Asymmetric encryption and digital signature

An RSA 2048-bit private/public key pair was generated. The public key encrypted `record.txt`, while the private key decrypted the ciphertext. The private key also signed the record with SHA-256, and verification with the public key returned `Verified OK`.

| Operation | Key used | Security property |
|---|---|---|
| Encrypt `record.txt` | Public key | Only the private-key holder can decrypt it (confidentiality). |
| Decrypt `record.rsa` | Private key | Restores the original plaintext. |
| Sign `record.txt` | Private key | Creates evidence from the key owner. |
| Verify signature | Public key | Detects changes and validates the signer. |

![Task 2 evidence](Task%202%20evidence.png)

## Task 3 — TLS encryption in transit

A temporary self-signed certificate for `localhost` was generated and mounted read-only into an NGINX Docker container. NGINX listens on port 443 and uses the supplied certificate and private key. `curl -k https://localhost/.../record.txt` successfully retrieves the record over HTTPS.

The command below includes the required Git Bash workaround. `MSYS_NO_PATHCONV=1` prevents Git Bash from rewriting the Linux container paths in Docker volume mounts, such as `/etc/nginx/cert.pem`.

```bash
MSYS_NO_PATHCONV=1 docker run --rm -d --name tls -p ...:443 \
  -v "$PWD/cert.pem:/etc/nginx/cert.pem:ro" \
  -v "$PWD/key.pem:/etc/nginx/key.pem:ro" \
  -v "$PWD/record.txt:/usr/share/nginx/html/record.txt:ro" \
  -v "$PWD/default.conf:/etc/nginx/conf.d/default.conf:ro" nginx
```

The use of `-k` is acceptable for this local lab because the certificate is self-signed. In production, clients must validate a certificate issued by a trusted CA, and private keys must be protected and rotated.

![Task 3 evidence](Task%203%20evidence.png)

## Task 4 — Create and use a KMS customer managed key

A symmetric customer managed KMS key for `CCSE tenant-A` was created in LocalStack. The returned metadata shows `KeyUsage: ENCRYPT_DECRYPT`, `KeyManager: CUSTOMER`, `KeySpec: SYMMETRIC_DEFAULT`, and `KeyState: Enabled`. The key then encrypted the base64-encoded value `hello` using `aws ... kms encrypt`.

This centralises key administration and permits the encryption key to be controlled separately from application data.

![Task 4 evidence](Task%204%20evidence.png)

## Task 5 — Envelope encryption

Envelope encryption was implemented by asking KMS to generate an AES-256 data key. The command output contains two values: a plaintext data key and a KMS-encrypted copy of that same data key. The plaintext data key encrypted `record.txt` locally with OpenSSL; it was then deleted. Only `datakey.enc`, the KMS-wrapped data key, remains with the encrypted record.

The command uses `awk` as a Git Bash-compatible workaround to split the two fields returned by LocalStack into separate files before decoding the plaintext key:

```bash
aws $EP kms generate-data-key --key-id $KEY_A --key-spec AES_256 \
  --query '[Plaintext,CiphertextBlob]' --output text |
  awk '{print $1 > "datakey.b64"; print $2 > "datakey.enc"}'
base64 -d datakey.b64 > datakey.bin
openssl enc -aes-256-cbc -pbkdf2 -in record.txt -out record.env.enc -pass file:./datakey.bin
rm datakey.bin datakey.b64
```

This design reduces KMS workload because KMS protects a small data key rather than encrypting the full data object. It also allows many data objects to be encrypted efficiently while retaining central key control.

![Task 5 evidence](Task%205%20evidence.png)

## Task 6 — Tenant key separation and secure key lifecycle

A separate symmetric KMS key was created for `CCSE tenant-B`, establishing distinct tenant master keys. The Tenant A key was scheduled for deletion with a seven-day pending window. KMS then rejected a request to disable that key because a key pending deletion cannot be modified. An attempted data-key decryption also failed, demonstrating that key lifecycle state can prevent normal cryptographic operations.

Tenant-specific keys reduce the impact of a key compromise and make access, rotation, and deletion independently manageable for each tenant. The deletion window provides a recovery period before irreversible key destruction.

![Task 6 evidence](Task%206%20evidence.png)

## Task 7 — Integrity verification and tamper-evident audit log

`sha256sum` was calculated for the original record. After a copy was altered by appending `x`, its SHA-256 value differed from the original. This proves that even a very small modification is detectable.

An append-only audit log was then built where each new entry includes the SHA-256 hash of the preceding entry (`PREV`). Linking each record to the previous hash makes later alteration evident because it would invalidate the chain from the changed entry onward.

![Task 7 evidence](Task%207%20evidence.png)

## Required verification commands

The final checks list the remaining KMS keys and verify the original RSA signature. The output includes the Tenant A and Tenant B key ARNs, and signature verification returned `Verified OK`.

```bash
aws --endpoint-url=http://localhost:4566 kms list-keys
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

![Verification commands](Verification%20commands.png)

## Cleanup and teardown

The TLS container was stopped. Temporary encrypted/decrypted data, keys, certificates, wrapped-data-key files, and tampered test data were removed. The LocalStack container was then stopped and removed. The attempted removal of `record.txt` reported that it is a directory in the Git Bash environment; this did not affect the local cleanup of the generated cryptographic artefacts.

![Cleanup and teardown](cleanup%20and%20teardown.png)

## Short-answer questions

### Q1. What security property does encryption provide, and what does it not provide by itself?

Encryption provides confidentiality by making data unreadable to parties without the required decryption key. Encryption alone does not prove who created the data, prove it was not changed, or guarantee availability. Digital signatures and hashes address authenticity and integrity, while backups and resilient design support availability.

### Q2. Why use both symmetric and asymmetric cryptography?

Symmetric encryption is fast and appropriate for large files, but both parties need access to the same secret key. Asymmetric cryptography solves key-distribution and signing problems using a public/private key pair, but it is slower. A practical system commonly encrypts data with a fast symmetric data key and protects that data key with an asymmetric or KMS-managed key.

### Q3. Why is `curl -k` not appropriate for production?

`-k` disables certificate validation. It therefore accepts an untrusted or forged certificate and can enable a man-in-the-middle attack. It was used only because the lab uses a self-signed `localhost` certificate; production clients must validate the certificate chain and hostname.

### Q4. What is envelope encryption and why is it used with KMS?

Envelope encryption encrypts the data locally with a short-lived data encryption key and encrypts that data key with a KMS master key. KMS manages and protects the master key, while local encryption efficiently handles the larger data. The plaintext data key is discarded after use and recovered only when KMS decrypts its wrapped copy for an authorised request.

### Q5. Why should Tenant A and Tenant B have different KMS keys?

Separate keys provide cryptographic isolation. A policy mistake, compromised key, rotation event, or deletion for one tenant does not automatically affect the other tenant. It also creates clearer audit records and supports tenant-specific access controls and lifecycle decisions.

### Q6. How do hashes and chained audit records reveal tampering?

A cryptographic hash changes when the protected content changes. In a hash chain, each record stores the previous record's hash; altering an earlier record breaks the link for that record and every following record. Verification can therefore identify unauthorised modification.

## Security best-practices checklist

- [x] Encrypt confidential data at rest with AES-256 and a strong password-derived key.
- [x] Use RSA keys for public-key encryption and digital signatures.
- [x] Protect data in transit with TLS; use trusted CA certificates in production.
- [x] Use KMS customer-managed keys for centrally controlled cryptographic operations.
- [x] Use envelope encryption and delete plaintext data keys after use.
- [x] Separate tenant keys and use a controlled deletion window.
- [x] Verify integrity with SHA-256 and tamper-evident chained audit records.

## END OF REPORT
