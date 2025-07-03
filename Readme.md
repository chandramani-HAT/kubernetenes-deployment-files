# Kubernetes CI/CD Pipeline on Self-Managed AWS EC2 Cluster

This repository provides a Jenkins pipeline and Kubernetes manifests to deploy a full-stack application (frontend, backend, celery, Redis, Postgres) on a self-managed Kubernetes cluster running on AWS EC2. The pipeline automates ECR secrets, Helm-based operators, ALB Ingress, and node providerID patching for AWS integrations.

## Prerequisites

- **Self-managed Kubernetes cluster** on AWS EC2 (created with kubeadm or similar)
- **Jenkins agent** running on the Kubernetes master node (with `kubectl`, `aws`, `helm`, and `jq` installed)
- **IAM instance profile** for all nodes with the following policies:
  - `AmazonEC2FullAccess`
  - `AmazonEKSClusterPolicy`
  - `AmazonEKSServicePolicy`
  - `AmazonEKS_CNI_Policy`
  - `AmazonEC2ContainerRegistryReadOnly`
  - `AmazonRoute53FullAccess`
  - **AWSLoadBalancerControllerIAMPolicy** (see below)
  - Any additional policies for SSM, Secrets Manager, etc.
- **ECR repository** for container images
- **ACM certificate** for HTTPS ALB
- **Kubeconfig** available to Jenkins agent (`/home/ubuntu/.kube/config` by default)
- **Namespace:** `document-expert`

## IAM Policy for ALB Controller

Create and attach the [official AWS Load Balancer Controller IAM policy](https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/latest/download/iam_policy.json) to all EC2 nodes:


## providerID Automation

The pipeline automatically patches all Kubernetes nodes with the correct AWS `providerID` using the AWS CLI and `kubectl`.  
This is required for AWS integrations (like ALB) to map pods to EC2 instances.

## Jenkins Pipeline Overview

The Jenkinsfile automates:

- **Git checkout** of manifests
- **ECR imagePullSecret** creation for pulling images
- **Helm install** of cert-manager, External Secrets Operator, and AWS Load Balancer Controller
- **Manifest application** for all components (Postgres, Redis, Celery, Backend, Frontend, Ingress)
- **providerID patching** for all nodes (required for ALB integration)
- **Verification** of controller deployments and ALB readiness

## Key Pipeline Stages

| Stage                              | Description                                                      |
|------------------------------------|------------------------------------------------------------------|
| Checkout                           | Clone manifests repo                                             |
| Test kubectl                       | Validate cluster access                                          |
| Create namespaces                  | Create required Kubernetes namespaces                            |
| Create ECR imagePullSecret         | Create secret for ECR Docker pulls                               |
| Install Helm                       | Install Helm CLI                                                 |
| Install cert-manager               | Install cert-manager for CRDs and webhook support                |
| Install External Secrets Operator  | Install and verify External Secrets Operator via Helm            |
| Install/Upgrade ALB Controller     | Install AWS Load Balancer Controller via Helm                    |
| Patch providerID for all nodes     | Patch each node with correct AWS providerID                      |
| Apply External Secrets             | Deploy external secrets manifests                                |
| Apply Postgres, Redis, Celery, etc.| Deploy and rollout status for all app components                 |
| Apply Ingress class and Ingress    | Deploy IngressClass and Ingress, wait for ALB readiness          |
|------------------------------------|------------------------------------------------------------------|

## Notes & Best Practices

- **providerID patching** is required for ALB integration on self-managed clusters. The pipeline automates this for all nodes.
- **ALB Controller** requires node IAM roles with the correct policy and providerID set.
- **All manifests** (backend, frontend, etc.) are applied to the `document-expert` namespace.
- **Helm** is used for all operators and controllers to ensure easy upgrades and management.
- **Ingress** and **IngressClass** are managed for ALB routing and HTTPS support.
- **Wait steps** are included to ensure webhooks and CRDs are ready before applying dependent resources.
- **Jenkins agent** must have AWS CLI, kubectl, helm, and jq installed and configured.

## Troubleshooting

- **ALB not created:** Check subnet tags, providerID, IAM policy, and controller logs.
- **504 Gateway Timeout:** Check target group health, pod readiness, and security groups.
- **Webhook errors:** Ensure the webhook pod is running and service endpoints are ready before applying Ingress.

## Example: providerID Patch Script

This script is run automatically in the pipeline to patch all nodes:


## License

MIT

## Authors

- Chandramani

**For more details, see the Jenkinsfile and manifests in this repository.**  
**Feel free to open issues or pull requests for improvements!**
