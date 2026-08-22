# Lab 4 Report — Access Control & Network Security

Name: Muhammad A'beed bin Firdaus 52215124303

Subject: Cloud Computing Security Essentials

Code: IKB 42603

Lecturer: Prof. Dr. Shahrulniza Musa

Date: 22 Aug 2026

## Purpose

This lab implements layered cloud security controls: authentication, multi-factor authentication (MFA), role-based authorization, network segmentation, default-deny firewalling, and container hardening. The work demonstrates the difference between proving an identity and limiting what that identity or workload may do.

## Requirements

- Docker Desktop and Git Bash for Windows.
- `kind` and `kubectl` for the local Kubernetes cluster.
- An authenticator application for Task 2.
- Trivy (or an equivalent scanner) for image scanning.
- Internet access for the initial container-image downloads.

## Task 1 — Authentication: Password-Protected Service

An NGINX service (`authsvc`) was configured with HTTP Basic authentication for the `student` account. The unauthenticated request returned **401**, which proves that the service rejects a request with no credentials. Supplying the correct username and password returned **Authenticated OK** (HTTP 200), proving that authentication succeeds only with valid credentials.

![Task 1 evidence](Task%201%20evidence.png)

## Task 2 — Add a Second Factor (MFA / TOTP)

A base32 shared secret was generated and enrolled in an authenticator app. A current six-digit TOTP code was entered and compared with the expected code, resulting in **MFA OK**.

`oathtool` could not be used in the Windows Git Bash environment. As a compatible workaround, the expected TOTP value was generated locally with Python using Base32 decoding, HMAC-SHA1, and the standard 30-second TOTP time step. This produces the same type of one-time code as `oathtool --totp -b`, without exposing the shared secret in this report.

![Task 2 enrolment evidence](Task%202%20Code%20evidence.jpeg)

![Task 2 MFA validation evidence](Task%202%20evidence.png)

## Task 3 — Authorization: Kubernetes RBAC

A local kind cluster named `ccse-lab4` was created. In the `app` namespace, the `dev` service account received the `dev-role` Role through the `dev-rb` RoleBinding. The Role permits only `get` and `list` operations on pods.

The authorization tests confirm least privilege:

| Request as `system:serviceaccount:app:dev` | Result | Reason |
|---|---:|---|
| List pods in `app` | YES | `list` on pods is explicitly allowed. |
| Create a deployment in `app` | NO | Deployment creation was not granted. |
| Delete pods in `app` | NO | `delete` was not granted. |

![Task 3.1 role and binding evidence](Task%203.1.png)

![Task 3.2 authorization-test evidence](Task%203.2.png)

## Task 4 — Network Segmentation (Three-Tier)

Two Docker networks establish a three-tier layout: `web` is attached only to `frontend-net`, `db` only to `backend-net`, and `app` is attached to both. This makes `app` the controlled bridge between front-end and database tiers.

The test shows that `web` cannot resolve or connect to `db` (**BLOCKED**), while `app` shares `backend-net` with `db` and can reach Redis on port 6379 (**REACHABLE**). The initial Alpine `apk` command was unavailable inside the Debian-based NGINX container; installing `netcat-openbsd` using `apt-get` provided the valid connectivity test.

![Task 4.1 network setup evidence](Task%204.1.png)

![Task 4.2 segmentation-test evidence](Task%204.2.png)

## Task 5 — Firewall Rules (Default-Deny)

Inside a temporary privileged test container, the firewall INPUT policy was set to `DROP`. Only inbound TCP port 443 and loopback traffic were explicitly accepted. The displayed ruleset confirms the default-deny policy and its two required exceptions.

![Task 5 firewall evidence](Task%205%20evidence.png)

## Task 6 — Container / Host Hardening

The `hardened` NGINX container was run as non-root user `1000:1000`, with a read-only root filesystem, all Linux capabilities dropped, `no-new-privileges`, and a temporary writable `/tmp` filesystem. The inspect output confirms the non-root user and `ReadOnly=true`.

The Trivy summary for `nginx:alpine (alpine 3.24.1)` reported **0 vulnerabilities**. Its Secrets field is shown as `-` (not scanned), so this report does not claim a secrets-scan result.

