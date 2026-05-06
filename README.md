# Craftista — Origami Store DevOps Project

A polyglot microservices application deployed on AWS EKS using Terraform, Helm, and monitored with Prometheus and Grafana. Based on the [School of Devops](https://github.com/craftista/craftista) Origami Store.

---

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │              AWS (ap-south-1)           │
                        │                                         │
                        │   ┌──────────────────────────────────┐  │
                        │   │         VPC (10.0.0.0/16)        │  │
                        │   │                                  │  │
                        │   │  Public Subnets                  │  │
                        │   │  ┌────────────────────────────┐  │  │
                        │   │  │  AWS Application Load      │  │  │
                        │   │  │  Balancer (ALB)            │  │  │
                        │   │  └────────────┬───────────────┘  │  │
                        │   │               │                  │  │
                        │   │  Private Subnets (EKS Nodes)     │  │
                        │   │  ┌────────────▼───────────────┐  │  │
                        │   │  │     EKS Cluster (os_app)   │  │  │
                        │   │  │                            │  │  │
                        │   │  │  default namespace         │  │  │
                        │   │  │  ├─ frontend (Node.js)     │  │  │
                        │   │  │  ├─ catalogue (Flask)      │  │  │
                        │   │  │  ├─ catalogue-db (Postgres)│  │  │
                        │   │  │  ├─ voting (Spring Boot)   │  │  │
                        │   │  │  └─ recco (Golang/Gin)     │  │  │
                        │   │  │                            │  │  │
                        │   │  │  monitoring namespace      │  │  │
                        │   │  │  ├─ Prometheus             │  │  │
                        │   │  │  └─ Grafana                │  │  │
                        │   │  └────────────────────────────┘  │  │
                        │   └──────────────────────────────────┘  │
                        └─────────────────────────────────────────┘
```

### Microservices

| Service | Language | Image | Port | Path |
|---|---|---|---|---|
| Frontend | Node.js / Express | `<docker_hub_repository>:<version>` | 3000 | `/` |
| Catalogue | Python / Flask | `<docker_hub_repository>:<version>` | 5000 | `/api/products` |
| Catalogue DB | PostgreSQL 15 | `postgres:15` | 5432 | internal |
| Voting | Java / Spring Boot | `<docker_hub_repository>:<version>` | 8080 | `/api/vote` |
| Recommendation | Golang / Gin | `<docker_hub_repository>:<version>` | 8080 | `/api/origami-of-the-day` |

### Infrastructure

| Component | Detail |
|---|---|
| EKS Version | 1.32 |
| Node Group | 2× t3.medium SPOT instances |
| Networking | VPC with 2 public + 2 private subnets, NAT Gateway |
| Storage | EBS gp3 via EBS CSI driver |
| Ingress | AWS Load Balancer Controller (ALB) |
| State Backend | S3 + DynamoDB lock table |
| Region | ap-south-1 |

---

## Prerequisites

Make sure the following tools are installed and configured before proceeding.

### Required Tools

| Tool | Version | Install |
|---|---|---|
| AWS CLI | v2 | [docs.aws.amazon.com](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) |
| Terraform | >= 1.3 | [terraform.io](https://developer.hashicorp.com/terraform/downloads) |
| kubectl | >= 1.28 | [kubernetes.io](https://kubernetes.io/docs/tasks/tools/) |
| Helm | >= 3.12 | [helm.sh](https://helm.sh/docs/intro/install/) |
| eksctl (optional) | latest | [eksctl.io](https://eksctl.io/) |

### AWS Configuration

```bash
aws configure
# AWS Access Key ID: <your-key>
# AWS Secret Access Key: <your-secret>
# AWS region: <region> 
# Default output format: json
```

Ensure your IAM user/role has permissions for:
- EKS (create/manage clusters)
- EC2 (VPC, subnets, security groups, instances)
- IAM (create roles and policies for IRSA)
- S3 and DynamoDB (Terraform state)
- Elastic Load Balancing

---

## 1. Infrastructure — Terraform

All infrastructure is defined in Terraform and lives in a **separate directory** from the Kubernetes manifests.

### Module Structure

```
terraform/
├── main.tf           # Wires all modules together
├── outputs.tf        # Exposes lbc_role_arn, vpc_id etc.
├── variables.tf
└── modules/
    ├── vpc/          # VPC, subnets, NAT Gateway
    ├── eks/          # EKS cluster, node group, OIDC provider
    ├── alb/          # AWS Load Balancer Controller IAM role
    └── backend/      # S3 bucket + DynamoDB for Terraform state
```

### Deploy Infrastructure

```bash
cd terraform

# Initialise — downloads providers and sets up remote state
terraform init

# Preview what will be created
terraform plan

# Apply — takes ~15 minutes for EKS to provision
terraform apply
```

### Update kubeconfig

After `terraform apply` completes, configure kubectl to talk to your new cluster:

```bash
aws eks update-kubeconfig --name <cluster_name> --region <region>

# Verify nodes are ready
kubectl get nodes
```

You should see 2 nodes in `Ready` status before proceeding.

### Key Terraform Outputs

```bash
terraform output lbc_role_arn   # Needed for Load Balancer Controller
terraform output vpc_id          # Needed for Load Balancer Controller
```

---

## 2. AWS Load Balancer Controller

The AWS Load Balancer Controller must be installed **before** deploying the app, as it provisions the ALB for the Ingress resource.

```bash
# Add the EKS Helm chart repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install the controller — replace <lbc_role_arn> and <vpc_id> with terraform outputs
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=os_app \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=<lbc_role_arn>" \
  --set region=<region> \
  --set vpcId=<vpc_id>

# Verify — should show 2 pods Running
kubectl get pods -n kube-system | grep load-balancer
```

---

## 3. Application — Helm

The application is packaged as an umbrella Helm chart with a subchart per microservice.

### Chart Structure

```
helm/
├── Chart.yaml              # Umbrella chart — declares subchart dependencies
├── values.yaml             # Top-level values — override anything here
├── .helmignore
├── templates/
│   ├── ingress.yaml        # ALB Ingress routing all services
│   └── storageclass.yaml   # gp3 EBS StorageClass
└── charts/
    ├── frontend/            # Deployment + Service + ConfigMap
    ├── catalogue/           # Deployment + Service + StatefulSet + Secret + Job
    ├── voting/              # Deployment + Service
    └── recommendation/      # Deployment + Service
```

### Configuration

Key values in `helm/values.yaml`:

```yaml
catalogue:
  db:
    password: "catalogue"   # Override this — never commit the real password

ingress:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: craftista  # Groups app + monitoring ALBs
```

### Deploy the Application

```bash
cd helm

# Install — pass DB password via --set, never hardcode in values.yaml
helm install craftista . --set catalogue.db.password=<your-password>

# Watch pods come up
kubectl get pods -w
```

All pods should reach `1/1 Running` within 3-5 minutes. The catalogue-db pod initialises an EBS volume on first run which takes slightly longer.

### Verify

```bash
# Get the ALB address (takes 2-3 minutes to provision)
kubectl get ingress craftista-ingress

# Test the catalogue API
curl http://<ALB-ADDRESS>/api/products

# Test the voting API
curl http://<ALB-ADDRESS>/api/vote

# Test the recommendation API
curl http://<ALB-ADDRESS>/api/origami-of-the-day
```

### Upgrading

When you make changes to values or templates:

```bash
helm upgrade craftista . --set catalogue.db.password=<your-password>
```

Helm performs a rolling update — only changed resources are restarted.

### Troubleshooting

| Problem | Fix |
|---|---|
| `ImagePullBackOff` | Check image repo/tag in `values.yaml` — verify image exists on Docker Hub |
| PVC stuck `Pending` | Ensure gp3 StorageClass exists: `kubectl get sc` |
| ALB address not populating | Verify LBC is running: `kubectl get pods -n kube-system \| grep load-balancer` |
| `Request entity too large` | Helm chart is picking up non-chart files — ensure you run `helm` from inside the `helm/` directory |

---

## 4. Monitoring — Prometheus & Grafana

Prometheus and Grafana are installed via the `kube-prometheus-stack` Helm chart into a dedicated `monitoring` namespace.

### Install

```bash
# Add the Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.service.type=ClusterIP

# Verify all pods are running (~2 minutes)
kubectl get pods -n monitoring
```

### Expose Grafana via ALB

Grafana is exposed on the same ALB as the application using ALB Ingress Groups.

```bash
# Apply the Grafana ingress into the monitoring namespace
kubectl apply -f helm/monitoring/grafana-ingress.yaml

# Set group order so /grafana path takes priority over the app's catch-all /
kubectl annotate ingress grafana-ingress -n monitoring \
  "alb.ingress.kubernetes.io/group.order=1" --overwrite

kubectl annotate ingress craftista-ingress \
  "alb.ingress.kubernetes.io/group.order=10" --overwrite

# Configure Grafana to serve from /grafana subpath
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --reuse-values \
  --set "grafana.grafana\.ini.server.root_url=http://<ALB-ADDRESS>/grafana" \
  --set "grafana.grafana\.ini.server.serve_from_sub_path=true"
```

### Access Grafana

```
URL:      http://<ALB-ADDRESS>/grafana
Username: admin
Password: kubectl get secret -n monitoring prometheus-grafana \
            -o jsonpath="{.data.admin-password}" | base64 -d
```

### Import the Craftista Dashboard

1. In Grafana go to **Dashboards → Import**
2. Upload `helm/monitoring/craftista-dashboard.json`
3. Select **Prometheus** as the datasource
4. Click **Import**

The dashboard includes:
- Node CPU & Memory usage
- Pod CPU & Memory per microservice
- HTTP request rates and latency
- PostgreSQL catalogue DB metrics

---

## 5. Destroy

Run this at the end of every session to avoid unnecessary AWS costs.

```bash
# 1. Remove the Grafana ingress
kubectl delete ingress grafana-ingress -n monitoring

# 2. Uninstall Helm releases
helm uninstall craftista
helm uninstall prometheus -n monitoring

# 3. Delete the PostgreSQL PVC (EBS volume)
kubectl delete pvc postgres-storage-catalogue-db-0

# 4. Wait for ALB to deregister targets (~60 seconds)
sleep 60

# 5. Destroy all AWS infrastructure
cd terraform
terraform destroy
```

> **Important:** Always delete the Helm releases and PVC **before** running `terraform destroy`. If the ALB or EBS volume still exists when Terraform tries to delete the VPC, the destroy will fail with dependency errors.

---

## Key Design Decisions

### Why an Umbrella Helm Chart?
A single `helm install` deploys all four microservices, the database, storage class, and ingress in one operation. Subchart values can be overridden from the top-level `values.yaml` making environment-specific configuration straightforward.

### Why ALB Ingress Groups?
The application and Grafana live in different Kubernetes namespaces (`default` and `monitoring`). ALB Ingress Groups allow both Ingress resources to share a single ALB, avoiding the cost and complexity of provisioning a second load balancer.

### Why gp3 StorageClass with WaitForFirstConsumer?
EBS volumes are AZ-specific. `WaitForFirstConsumer` binding mode ensures the PVC is provisioned in the same AZ as the pod that claims it, preventing scheduling failures due to AZ affinity conflicts.

### PGDATA Subdirectory
PostgreSQL crashes on startup if its data directory contains files it doesn't recognise (like the `lost+found` directory EBS volumes initialise with). Setting `PGDATA=/var/lib/postgresql/data/pgdata` points Postgres at a clean subdirectory, avoiding this issue.

---

## Repository Structure

```
craftista/
├── helm/                    # Helm umbrella chart
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── .helmignore
│   ├── templates/
│   │   ├── ingress.yaml
│   │   └── storageclass.yaml
│   ├── monitoring/
│   │   ├── grafana-ingress.yaml
│   │   └── craftista-dashboard.json
│   └── charts/
│       ├── frontend/
│       ├── catalogue/
│       ├── voting/
│       └── recommendation/
├── k8s/                     # Raw Kubernetes manifests (reference)
├── terraform/               # AWS infrastructure
├── docs/
└── README.md
```

---

## Credits

- Original application: [craftista/craftista](https://github.com/craftista/craftista) by School of Devops
- License: Apache 2.0
