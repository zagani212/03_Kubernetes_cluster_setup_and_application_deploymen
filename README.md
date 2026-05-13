# Kubernetes cluster setup and application deployment

This project documents how to provision an Amazon EKS cluster and deploy applications to it.

## Prerequisites

- An AWS account with permissions to create EKS clusters, VPCs, IAM roles, and EC2 instances
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured (`aws configure`)
- [eksctl](https://eksctl.io/installation/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/) (for the AWS Load Balancer Controller in Step 2 and for the application charts in the [Helm application deployment](#helm-application-deployment) section)

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

## Step 4: Create environments and apply ResourceQuotas

Create the **dev**, **staging**, and **prod** namespaces, then apply the [`ResourceQuota`](https://kubernetes.io/docs/concepts/policy/resource-quotas/) manifests in [`resourcesQuotas/`](resourcesQuotas/) to each namespace.

Run these commands from the repository root:

### 1. Create namespaces

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod
```

If a namespace already exists, `kubectl create namespace` fails; you can ignore that or use `kubectl get namespace` to confirm.

### 2. Apply ResourceQuota manifests

The quota YAML files do not set `metadata.namespace`; target the namespace with `-n`:

```bash
kubectl apply -f resourcesQuotas/dev.yml -n dev
kubectl apply -f resourcesQuotas/staging.yml -n staging
kubectl apply -f resourcesQuotas/prod.yml -n prod
```

### 3. Verify

```bash
kubectl get resourcequota -n dev
kubectl get resourcequota -n staging
kubectl get resourcequota -n prod
```

## Helm application deployment

This repository ships three local Helm charts under [`db/`](db/), [`back/`](back/), and [`frontend/`](frontend/). They are intended for the same namespace as your environment (the examples below use **`dev`**; use `staging` or `prod` if you created those namespaces and quotas in Step 4).

### Install or upgrade the Helm CLI

If Helm is not already installed, follow the [official installation guide](https://helm.sh/docs/intro/install/). Confirm it works:

```bash
helm version
```

### What each chart does

| Chart path | Release name (example) | Main resources |
|------------|-------------------------|----------------|
| [`db/`](db/) | `project3-db` | `StorageClass` (name from [`db/values.yaml`](db/values.yaml) `volume.name`, default `ebs-sc-1`), `Secret` `db-credentials`, `StatefulSet` and `Service` **`db`** (Postgres 16) |
| [`back/`](back/) | `project3-backend` | `ConfigMap` `backend-app` (Node app + `package.json`), `Deployment` and `Service` **`backend`** (public `node:20-alpine` image; init container runs `npm install`) |
| [`frontend/`](frontend/) | `project3-frontend` | `ConfigMap` for HTML, `Deployment` and `Service` **`nginx`** (see [`frontend/Chart.yaml`](frontend/Chart.yaml)), public `nginx` image, `LoadBalancer` with AWS NLB annotations from [`frontend/values.yaml`](frontend/values.yaml) |

The backend chart expects Postgres at host **`db`** on port **5432** and secret **`db-credentials`** with `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` (matching the `db` chart). Override `container.pgHost` or `secret.name` in [`back/values.yaml`](back/values.yaml) if you change those in the database chart.

### Prerequisites on the cluster

1. **EBS CSI driver** (Step 3): the `db` chart provisions a PVC using the `StorageClass` rendered from [`db/templates/storage-class.yml`](db/templates/storage-class.yml) (`ebs.csi.aws.com`, volume type from `volume.type` in values, default gp2). Without the driver, the Postgres pod stays pending.
2. **AWS Load Balancer Controller** (Step 2): recommended for the frontend `Service` type `LoadBalancer` so Kubernetes can provision an AWS load balancer (NLB per your annotations).
3. **Namespace**: create `dev` (or your target namespace) before installing, for example as in Step 4.

### Install the charts (from the repository root)

Use `helm upgrade --install` so the same commands work for first install and later upgrades. Pick release names you prefer; the examples use `project3-db`, `project3-backend`, and `project3-frontend`.

**1. Database (install first)**

```bash
helm upgrade --install project3-db ./db -n dev
```

Wait until Postgres is ready:

```bash
kubectl rollout status statefulset/db -n dev
kubectl get pods,svc,pvc -n dev -l app=db
```

**2. Backend API**

```bash
helm upgrade --install project3-backend ./back -n dev
```

```bash
kubectl rollout status deployment/backend -n dev
```

**3. Frontend (nginx + LoadBalancer)**

The frontend chart reads [`frontend/values.yaml`](frontend/values.yaml) (including `namespace: dev`). Install into the same namespace:

```bash
helm upgrade --install project3-frontend ./frontend -n dev
```

Check the load balancer hostname or IP:

```bash
kubectl get svc nginx -n dev
```

### Override values at install time

You can pass overrides without editing files, for example a stronger database password or a different namespace for the frontend chart’s namespaced resources:

```bash
helm upgrade --install project3-db ./db -n dev \
  --set secret.password='your-secure-password'

helm upgrade --install project3-frontend ./frontend -n staging \
  --set namespace=staging
```

For larger changes, copy a values file and pass `-f my-values.yaml`.

### Lint and dry-run

```bash
helm lint ./db ./back ./frontend
helm upgrade --install project3-db ./db -n dev --dry-run=client
```

### Upgrade and uninstall

After changing chart templates or values:

```bash
helm upgrade project3-db ./db -n dev
helm upgrade project3-backend ./back -n dev
helm upgrade project3-frontend ./frontend -n dev
```

Remove a release (namespaced resources for that release are removed; the `StorageClass` created by the `db` chart is cluster-scoped and may remain until you delete it separately if needed):

```bash
helm uninstall project3-frontend -n dev
helm uninstall project3-backend -n dev
helm uninstall project3-db -n dev
```

## Cleanup (optional)

To delete the cluster and related resources:

```bash
eksctl delete cluster --name project3-cluster
```