![Task 6.1 hardened-container evidence](Task%206.1.png)

![Task 6.2 image-scan evidence](Task%206.2.png)

### Hardening measures and the attacks they blunt

| Measure | Attack surface reduced |
|---|---|
| Run as `1000:1000` rather than root | Limits the impact of a process compromise; an attacker does not automatically obtain root privileges inside the container. |
| `--read-only` root filesystem | Prevents a compromised process from modifying binaries, configuration, or writing persistent malware to the container filesystem. |
| `--cap-drop=ALL` | Removes powerful Linux kernel privileges that could otherwise support network changes, mount operations, or container escape attempts. |
| `--security-opt no-new-privileges` | Prevents processes from gaining additional privileges through setuid/setgid binaries or similar escalation paths. |
| `--tmpfs /tmp` | Provides only temporary writable space and avoids writes to the container’s root filesystem. |
| Trivy image scan | Identifies known vulnerabilities before the image is deployed. |

## Short-Answer Questions

### Q1. Explain the difference between authentication and authorization using Tasks 1 and 3.

Authentication answers **who are you?** In Task 1, HTTP Basic authentication verifies the supplied `student` credentials: no credentials are rejected with 401, while correct credentials receive a 200 response. Authorization answers **what are you allowed to do?** In Task 3, Kubernetes recognises the `dev` service-account identity, then RBAC allows it to list pods but denies deployment creation and pod deletion.

### Q2. Why is MFA so effective, and which attacks does it defeat?

MFA requires an additional factor beyond a password: here, a current TOTP code from an authenticator app. A stolen, reused, guessed, or phished password alone is therefore insufficient for access. It strongly reduces the success of credential stuffing, password spraying, password reuse, and many phishing attacks, although sophisticated real-time phishing can still steal both factors.

### Q3. How does network segmentation limit the damage of a compromised web server?

The internet-facing `web` container is connected only to `frontend-net`, whereas the database exists only on `backend-net`. A compromise of `web` therefore does not give the attacker a direct network path to `db`; the evidence shows `web → db` is blocked. Access to the database must go through the app tier, which limits lateral movement and contains the breach.

### Q4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?

A default-deny policy drops all inbound traffic unless an explicit rule permits it. The lab allowed only TCP 443 and loopback traffic. This is the same least-privilege model used by cloud security groups: inbound and outbound connectivity should be denied by default, then narrowly allowed by port, protocol, source, and destination only where required.

### Q5. List the hardening measures you applied and the attack surface each one removes.

The container runs as a non-root user, reducing privilege-escalation impact; has a read-only root filesystem, preventing persistent modification; drops all capabilities, removing unnecessary kernel-level powers; sets `no-new-privileges`, blocking privilege gain; uses a temporary `/tmp` filesystem; and is scanned for known image vulnerabilities. Together these controls reduce the opportunities available after a container compromise.

## Required Verification Commands

The following output verifies that the `dev-rb` RoleBinding exists in `app`, refers to `dev-role`, and binds the `dev` service account. It also confirms that all container capabilities were dropped.

```bash
kubectl get rolebinding dev-rb -n app -o yaml
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```

![Verification-command evidence](Verification%20command.png)

## Security Best-Practices Checklist

- [x] Service requires authentication; unauthenticated requests were rejected.
- [x] MFA / a second factor was implemented and validated.
- [x] RBAC enforced least privilege; unauthorised actions were denied.
- [x] The data tier was segmented from the front-end tier.
- [x] A default-deny firewall with explicit allow rules was applied.
- [x] The container ran non-root, read-only, with capabilities dropped; the image was scanned.

## Cleanup and Teardown

The authentication service, three-tier containers, hardened container, Docker networks, and kind cluster were removed after verification.

```bash
docker rm -f authsvc db app web hardened 2>/dev/null
docker network rm frontend-net backend-net 2>/dev/null
kind delete cluster --name ccse-lab4
```

![Cleanup and teardown evidence](Cleanup%20and%20teardown.png)

## END OF REPORT
