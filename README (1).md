# AWS EKS + Argo CD GitOps Installation Guide

This guide explains how to prepare an Ubuntu machine, create an Amazon
EKS cluster using an **eksctl YAML configuration file**, install Argo
CD, connect a GitHub repository, and deploy an application.

> **Important:** This guide uses `eksctl` with a cluster configuration
> YAML file. It does not use the `eksctl create cluster` command with
> all settings supplied as command-line flags.

------------------------------------------------------------------------

## 1. Architecture overview

``` text
Developer
   |
   | writes code and pushes changes
   v
GitHub Repository
   |
   | stores Kubernetes YAML files or Helm charts
   v
Argo CD (inside EKS)
   |
   | compares Git desired state with cluster state
   | synchronizes Kubernetes resources
   v
Amazon EKS Cluster
   |
   +-- Control Plane (managed by AWS)
   |
   +-- Worker Nodes
         |
         +-- Application Deployments
         +-- Pods
         +-- Services
         +-- ConfigMaps / Secrets
         +-- Gateway or Ingress (if configured)
   |
   v
Users access the application through an exposed endpoint
```

## 2. Prerequisites: install tools on Ubuntu

  Tool      Purpose
  --------- ----------------------------------------------------
  AWS CLI   Configure and communicate with AWS
  kubectl   Manage Kubernetes resources
  eksctl    Create and manage EKS clusters
  Helm      Install Kubernetes packages and charts
  Git       Manage source code and GitOps configuration
  Docker    Build container images, if building your own image

### 2.1 Update Ubuntu

``` bash
sudo apt update
sudo apt install -y curl unzip git
```

### 2.2 Install AWS CLI

``` bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Configure credentials using an approved AWS authentication method. For a
learning environment:

``` bash
aws configure
```

Check the identity:

``` bash
aws sts get-caller-identity
```

Do not commit AWS access keys or credentials to GitHub.

### 2.3 Install kubectl

Use the official Kubernetes installation instructions to select a
version compatible with your EKS cluster:

<https://kubernetes.io/docs/tasks/tools/>

Verify:

``` bash
kubectl version --client
```

### 2.4 Install eksctl

``` bash
ARCH=amd64
PLATFORM="$(uname -s)_${ARCH}"

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
tar -xzf "eksctl_${PLATFORM}.tar.gz"
sudo install -m 0755 eksctl /usr/local/bin/eksctl

eksctl version
```

### 2.5 Install Helm

``` bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### 2.6 Verify Git

``` bash
git --version
```

### 2.7 Install Docker (only if required)

Docker is needed when you build your own application image. If you
already use an image from a registry, Docker may not be needed on this
machine.

``` bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
docker --version
```

For Docker access without `sudo`, add your user to the Docker group and
then log out and back in:

``` bash
sudo usermod -aG docker "$USER"
```

Membership in the Docker group grants powerful privileges on the
machine. Use it only when appropriate.

------------------------------------------------------------------------

## 3. Create the EKS cluster using a YAML file

Create a directory for the cluster configuration:

``` bash
mkdir -p ~/gocart/eks-config
cd ~/gocart/eks-config
```

Create a file named **`cluster.yaml`**:

``` yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: gocart-cluster
  region: ap-south-1

# Enable IAM OIDC for add-ons that need AWS permissions
iam:
  withOIDC: true

# Amazon EKS managed cluster add-ons
addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: metrics-server
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    wellKnownPolicies:
      ebsCSIController: true

nodeGroups:
  - name: worker-nodes
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 3
```

### Understand the YAML fields

  Field               Meaning
  ------------------- ----------------------------------------------
  `apiVersion`        eksctl configuration API version
  `kind`              Defines this file as a cluster configuration
  `metadata.name`     EKS cluster name
  `metadata.region`   AWS region
  `nodeGroups`        Defines the worker node group
  `instanceType`      EC2 instance type for worker nodes
  `desiredCapacity`   Number of worker nodes to start with
  `minSize`           Minimum number of worker nodes
  `maxSize`           Maximum number of worker nodes

