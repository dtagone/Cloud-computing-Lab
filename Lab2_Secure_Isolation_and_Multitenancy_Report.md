# Lab 2 Report — Secure Isolation & Multi-Tenancy

Name: Muhammad A'beed bin Firdaus 52215124303

Subject: Cloud Computing Security Essentials

Code: IKB 42603

Lecturer: Prof. Dr. Shahrulniza Musa

Date: 9 Aug 2026

## Purpose

This lab demonstrates how a shared Kubernetes cluster can isolate multiple tenants across compute, network, and storage. It models two tenants, exposes the risk of default-open networking, applies resource and network controls, restricts secret access through RBAC, and demonstrates the need for secure deletion.

## Requirements

- A computer with at least 8 GB RAM and administrator rights.
- Docker Desktop / Docker Engine.
- `kind` and `kubectl`.
- Internet access for the initial image and Calico download.

## Environment setup

A `kind` cluster named `ccse-lab2` was created with the default CNI disabled and pod subnet `192.168.0.0/16`. Calico v3.27.0 was then installed, and the `calico-node` DaemonSet rollout completed successfully. Calico is required because a default `kind` cluster does not enforce Kubernetes `NetworkPolicy` objects.

<img width="803" alt="Lab 2 cluster setup" src="Lab%202%20Setup%20Part%201.png" />

<img width="803" alt="Calico installation and rollout" src="Lab%202%20Setup%20Part%202.png" />

## Task 1 — Two tenants on one cluster

Two namespaces, `tenant-a` and `tenant-b`, were created to represent separate customers sharing the same Kubernetes cluster. An NGINX `web` deployment and a ClusterIP service were created in each namespace. The evidence shows that the `tenant-a` web pod was running and its service was available on port 80.

Namespaces establish logical compute and administrative boundaries, but they do not automatically prevent traffic between workloads.

<img width="803" alt="Task 1 tenant namespaces, workloads, and service" src="Task%201.png" />

## Task 2 — Default-open network risk

The `tenant-b` web service had ClusterIP `10.96.221.178`. A curl probe launched from `tenant-a` reached that address and returned `HTTP 200`. This proves that, before a NetworkPolicy was applied, one tenant could access another tenant's service across namespaces.

This is a multi-tenancy risk: a namespace is an organisational boundary, not a network firewall. A compromised or unintended workload in one tenant could contact services belonging to another tenant unless traffic is explicitly restricted.

<img width="803" alt="Task 2 cross-tenant probe returns HTTP 200" src="Task%202.png" />

## Task 3 — Resource quota to contain a noisy neighbour

The `tenant-a-quota` ResourceQuota was created with hard limits of one CPU request, 512 MiB of memory requests, and five pods. The quota status shows one existing pod in use, leaving capacity for only four additional pods.

This is compute isolation: it limits the amount of shared cluster capacity a tenant can consume and reduces the risk that one noisy tenant exhausts the node for the other tenant.

<img width="803" alt="Task 3 tenant-a resource quota" src="Task%203.png" />

## Task 4 — Default-deny network isolation

A `default-deny-ingress` NetworkPolicy was created in `tenant-b` with an empty pod selector and `Ingress` policy type. It selects every pod in `tenant-b` and denies ingress unless another policy explicitly allows it. This implements segmentation by denying access by default and allowing only approved flows by exception.

The recorded re-test did **not** reach the NetworkPolicy check: Kubernetes rejected the new `probe` pod because `tenant-a-quota` requires CPU and memory requests, which the command did not supply. Thus, the screenshot proves that the policy was created and the quota was enforced, but it does not prove the required post-policy timeout. A valid follow-up probe would include resource requests (for example `--requests=cpu=100m,memory=64Mi`) and should time out when attempting to reach the tenant-b service.

<img width="803" alt="Task 4 default-deny policy and quota admission rejection" src="Task%204.png" />

## Task 5 — Storage and secret isolation

Each tenant received a distinct `data` Secret. Service account `app-a` was created only in `tenant-a` and bound to the namespaced `reader` Role, which grants `get` on Secrets. The authorisation tests show:

| Identity and request | Result | Meaning |
|---|---:|---|
| `app-a` gets Secrets in `tenant-a` | yes | The tenant-a RoleBinding grants the requested access. |
| `app-a` gets Secrets in `tenant-b` | no | The tenant-a RoleBinding has no authority in tenant-b. |

