# Kubernetes cluster setup and application deployment

This project documents how to provision an Amazon EKS cluster and deploy applications to it.

## Prerequisites

- An AWS account with permissions to create EKS clusters, VPCs, IAM roles, and EC2 instances
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured (`aws configure`)
- [eksctl](https://eksctl.io/installation/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/) (for Step 2, AWS Load Balancer Controller)

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

## Step 2: Install AWS Load Balancer Controller

These steps follow the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) install flow. Examples use cluster `project3-cluster` and region `eu-west-3`; change them if yours differ.

### 1. Download the IAM policy document

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
```

### 2. Create the IAM policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

Copy the policy `Arn` from the command output. If the policy already exists, get its ARN:

```bash
aws iam list-policies --query "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy'].Arn" --output text
```

### 3. Enable the IAM OIDC provider on the cluster

```bash
eksctl utils associate-iam-oidc-provider \
  --region=eu-west-3 \
  --cluster=project3-cluster \
  --approve
```

### 4. Create the IRSA (IAM Roles for Service Accounts)

Use the policy ARN from step 2, or build it from your account ID:

```bash
export POLICY_ARN="arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/AWSLoadBalancerControllerIAMPolicy"

eksctl create iamserviceaccount \
  --cluster=project3-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn="${POLICY_ARN}" \
  --override-existing-serviceaccounts \
  --region eu-west-3 \
  --approve
```

### 5. Add and update the Helm repo

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

### 6. Install the controller with Helm

Use the same cluster name as in EKS (`project3-cluster`, not a placeholder).

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=project3-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0
```

### 7. Verify the installation

```bash
kubectl get deployment -n kube-system
```

You should see `aws-load-balancer-controller` among the deployments in `kube-system`.

## Step 3: Configure and install the EBS CSI driver

The [Amazon EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) lets Kubernetes provision EBS volumes for `PersistentVolumeClaim` resources. Ensure the OIDC provider is associated (Step 2.3) before creating the IAM service account.

### 1. Create the IAM role for the EBS CSI controller (IRSA, role only)

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster project3-cluster \
  --region eu-west-3 \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

### 2. Install the `aws-ebs-csi-driver` EKS add-on

Point the add-on at the role created above (account ID is resolved automatically):

```bash
export EBS_CSI_ROLE_ARN="arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/AmazonEKS_EBS_CSI_DriverRole"

eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster project3-cluster \
  --region eu-west-3 \
  --service-account-role-arn "${EBS_CSI_ROLE_ARN}" \
  --force
```

`--force` replaces an existing add-on configuration if the add-on is already present.

### 3. Verify the driver pods

```bash
kubectl get pods -n kube-system -l "app.kubernetes.io/name=aws-ebs-csi-driver"
```

You should see controller and node plugin pods in `Running` state.

## Application deployment

Add manifests, Helm charts, or CI/CD steps here as you extend the project (for example `kubectl apply -f ...`).

## Cleanup (optional)

To delete the cluster and related resources:

```bash
eksctl delete cluster --name project3-cluster
```