> This is a basic example using a managed-by-eksctl node group
> configuration. Review the current eksctl schema and your AWS account
> limits before creating a cluster. EKS incurs AWS charges.

### Cluster add-ons included in this configuration

| Add-on | Purpose |
|---|---|
| `vpc-cni` | Provides pod networking and IP addresses in the VPC |
| `coredns` | Provides DNS-based service discovery inside Kubernetes |
| `kube-proxy` | Maintains network rules for Kubernetes Services |
| `metrics-server` | Provides CPU and memory metrics for commands and features such as `kubectl top` and HPA |
| `aws-ebs-csi-driver` | Allows Kubernetes workloads to use Amazon EBS persistent volumes |

The first three are core networking add-ons. Metrics Server provides resource
metrics. The EBS CSI driver is useful when workloads need persistent block
storage. The EBS CSI driver needs AWS IAM permissions;
`wellKnownPolicies.ebsCSIController` asks eksctl to configure its recommended
policy integration. Keep `iam.withOIDC: true` enabled for this configuration.
Review the current AWS permissions guidance before production use.

`version: latest` asks eksctl to select the latest compatible add-on version
for the cluster. You can omit `version` to let eksctl select its default, or
pin a tested version for controlled production upgrades. Use a recent eksctl
release that supports the add-ons in this file (Metrics Server is supported
as an EKS add-on in eksctl v0.201.0 and later).

### 3.1 Validate the configuration

``` bash
eksctl create cluster --config-file cluster.yaml --dry-run
```

### 3.2 Create the EKS cluster

``` bash
eksctl create cluster --config-file cluster.yaml
```

Cluster creation can take several minutes.

### 3.3 Update kubeconfig

``` bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name gocart-cluster
```

### 3.4 Verify the cluster

``` bash
kubectl get nodes
kubectl get namespaces
kubectl cluster-info
```

Worker nodes should eventually show `Ready`.

### 3.5 Verify installed cluster add-ons

List the EKS managed add-ons:

``` bash
eksctl get addons --cluster gocart-cluster --region ap-south-1
```

Check add-on Pods in the system namespace:

``` bash
kubectl get pods -n kube-system
```

Check resource metrics (requires Metrics Server to be ready):

``` bash
kubectl top nodes
kubectl top pods -A
```

If `kubectl top` reports that metrics are unavailable, check the Metrics
Server add-on status and its Pods before troubleshooting further.

------------------------------------------------------------------------

## 4. Install Argo CD inside EKS

### 4.1 Create the Argo CD namespace

``` bash
kubectl create namespace argocd
```

### 4.2 Install Argo CD

``` bash
kubectl apply -n argocd \
  --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

For production, pin and review a specific Argo CD release rather than
relying on the moving `stable` manifest URL.

### 4.3 Check Argo CD resources

``` bash
kubectl get pods -n argocd
kubectl get services -n argocd
kubectl get deployments -n argocd
```

Wait until the required Pods are ready.

### 4.4 Access the Argo CD dashboard

Run this command and keep the terminal session open:

``` bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open this URL in a browser on the same machine:

``` text
https://localhost:8080
```

If the cluster is on a remote Ubuntu EC2 instance, use an SSH tunnel
from your local computer:

``` bash
ssh -L 8080:localhost:8080 ubuntu@YOUR_EC2_PUBLIC_IP
```

Then open `https://localhost:8080` on your local computer. Do not expose
the Argo CD server publicly just to access the dashboard.

### 4.5 Get the initial admin password

``` bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

-   Username: `admin`
-   Password: output of the command above

Change the initial password after first login and follow your
organization's authentication practices. The initial secret should not
be stored in Git.

------------------------------------------------------------------------

## 5. Prepare the GitHub GitOps repository

Create a GitHub repository, for example:

``` text
gocart-gitops
```

Example structure:

``` text
gocart-gitops/
├── deployment/
│   ├── frontend.yaml
│   └── backend.yaml
├── service/
│   ├── frontend-service.yaml
│   └── backend-service.yaml
├── configmap/
│   └── app-config.yaml
└── README.md
```

The `deployment` directory in this example will be the source path for
the Argo CD Application. If your manifests are spread across multiple
directories, use a suitable repository structure and configure one or
more Argo CD Applications accordingly.

### Push the repository to GitHub

From your local project directory:

``` bash
git init
git add .
git commit -m "Add GitOps Kubernetes manifests"
git branch -M main
git remote add origin https://github.com/USERNAME/gocart-gitops.git
git push -u origin main
```

Replace `USERNAME` and the repository URL with your own values.

Do not commit plain-text passwords, access keys, tokens, or other
secrets. Use an appropriate secret-management solution.

------------------------------------------------------------------------

## 6. Create an Argo CD Application

Create a file named **`application.yaml`**:

``` yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: gocart
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/USERNAME/gocart-gitops.git
    targetRevision: main
    path: deployment

  destination:
    server: https://kubernetes.default.svc
    namespace: gocart

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Replace the repository URL with your actual GitHub repository.

### Understand the important fields

  -----------------------------------------------------------------------
  Field                               Meaning
  ----------------------------------- -----------------------------------
  `repoURL`                           GitHub repository URL

  `targetRevision`                    Branch, tag, or commit to track

  `path`                              Directory containing the deployment
                                      manifests

  `destination.server`                Kubernetes API destination; this
                                      URL refers to the cluster where
                                      Argo CD is installed

  `destination.namespace`             Namespace where application
                                      resources will be deployed

  `automated`                         Enables automatic synchronization

  `prune`                             Removes managed resources deleted
                                      from Git

  `selfHeal`                          Reverts live drift to the
                                      Git-defined state

  `CreateNamespace=true`              Creates the destination namespace
                                      if it does not exist
  -----------------------------------------------------------------------

**Caution:** `prune: true` can delete resources removed from Git. Review
changes before using automated pruning in production.

Apply the Application:

``` bash
kubectl apply -f application.yaml
```

Check it:

``` bash
kubectl get applications -n argocd
```

For a private GitHub repository, configure repository credentials
securely in Argo CD before syncing.

------------------------------------------------------------------------

## 7. Verify the application

``` bash
kubectl get applications -n argocd
kubectl get pods -n gocart
kubectl get deployments -n gocart
kubectl get services -n gocart
kubectl get all -n gocart
```

In the Argo CD dashboard, check:

-   **Synced:** Git desired state matches the cluster's managed state.
-   **OutOfSync:** Git and the live state differ.
-   **Healthy:** Resources are reporting healthy status.
-   **Progressing:** Resources are still being created or updated.
-   **Degraded:** One or more resources are unhealthy.

> A running Pod does not automatically make an application accessible
> from the internet. Configure an appropriate Service and, if external
> access is required, a supported Ingress or Gateway controller and AWS
> networking.

------------------------------------------------------------------------

## 8. How automatic GitOps deployment works

For example, suppose the frontend Deployment has two replicas:

``` yaml
spec:
  replicas: 2
```

Change it to four replicas:

``` yaml
spec:
  replicas: 4
```

Commit and push the change:

``` bash
git add .
git commit -m "Scale frontend to four replicas"
git push origin main
```

With automated synchronization enabled:

1.  The change is pushed to GitHub.
2.  Argo CD detects the updated repository state.
3.  Argo CD compares the desired state with the live Kubernetes state.
4.  Argo CD synchronizes the change.
5.  Kubernetes adjusts the frontend Deployment to four replicas.

This process is called **continuous reconciliation**.

------------------------------------------------------------------------

## 9. Optional components

