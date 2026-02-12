# 🚀 iac-actions — GitOps Infrastructure Automation (AWS EKS + Terraform + GitHub Actions + Ansible)
Terraform AWS EKS GitHub Actions Ansible Helm GitOps

**iac-actions** provisions an **AWS EKS cluster** (plus VPC, Bastion, and ECR) using **Terraform**, then installs core DevOps tooling into Kubernetes using **Ansible + Helm** — all automated by **GitHub Actions** on every push to `main`.

---

## ✨ What this project deploys

### AWS Infrastructure (Terraform)
- **VPC** with public + private subnets + NAT Gateway
- **EKS Cluster** (public + private endpoint access enabled)
- **Managed node group** with autoscaling (min 2, desired 3, max 4)
- **Bastion Host** (public EC2) used to tunnel securely to the EKS API endpoint
- **ECR Repository** for container images
- **AWS Secrets Manager secret** placeholder (Cosign signing keys)

### Kubernetes Tooling (Ansible + Helm)
- **Argo CD** (GitOps CD)
- **Monitoring**: kube-prometheus-stack (Prometheus + Grafana + alerting stack)
- **Kyverno** (policy engine)
- **CloudWatch logging** using **aws-for-fluent-bit** (logs shipped to a CloudWatch Log Group)

---

## ⚙️ How the automation works (GitHub Actions)

Workflow: `.github/workflows/iac.yml`  
Name: `infra-gitops`

### Triggers
- **Push to `main`** when files change under:
  - `terraform/**`
  - `ansible/**`
- **Manual trigger** (`workflow_dispatch`) to destroy infrastructure

### Jobs
1) **terraform** (runs on push)
- Terraform `init → validate → plan → apply`
- Configures `kubectl` for EKS
- Opens an **SSH tunnel through the Bastion** to the EKS API endpoint and points kubeconfig to `https://localhost:8443`
- Installs **ingress-nginx** (AWS provider manifest)

2) **install_tools** (runs on push; depends on terraform)
- Configures `kubectl` for EKS
- Opens the same Bastion tunnel to reach the EKS API securely
- Runs Ansible playbook to install ArgoCD + monitoring + kyverno + fluent-bit

3) **destroy** (manual)
- Opens tunnel
- Removes ingress-nginx
- Empties ECR images (so the repo can be deleted)
- Runs `terraform destroy`

---

## 🔐 GitHub repository configuration (Secrets & Variables)

### Secrets (Repo → Settings → Secrets and variables → Actions)
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `BUCKET` (Terraform remote-state bucket name passed at `terraform init`)
- `BASTION_SSH_KEY` (private key content for SSH tunnel)

### Variables
- `AWS_REGION`
- `EKS_CLUSTER`
- `ECR_REPO`

---

## 🧱 Repository structure (what’s inside)

* `.github/workflows/iac.yml`
  Full CI/CD automation: Terraform apply, tunnel setup, ingress install, Ansible tooling install, and destroy.

* `ansible/install_tools.yml`
  Installs platform tooling into Kubernetes using `kubernetes.core` modules + Helm charts.

* `terraform/eks-vpc-ecr.tf`
  VPC module + EKS module + ECR module definitions.

* `terraform/bastion.tf`
  Bastion security group + EC2 instance + rule allowing Bastion SG to reach EKS control plane on 443.

* `terraform/main.tf`
  Provider configuration (AWS + Kubernetes provider wired to EKS output).

* `terraform/variables.tf`
  Defaults for region, cluster name, VPC CIDRs, subnet CIDRs, tags, ECR repo name, Kubernetes version.

* `terraform/outputs.tf`
  Outputs for cluster details, kubeconfig command, ECR details, and Bastion public IP.

* `terraform/secret.tf`
  Creates an AWS Secrets Manager secret + version (currently placeholder values).

* `terraform/terraform.tf`
  Terraform required versions/providers + S3 backend configuration.

---

## 🧾 Purpose of every file :

### `.github/workflows/iac.yml`
**Purpose:** End-to-end automation pipeline for provisioning and cluster bootstrapping.

**What it contains:**
- Triggers:
  - Push to `main` with path filters (`terraform/**`, `ansible/**`)
  - Manual trigger for destroy
