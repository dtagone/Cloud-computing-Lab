# Lab 0: Environment Setup Report

## Purpose

This report documents the local development environment setup and verification, following the supplied **IKB42603 Lab 0 Environment Setup Cheatsheet**. The accompanying screenshots are used as evidence of each completed check.

## 1. Docker

Docker was installed and verified from the command line.

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

**Evidence:** `1. Docker.png`

## 2. AWS CLI

The AWS Command Line Interface was installed and its version was checked.

```text
aws --version
aws-cli/2.36.9 Python/3.14.6 Windows/11 exe/AMD64
```

**Result:** AWS CLI v2 is installed and available from the command line.

**Evidence:** `2. AWS.png`

## 3. kind and kubectl

The Kubernetes-in-Docker (`kind`) tool and Kubernetes command-line client (`kubectl`) were installed and verified.

```text
kind --version
kind version 0.31.0

kubectl version --client
Client Version: v1.36.3
Kustomize Version: v5.8.1
```

**Result:** Both `kind` and `kubectl` are installed and ready for local Kubernetes management.

**Evidence:** `3. Kind and Kubectl.png`

## 4. Helper Tools

Git Bash was opened successfully, providing a Unix-like terminal for running the lab commands.

**Result:** The helper terminal environment is available.

**Evidence:** `4. Helper Tools.png`

## 5. Local Kubernetes Cluster

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

**Evidence:** `5. Kubernetes cluster (kind).png`

## 6. LocalStack

LocalStack was started in Docker and exposed on port `4566`.

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

**Evidence:** `5. Localstack.png`

## 7. AWS CLI Configuration for LocalStack

The AWS CLI was configured with LocalStack test credentials and a default region.

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

**Evidence:** `6. AWS CLI Configuration.png`

## Conclusion

The required Lab 0 environment has been set up and verified. Docker, AWS CLI, `kind`, `kubectl`, Git Bash, LocalStack, and the LocalStack AWS CLI configuration are operational. The Kubernetes cluster creation and removal test also completed successfully.