These components are useful for additional capabilities, but they are
not all required to install Argo CD.

  -----------------------------------------------------------------------
  Component                           Purpose
  ----------------------------------- -----------------------------------
  AWS Load Balancer Controller        Integrates Kubernetes load balancer
                                      resources with AWS load balancing

  Envoy Gateway or another Gateway    Implements Gateway API traffic
  controller                          routing

  Prometheus                          Collects metrics

  Grafana                             Displays monitoring dashboards

  Loki                                Collects and queries logs
  -----------------------------------------------------------------------

Metrics Server is already included in the `cluster.yaml` add-ons list above.
The AWS Load Balancer Controller and Gateway controller are separate
installations, not part of the EKS managed add-ons listed in that YAML.
Install only the components your architecture needs. For example, an
application exposed through an AWS load balancer needs an appropriate
load-balancing and routing setup.

------------------------------------------------------------------------

## 10. Complete installation checklist

### Ubuntu machine

-   [ ] Install AWS CLI
-   [ ] Configure AWS authentication
-   [ ] Verify AWS identity
-   [ ] Install kubectl
-   [ ] Install eksctl
-   [ ] Install Helm
-   [ ] Install Git
-   [ ] Install Docker if building images

### AWS EKS

-   [ ] Create `cluster.yaml`
-   [ ] Validate the eksctl configuration
-   [ ] Create EKS cluster using the YAML file
-   [ ] Update kubeconfig
-   [ ] Verify worker nodes are Ready
-   [ ] Verify EKS add-ons (VPC CNI, CoreDNS, kube-proxy, Metrics Server, EBS CSI driver)

### Argo CD

-   [ ] Create `argocd` namespace
-   [ ] Install Argo CD
-   [ ] Verify Argo CD Pods and Services
-   [ ] Access the dashboard securely
-   [ ] Retrieve the initial password and change it

### GitOps deployment

-   [ ] Create GitHub repository
-   [ ] Add Kubernetes manifests
-   [ ] Push files to GitHub
-   [ ] Create `application.yaml`
-   [ ] Connect GitHub repository to Argo CD
-   [ ] Synchronize the Application
-   [ ] Verify Pods, Deployments, and Services
-   [ ] Configure external access if required

------------------------------------------------------------------------

## 11. Quick reference: important commands

``` bash
# AWS identity
aws sts get-caller-identity

# Validate EKS cluster configuration
eksctl create cluster --config-file cluster.yaml --dry-run

# Create EKS cluster from YAML
eksctl create cluster --config-file cluster.yaml

# Update kubeconfig
aws eks update-kubeconfig --region ap-south-1 --name gocart-cluster

# Check nodes
kubectl get nodes

# List EKS managed cluster add-ons
eksctl get addons --cluster gocart-cluster --region ap-south-1

# Check system add-on Pods
kubectl get pods -n kube-system

# Check node and Pod resource metrics (requires Metrics Server)
kubectl top nodes
kubectl top pods -A

# Check Argo CD
kubectl get pods -n argocd
kubectl get svc -n argocd

# Check Argo CD Applications
kubectl get applications -n argocd

# Check application resources
kubectl get all -n gocart
```

## 12. Official documentation

-   AWS CLI: <https://docs.aws.amazon.com/cli/latest/userguide/>
-   Amazon EKS: <https://docs.aws.amazon.com/eks/latest/userguide/>
-   eksctl cluster configuration: <https://eksctl.io/usage/schema/>
-   eksctl installation: <https://eksctl.io/installation/>
-   kubectl: <https://kubernetes.io/docs/reference/kubectl/>
-   Helm: <https://helm.sh/docs/>
-   Argo CD: <https://argo-cd.readthedocs.io/>
-   Docker: <https://docs.docker.com/>

------------------------------------------------------------------------

## Final learning goal

Understand and practice the complete flow:

**Ubuntu tools → eksctl YAML → AWS EKS cluster → Argo CD installation →
GitHub repository → Argo CD Application → Kubernetes resources → Running
application.**

The EKS cluster is created first using the `cluster.yaml` configuration
file. Argo CD is then installed inside that cluster and manages
application resources from Git.
