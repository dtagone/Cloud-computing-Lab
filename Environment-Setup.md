# Lab 0: Environment Setup Report
Name: Muhammad A'beed bin Firdaus 52215124303

Subject: Cloud computing security essentials

Code: IKB 42603

Date: 28 July 2026

## Purpose

This report documents the local development environment setup and verification for future lab usage. The accompanying screenshots are used as evidence of each completed check.

## 1. Docker

Docker was installed and verified from the command line from the docker official website. Ensure WSL2 is picked when prompted.

```text
docker --version
Docker version 29.6.2, build dfc4efb
```

A test container was also run successfully.

```text
docker run --rm hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

**Result:** Docker Engine is installed and can pull and run containers.

**Evidence:** 

<img width="757" height="547" alt="1  Docker" src="https://github.com/user-attachments/assets/9fcc69c8-ccdf-47ef-8709-1c0b00cd5d68" />


## 2. AWS CLI

The AWS Command Line Interface was installed and its version was checked. Installed from offical AWS v2 msi file.

```text
aws --version
aws-cli/2.36.9 Python/3.14.6 Windows/11 exe/AMD64
```

**Result:** AWS CLI v2 is installed and available from the command line.

**Evidence:** 

<img width="622" height="72" alt="2  AWS" src="https://github.com/user-attachments/assets/62e14246-6afd-41dc-88fc-a7d41b5cd82a" />


## 3. kind and kubectl

The Kubernetes-in-Docker (`kind`) tool and Kubernetes command-line client (`kubectl`) were installed and verified with the use of choco through cmd.

```text
kind --version
kind version 0.31.0

kubectl version --client
Client Version: v1.36.3
Kustomize Version: v5.8.1
```

**Result:** Both `kind` and `kubectl` are installed and ready for local Kubernetes management.

**Evidence:** 

<img width="487" height="155" alt="3  Kind and Kubectl" src="https://github.com/user-attachments/assets/1c5efe1d-2c74-4669-870a-3b74683b9ffa" />


## 4. Helper Tools

Git Bash was installed successfully, providing a Unix-like terminal for running the lab commands.

**Result:** The helper terminal environment is available.

**Evidence:** 

<img width="742" height="443" alt="4  Helper Tools" src="https://github.com/user-attachments/assets/b625bdb5-5d88-494a-833d-4e5a7d7b0783" />


## 5. Start and Stop the Lab Environment 
(Local AWS)

LocalStack was started in Docker and exposed on port `4566`. An old version is used so credentials aren't needed.

```text
docker run -d --name localstack -p 4566:4566 localstack/localstack:4.14.0
```

Its health endpoint returned an available status for LocalStack services:

```text
curl http://localhost:4566/_localstack/health
```

The container lifecycle was also checked:

```text
docker stop localstack
docker start localstack
```

**Result:** LocalStack is running as a Docker container and its AWS-compatible local service endpoint is healthy.

**Evidence:** 

<img width="752" height="477" alt="5  Localstack" src="https://github.com/user-attachments/assets/db62e0f7-22c1-4d5f-a8fb-1c129b319b32" />




(Local Kubernetes Cluster)

A local Kubernetes cluster named `ccse` was created using `kind`.

```text
kind create cluster --name ccse
```

The cluster was verified with the following commands:

```text
kubectl cluster-info --context kind-ccse
kubectl get nodes
```

The output showed that the Kubernetes control plane and CoreDNS were running, and that node `ccse-control-plane` had status `Ready` with Kubernetes version `v1.35.0`.

The cluster was then removed as part of the lifecycle test:

```text
kind delete cluster --name ccse
```

**Result:** A local Kubernetes cluster can be created, accessed, checked, and deleted successfully.

**Evidence:** 

<img width="922" height="837" alt="5  Kubernetes cluster (kind)" src="https://github.com/user-attachments/assets/5416ceae-b860-4036-b62f-4339e8251f58" />


## 6. AWS CLI Configuration for LocalStack

The AWS CLI was configured with LocalStack test credentials and a default region. Command is ran on GitBash.

```text
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```

An endpoint variable was set for LocalStack:

```text
EP='--endpoint-url=http://localhost:4566'
```

The connection was verified using AWS STS:

```text
aws $EP sts get-caller-identity
```

The command returned the LocalStack test identity, including account ID `000000000000` and ARN `arn:aws:iam::000000000000:root`.

**Result:** The AWS CLI is configured correctly to communicate with LocalStack rather than a live AWS account.

**Evidence:** 

<img width="431" height="352" alt="6  AWS CLI Configuration" src="https://github.com/user-attachments/assets/bd408651-4bea-4f91-a58f-5ea28b53a046" />


## Conclusion

The required Lab 0 environment has been set up and verified. Docker, AWS CLI, `kind`, `kubectl`, Git Bash, LocalStack, and the LocalStack AWS CLI configuration are operational. The Kubernetes cluster creation and removal test also completed successfully.
