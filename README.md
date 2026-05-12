# Kubernetes cluster setup and application deployment

This project documents how to provision an Amazon EKS cluster and deploy applications to it.

## Prerequisites

- An AWS account with permissions to create EKS clusters, VPCs, IAM roles, and EC2 instances
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured (`aws configure`)
- [eksctl](https://eksctl.io/installation/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

Verify your AWS identity:

```bash
aws sts get-caller-identity
```

## Create the EKS cluster

To create an EKS cluster, run:

```bash
eksctl create cluster \
  --name project3-cluster \
  --nodes-min 1 \
  --nodes 2 \
  --nodes-max 3 \
  --instance-types t3.small
```

This provisions a managed cluster named `project3-cluster` with autoscaling between 1 and 3 nodes and `t3.small` instances for the default node group.

**Notes:**

- Cluster creation usually takes 15–25 minutes and incurs AWS charges.
- After creation, `eksctl` updates your kubeconfig so you can use `kubectl` against the new cluster.

## Verify cluster access

```bash
kubectl get nodes
kubectl cluster-info
```

## Application deployment

Add manifests, Helm charts, or CI/CD steps here as you extend the project (for example `kubectl apply -f ...`).

## Cleanup (optional)

To delete the cluster and related resources:

```bash
eksctl delete cluster --name project3-cluster
```