This demonstrates storage isolation through least-privilege, namespaced Kubernetes RBAC: an identity can read only the tenant data explicitly assigned to it.

<img width="803" alt="Task 5 secret isolation using RBAC" src="Task%205.png" />

## Task 6 — Data remanence and secure deletion

The first container wrote `SENSITIVE-PATIENT-RECORD` into the shared Docker volume, synchronised it, and removed the file normally. The scan completed but did not display the removed text; therefore the screenshot does not itself prove residual bytes were recoverable. It still demonstrates the normal-delete step used to discuss data remanence.

For the second file, `dd` overwrote the first 1 KiB with zeros before removal and reported `wiped`. Overwriting before deletion reduces the chance of recoverable data on media where overwrite semantics apply. In cloud storage, physical blocks are normally not directly controlled by a tenant; cryptographic erasure—destroying the encryption key—is the preferred practical defence against data remanence.

<img width="803" alt="Task 6 normal deletion and overwrite before deletion" src="Task%206.png" />

## Short-answer questions

### Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?

Kubernetes namespaces primarily scope names and many administrative objects; they are not default network barriers. Without an enforcing NetworkPolicy, the cluster network ordinarily permits pod-to-pod traffic, including traffic across namespaces. In a multi-tenant environment this is dangerous because a compromised tenant workload can discover, probe, or communicate with another tenant's services, expanding the incident's blast radius.

### Q2. Explain the default-deny principle and how your NetworkPolicy implements it.

Default-deny means access is blocked unless an explicit rule permits it. The `default-deny-ingress` policy selects all pods in `tenant-b` and declares `Ingress` policy enforcement without any allow rules. Consequently, ingress to those pods is denied by default; a separate allow policy would be required for approved traffic, such as same-namespace communication.

### Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?

Containers share the host operating system kernel, so their isolation relies on kernel namespaces, cgroups, runtime settings, and policy controls. Virtual machines provide a stronger boundary by virtualising hardware and running a separate guest kernel for each VM. A VM boundary is appropriate for untrusted tenants, different security classifications, workloads with high compromise impact, or cases where shared-kernel risk is unacceptable.

### Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?

Data remanence is residual data that may remain on storage after a file is deleted. Cloud tenants generally cannot address or reliably overwrite the underlying physical blocks, especially when storage is replicated, thin-provisioned, or managed by the provider. With encryption at rest, destroying the relevant encryption key makes all encrypted copies unreadable, which is why cryptographic erasure is more practical and reliable in cloud environments.

### Q5. Which of the three isolation dimensions—compute, network, storage—did each task exercise?

| Task | Isolation dimension | How it was exercised |
|---|---|---|
| Task 1 | Compute | Tenant workloads were separated into distinct Kubernetes namespaces. |
| Task 2 | Network | The successful cross-namespace HTTP probe exposed the default-open network risk. |
| Task 3 | Compute | A ResourceQuota constrained tenant-a's pod, CPU, and memory-request usage. |
| Task 4 | Network | A default-deny ingress policy was created for tenant-b. |
| Task 5 | Storage | Secrets were scoped per tenant and protected with namespaced RBAC. |
| Task 6 | Storage | Normal deletion, overwrite-before-delete, and cryptographic erasure were considered for data remanence. |

## Required verification commands

The lab specifies these commands to confirm the network policy and tenant-a quota:

```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

The supplied Task 3 evidence contains the required `describe resourcequota` output. No separate screenshot of `kubectl get networkpolicy -A` was supplied. The Task 4 evidence does confirm creation of `default-deny-ingress` in `tenant-b`.

## Security best-practices checklist

- [x] Tenants were separated into distinct namespaces.
- [ ] A default-deny NetworkPolicy was fully verified with the required before/after probe: the policy was created, but the recorded after-probe was blocked by ResourceQuota before a timeout could be observed.
- [x] A ResourceQuota constrained tenant-a's shared-capacity use.
- [x] RBAC prevented the tenant-a service account from reading tenant-b Secrets.
- [x] Secure deletion and cryptographic erasure were addressed for data remanence.

## Cleanup and teardown

The `ccse-lab2` kind cluster and the `ccse-vol` Docker volume were removed after the lab.

<img width="803" alt="Lab 2 cleanup and teardown" src="Cleanup%20and%20Teardown.png" />

## End of report