- Shared env values from GitHub **secrets/vars**
- **terraform job**
  - `terraform init` using the bucket value from secrets
  - `terraform validate`, `terraform plan`, `terraform apply`
  - `aws eks update-kubeconfig`
  - Creates an **SSH tunnel**: `localhost:8443 → <EKS endpoint>:443` through the Bastion
  - Updates kubeconfig cluster server to `https://localhost:8443`
  - Installs ingress-nginx
- **install_tools job**
  - Updates kubeconfig again
  - Builds SSH tunnel again
  - Installs Python deps, Helm, then runs `ansible-playbook ansible/install_tools.yml`
- **destroy job**
  - Updates kubeconfig + tunnel
  - Deletes ingress-nginx
  - Empties ECR images (cleanup)
  - Terraform destroy

---

### `ansible/install_tools.yml`
**Purpose:** Cluster “day-2” tooling installation (after EKS exists).

**What it contains:**
- Runs locally (`hosts: localhost`, `connection: local`) using kubectl config from the workflow
- Creates `argocd` namespace, installs Argo CD from upstream manifest
- Adds Helm repos, installs:
  - `prometheus-community/kube-prometheus-stack` into namespace `monitoring`
  - `kyverno/kyverno` into namespace `kyverno`
  - `eks/aws-for-fluent-bit` into namespace `logging`
- Fluent Bit values enable CloudWatch with region and logGroupName

---

### `terraform/eks-vpc-ecr.tf`
**Purpose:** The core AWS infrastructure modules.

**What it contains:**
- VPC module:
  - Public + private subnets across AZs
  - NAT gateway enabled (single NAT)
  - Subnet tags for load balancers (`kubernetes.io/role/elb` and internal ELB tag)
- EKS module:
  - EKS cluster name/version
  - Both public and private endpoint access enabled
  - Public endpoint access CIDR restricted to the Bastion IP (`3.226.241.153/32`)
  - Managed node group with scaling values (min 2 / desired 3 / max 4)
  - Adds CloudWatch policy to node IAM role
- ECR module:
  - Creates a private ECR repo with mutable tags and basic scanning

---

### `terraform/bastion.tf`
**Purpose:** Secure “jump host” for SSH tunneling into your cluster endpoint.

**What it contains:**
- Bastion SG:
  - Inbound SSH (22) open to 0.0.0.0/0
  - Full outbound allowed
- Bastion EC2 instance:
  - `t3.micro`
  - Public subnet
  - Public IP enabled
  - Key pair name specified
- Security group rule allowing the **Bastion SG** to access the **EKS cluster security group** on port 443  
  (this is what makes the tunnel-to-EKS endpoint possible)

---

### `terraform/main.tf`
**Purpose:** Provider wiring.

**What it contains:**
- Kubernetes provider points to:
  - `module.eks.cluster_endpoint`
  - CA certificate decoded from EKS output
- AWS provider uses region from variable

---

### `terraform/variables.tf`
**Purpose:** Central configuration defaults.

**What it contains:**
- Region default (`us-east-1`)
- Cluster name default (`githubactions-eks`)
- ECR repo default (`gitops-webapp`)
- Kubernetes version value (you set `1.32`)
- CIDR blocks for VPC + private + public subnets
- Default tags

---

### `terraform/outputs.tf`
**Purpose:** Exposes important values for automation and visibility.

**What it contains:**
- Cluster details: name, endpoint, platform version, status
- Convenience kubeconfig command string
- ECR repo URL + name
- Bastion public IP output (very useful to remove hardcoding in the workflow)

---

### `terraform/secret.tf`
**Purpose:** Creates a Secrets Manager secret for Cosign signing keys (currently placeholders).

**What it contains:**
- `aws_secretsmanager_secret`
- `aws_secretsmanager_secret_version` with JSON payload of placeholders

---

### `terraform/terraform.tf`
**Purpose:** Terraform runtime rules + providers + remote state backend.

**What it contains:**
- Required Terraform version
- S3 backend config (bucket/key/region)
- Required providers and version constraints (AWS, Kubernetes, Helm)

---

## References
- AWS EKS official documentation
- Terraform AWS provider documentation
- GitHub Actions documentation
- Ansible Kubernetes collections\

---

## ✅ Using the Project

This repository follows a **GitOps workflow**. All infrastructure provisioning and platform configuration are executed automatically through **GitHub Actions**.

No local execution of Terraform or Ansible is required.  
Changes are applied by committing and pushing updates to the repository.

```bash
git add .
git status
git commit -m "Update infrastructure configuration"
git push


