# Lab 5 Report — Monitoring, Logging & Incident Detection

Name: Muhammad A'beed bin Firdaus 52215124303

Subject: Cloud Computing Security Essentials

Code: IKB 42603

Lecturer: Madam Adani

Date: 1 September 2026

## Purpose

This lab demonstrates cloud security monitoring and incident response through centralised application logging, security queries, tamper-evident hash-chained logs, event correlation, containment, and evidence preservation. The scenario identifies a probable brute-force login followed by unauthorised data export.

## Requirements

- Windows OS with Docker Desktop running locally.
- Git Bash with AWS CLI v2, `grep`, `awk`, `sed`, `sha256sum`, and standard shell tools.
- LocalStack running locally and reachable at `http://localhost:4566`.
- AWS CLI configured to use the LocalStack CloudWatch Logs endpoint.

## Setup — Central log service

LocalStack was started and the CloudWatch log group `/ccse/app` plus the `auth` stream were created. This provides one central destination for application telemetry instead of leaving logs only on the application host.

<img width="747" height="157" alt="Setup" src="https://github.com/user-attachments/assets/db79ea3a-9aeb-4709-b3d9-3c7a992d0ed2" />


## Task 1 — Generate application logs

`auth.log` was created with seven authentication and data-access events. The log contains one normal login by `ahmad`, four failed `admin` logins from `203.0.113.9`, a later successful `admin` login from that same IP, and a `500MB` data export. Together, these entries provide the raw evidence needed to detect a suspicious sequence.

<img width="630" height="350" alt="Task 1 Evidence" src="https://github.com/user-attachments/assets/103d2276-83cd-48fe-ad33-535f1b567aa9" />


## Task 2 — Centralise logs in CloudWatch

Each line from `auth.log` was sent to the `/ccse/app` / `auth` CloudWatch Logs stream in LocalStack. The `get-log-events` read-back returns all seven messages, confirming that central collection was successful. Centralised logging improves visibility, supports investigation after a host incident, and enables organisation-wide monitoring.

<img width="938" height="305" alt="Task 2 Evidence" src="https://github.com/user-attachments/assets/bb19d31e-f7e2-4c5f-9376-7b8e098f604b" />


## Task 3 — Query security-relevant activity

The failed-login query grouped failures by the username and source IP fields. It returned `4  ip=203.0.113.9`, identifying four failed attempts from the suspicious external address. This is a security-relevant indicator because repeated failures can indicate password guessing or brute-force activity.

```bash
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

<img width="635" height="75" alt="Task 3 Evidence" src="https://github.com/user-attachments/assets/f730ff5d-cb54-447d-936e-c962ca868579" />


## Task 4 — Tamper-evident hash-chained log

Each record was chained to the SHA-256 hash of the previous record, beginning with `PREV=0`. The resulting `auth.chain` stores each original log entry together with its chained hash. The final hash for the original chain begins `ababa787b4bf...`.

The export value was then altered from `500MB` to `5MB` in `auth.tampered`. Because the altered line is part of the input to the final chained hash, recomputing the chain produces a different final hash from the original. This makes the modification detectable. In an operational design, the final hash or the chain should also be forwarded to a separate append-only store, preventing an attacker who changes the application log from rewriting the reference audit trail.

<img width="935" height="593" alt="Task 4 Evidence" src="https://github.com/user-attachments/assets/088ad689-c26a-471e-bcb0-461abbb65dc9" />

<img width="612" height="170" alt="Task 4 Evidence tampered" src="https://github.com/user-attachments/assets/2bc7ede2-29c9-458e-b3be-8d1c35981386" />

## Task 5 — Detect the incident through correlation

The monitoring logic correlated three observations for `203.0.113.9`: four failed logins, one successful login, and one data export. It produced:

```text
IP=203.0.113.9 fails=4 success=1 export=1
ALERT: probable brute-force -> compromise -> data exfiltration
```

No individual log entry is conclusive by itself. The sequence indicates that brute-force attempts were followed by a likely account compromise and a potentially unauthorised 500MB data export—an incident requiring response.

<img width="705" height="372" alt="Task 5 Evidence" src="https://github.com/user-attachments/assets/2c6dcbd8-e122-47f4-af78-3cd4957432bc" />


## Task 6 — Incident response: contain and collect evidence

The response first contained the threat by adding an `iptables` INPUT rule that drops traffic from `203.0.113.9`. The displayed rule confirms the source address is blocked.

Next, `auth.log` was copied to the timestamped file `evidence_20260901.log`, and SHA-256 was calculated into `evidence.sha256`. The evidence hash was:

```text
0adcbdd3b66099bcc0540c4c0f76946e71b52e4c99322731696a203b *evidence_20260901.log
```

This establishes a baseline for proving that the collected evidence has not changed after acquisition.

<img width="878" height="267" alt="Task 6 Evidence" src="https://github.com/user-attachments/assets/4b5327f9-ec5b-43ea-bb3c-fde65a3309b3" />


## Incident report

### Detection

The correlation rule identified `203.0.113.9` as suspicious: it recorded four failed `admin` logins, followed by a successful login and a 500MB `EXPORT_DATA` action. The generated alert classified the pattern as probable brute-force, compromise, and data exfiltration.

### Analysis

The four closely grouped failures suggest password guessing against the `admin` account. The later success from the same external IP is inconsistent with the preceding failures and, when combined with the large export, indicates that the account may have been compromised. Centralised CloudWatch read-back retained the activity for review.

### Containment

Traffic from `203.0.113.9` was blocked with an `iptables` INPUT rule using the `DROP` target. This immediately prevents further inbound activity from the suspected address while the incident is investigated.

### Evidence & integrity

The original log was copied to `evidence_20260901.log` and its SHA-256 digest was recorded in `evidence.sha256`. `sha256sum -c evidence.sha256` later returned `OK`, confirming that the collected evidence was unchanged. The original audit entries were also hash-chained, so alteration of a chained log record changes the subsequent hash value and is detectable.

### Lesson learned

The incident was visible only when multiple events were correlated. Centralised, tamper-evident logs and prompt preservation of a hashed evidence copy make it possible to detect, contain, and investigate the activity with confidence.

## Verification commands

The final validation confirmed the CloudWatch log group and the integrity of the evidence copy:

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
sha256sum -c evidence.sha256
```

