# Lab 4 Report — Access Control & Network Security

Name: Muhammad A'beed bin Firdaus 52215124303

Subject: Cloud Computing Security Essentials

Code: IKB 42603

Lecturer: Madam Adani

Date: 22 Aug 2026

## Purpose

This lab implements layered cloud security controls: authentication, multi-factor authentication (MFA), role-based authorization, network segmentation, default-deny firewalling, and container hardening. The work demonstrates the difference between proving an identity and limiting what that identity or workload may do.

## Requirements

- Docker Desktop and Git Bash for Windows.
- `kind` and `kubectl` for the local Kubernetes cluster.
- An authenticator application for Task 2. (This lab uses Google authenticator)
- A workaround code will be provided due to oathtool not being able in Windows.
- Trivy (or an equivalent scanner) for image scanning.
- Internet access for the initial container-image downloads.

## Task 1 — Authentication: Password-Protected Service

A `student` account was created with:
``docker run --rm httpd:alpine htpasswd -nbB student 'P@ssw0rd!' > htpasswd.txt``

Then an NGINX service (`authsvc`) was configured with HTTP Basic authentication for the `student` account. 

The unauthenticated request returned **401**, which proves that the service rejects a request with no credentials. 

``curl -s -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080``

Supplying the correct username and password returned **Authenticated OK** (HTTP 200), proving that authentication succeeds only with valid credentials.

``curl -s -u student:'P@ssw0rd!' http://localhost:8080``

Evidence:

<img width="687" height="542" alt="Task 1 evidence" src="https://github.com/user-attachments/assets/48b61736-5afe-4ffc-b98f-7e394f812dab" />


## Task 2 — Add a Second Factor (MFA / TOTP)

A base32 shared secret was generated and enrolled in an authenticator app. A current six-digit TOTP code was entered and compared with the expected code, resulting in **MFA OK**.

`oathtool` could not be used in the Windows Git Bash environment. As a compatible workaround, the expected TOTP value was generated locally with Python using Base32 decoding, HMAC-SHA1, and the standard 30-second TOTP time step. This produces the same type of one-time code as `oathtool --totp -b`, without exposing the shared secret in this report.

<img width="628" height="350" alt="Oathtool workaround" src="https://github.com/user-attachments/assets/06491bfb-ab65-4b9e-adb9-181a73851bea" />

Google authenticator is used for the 6 digit code.

<img width="520" height="396" alt="Task 2 Code evidence" src="https://github.com/user-attachments/assets/be593c42-0c41-4b65-ad3b-dfc3773f4502" />

The commands comes as followed:

<img width="578" height="204" alt="Task 2 evidence" src="https://github.com/user-attachments/assets/5d0563e3-75e5-4d38-91e5-07746522331e" />


## Task 3 — Authorization: Kubernetes RBAC

A local kind cluster named `ccse-lab4` was created. In the `app` namespace, the `dev` service account received the `dev-role` Role through the `dev-rb` RoleBinding. The Role permits only `get` and `list` operations on pods.

The authorization tests confirm least privilege:

| Request as `system:serviceaccount:app:dev` | Result | Reason |
|---|---:|---|
| List pods in `app` | YES | `list` on pods is explicitly allowed. |
| Create a deployment in `app` | NO | Deployment creation was not granted. |
| Delete pods in `app` | NO | `delete` was not granted. |

<img width="768" height="645" alt="Task 3 1" src="https://github.com/user-attachments/assets/361c256f-fc5d-45f7-8600-2281324a1a90" />

<img width="477" height="251" alt="Task 3 2" src="https://github.com/user-attachments/assets/c7468884-f5f5-4e44-ad92-80b96b290faa" />


## Task 4 — Network Segmentation (Three-Tier)

Two Docker networks establish a three-tier layout: `web` is attached only to `frontend-net`, `db` only to `backend-net`, and `app` is attached to both. This makes `app` the controlled bridge between front-end and database tiers.

The test shows that `web` cannot resolve or connect to `db` (**BLOCKED**), while `app` shares `backend-net` with `db` and can reach Redis on port 6379 (**REACHABLE**). The initial Alpine `apk` command was unavailable inside the Debian-based NGINX container; installing `netcat-openbsd` using `apt-get` provided the valid connectivity test.

<img width="720" height="610" alt="Task 4 1" src="https://github.com/user-attachments/assets/a50fcf4a-3295-437d-8991-131aa914809a" />

<img width="749" height="448" alt="Task 4 2" src="https://github.com/user-attachments/assets/54271080-37fa-479e-a9f6-209067eb9046" />


## Task 5 — Firewall Rules (Default-Deny)

Inside a temporary privileged test container, the firewall INPUT policy was set to `DROP`. Only inbound TCP port 443 and loopback traffic were explicitly accepted. The displayed ruleset confirms the default-deny policy and its two required exceptions.

<img width="675" height="209" alt="Task 5 evidence" src="https://github.com/user-attachments/assets/33870ee9-1cdd-4c3f-8831-dee8df4842be" />


## Task 6 — Container / Host Hardening

The `hardened` NGINX container was run as non-root user `1000:1000`, with a read-only root filesystem, all Linux capabilities dropped, `no-new-privileges`, and a temporary writable `/tmp` filesystem. The inspect output confirms the non-root user and `ReadOnly=true`.

The Trivy summary for `nginx:alpine (alpine 3.24.1)` reported **0 vulnerabilities**. Its Secrets field is shown as `-` (not scanned), so this report does not claim a secrets-scan result.

<img width="720" height="488" alt="Task 6 1" src="https://github.com/user-attachments/assets/b50421ad-4e67-4113-afd5-ded1321809e8" />


<img width="635" height="193" alt="Task 6 2" src="https://github.com/user-attachments/assets/d6aaf291-9219-4f8e-b817-1dce965aadeb" />


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

<img width="600" height="390" alt="Verification command" src="https://github.com/user-attachments/assets/750f9844-e61d-4e33-be9d-22f2100541ee" />


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

<img width="520" height="279" alt="Cleanup and teardown" src="https://github.com/user-attachments/assets/4bdffe29-3532-4e28-be93-cb9adf48c042" />


## END OF REPORT