The hash verification returned `evidence_20260901.log: OK`. The LocalStack response listed the created log group; its displayed local path reflects the Git Bash path-conversion behaviour in this environment.

<img width="935" height="363" alt="Verification commands" src="https://github.com/user-attachments/assets/e9da3738-6062-42f0-887a-47d2a51bfab9" />


## Cleanup and teardown

The temporary logs, chained and tampered copies, and evidence artefacts were removed with `rm -f`. The LocalStack Docker container was then stopped. This removes the temporary lab environment after evidence collection and verification.

<img width="665" height="122" alt="Cleanup and teardown" src="https://github.com/user-attachments/assets/38acc633-c9b1-4b82-acfc-cb06286f831a" />


## Short-answer questions

### Q1. What is the difference between a log and an event? Give an example of each from this lab.

A log is a durable record of something that happened; for example, `2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9` in `auth.log`. An event is an actionable signal or trigger derived from records, such as the `ALERT: probable brute-force -> compromise -> data exfiltration` generated when the correlation conditions were met.

### Q2. Why must audit logs be tamper-proof, and how does a hash chain achieve this?

Audit logs must be tamper-proof so they can be trusted for investigations, accountability, and compliance. In a hash chain, every entry’s hash incorporates the previous hash. Changing the `EXPORT_DATA` size changes that entry’s hash and all following hashes, so comparison with the stored original final hash reveals the alteration.

### Q3. How did correlation detect an incident that no single log line revealed?

The rule linked four failures, then a successful login, then an export from the same IP. A failure alone may be a user mistake, a success alone can be legitimate, and an export alone may be authorised. Their ordered combination from one address reveals the likely attack lifecycle.

### Q4. List the incident-response steps you performed and the goal of each.

1. Detected the incident by correlating failed logins, a success, and an export; the goal was to identify suspicious behaviour.
2. Contained the suspected attacker by dropping inbound traffic from `203.0.113.9`; the goal was to prevent further access.
3. Collected a timestamped copy of `auth.log`; the goal was to preserve the source evidence.
4. Hashed the evidence and verified it later; the goal was to demonstrate that the evidence remained unchanged.
5. Documented the incident report and timeline; the goal was to preserve conclusions and support follow-up action.

### Q5. How do the same logs serve both security monitoring and compliance evidence?

For monitoring, the logs expose failed authentication, successful access, exports, and patterns requiring alerts. For compliance, they provide an auditable record of actions, centralised retention, and integrity evidence through hashes and the tamper-evident chain. The same records can therefore demonstrate both operational oversight and accountability.

## Security best-practices checklist

- [x] Logs are centralised in CloudWatch Logs instead of remaining only on the application host.
- [x] Failed logins are queried and grouped by source IP.
- [x] Logs are hash-chained to make tampering evident; an independent append-only store is recommended for the chain reference.
- [x] Multiple events are correlated to detect the probable incident.
- [x] Incident response included detection, containment, evidence collection, integrity verification, and documentation.

## END OF REPORT

