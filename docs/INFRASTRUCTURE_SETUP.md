# Movie Picture Pipeline: Infrastructure Setup & Deployment

## Complete Infrastructure Execution Guide

This document provides step-by-step instructions to provision AWS infrastructure (EKS cluster, ECR repositories) and configure Kubernetes for the CI/CD pipeline.

---

## Prerequisites

Ensure the following tools are installed on your local machine:

```bash
# Check installed versions
aws --version              # AWS CLI v2
terraform --version       # Terraform v1.0+
kubectl version --client  # Kubernetes CLI
kustomize version        # Kustomize v4.0+
docker --version         # Docker Desktop or Engine
jq --version             # JSON query tool (for parsing AWS responses)
```

### Installation Commands

```bash
# macOS (Homebrew)
brew install awscli terraform kubectl kustomize docker jq

# Linux (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y awscli terraform kubectl kustomize docker.io jq

# Windows (Chocolatey)
choco install awscli terraform kubectl kustomize docker jq
```

---

## Part 1: AWS Account & Credentials Setup

### Step 1.1: Configure AWS CLI

```bash
# Configure default AWS credentials
aws configure

# When prompted, enter:
# AWS Access Key ID: [your CI/CD user access key]
# AWS Secret Access Key: [your CI/CD user secret key]
# Default region: us-east-1
# Default output format: json
```

### Step 1.2: Verify AWS Credentials

```bash
# Test AWS CLI connection
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAI...",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/github-actions-cicd"
# }
```

### Step 1.3: Set AWS Environment Variables (Optional but Recommended)

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export AWS_REGION="us-east-1"
export EKS_CLUSTER_NAME="cluster"
export ECR_FRONTEND_REPO="movie-frontend"
export ECR_BACKEND_REPO="movie-backend"

# Save for reuse in subsequent sessions
echo "export AWS_ACCOUNT_ID=${AWS_ACCOUNT_ID}" >> ~/.bashrc
echo "export AWS_REGION=${AWS_REGION}" >> ~/.bashrc
```

---

## Part 2: Terraform Infrastructure Provisioning

### Step 2.1: Navigate to Terraform Directory

```bash
# Navigate to the Terraform manifests directory
cd setup/terraform

# List Terraform files to verify structure
ls -la
# Expected files:
# - main.tf (EKS cluster and ECR repos)
# - variables.tf
# - outputs.tf
# - terraform.tfvars (or create it)
```

### Step 2.2: Create terraform.tfvars (if not present)

```bash
# Create terraform.tfvars with your configuration
cat > terraform.tfvars << 'EOF'
aws_region             = "us-east-1"
cluster_name           = "cluster"
cluster_version        = "1.28"
node_group_desired     = 2
node_group_min         = 1
node_group_max         = 4
node_instance_type     = "t3.medium"

ecr_repositories = {
  "movie-frontend" = {
    image_tag_mutability = "MUTABLE"
    scan_on_push         = true
  }
  "movie-backend" = {
    image_tag_mutability = "MUTABLE"
    scan_on_push         = true
  }
}

tags = {
  Environment = "production"
  Project     = "movie-picture-pipeline"
  ManagedBy   = "terraform"
}
EOF

# Verify the file was created
cat terraform.tfvars
```

### Step 2.3: Initialize Terraform

```bash
# Initialize Terraform working directory
terraform init

# Expected output:
# Terraform has been successfully configured!
# You may now begin working with Terraform. Try running "terraform plan" to see
# any changes that would be made to your infrastructure.
```

### Step 2.4: Validate Terraform Configuration

```bash
# Validate syntax and logic
terraform validate

# Expected output:
# Success! The configuration is valid.

# Format Terraform files (best practice)
terraform fmt -recursive
```

### Step 2.5: Plan Infrastructure Changes

```bash
# Preview infrastructure changes (DO NOT apply yet)
terraform plan -out=tfplan

# Review the output carefully:
# - Number of resources to be created
# - EKS cluster configuration
# - ECR repository details
# - IAM roles and policies

# Example output snippet:
# Plan: 15 to add, 0 to change, 0 to destroy.
# Saved the plan to: tfplan
```

### Step 2.6: Apply Terraform Configuration

```bash
# Apply the planned infrastructure
# WARNING: This will provision AWS resources and incur costs
terraform apply tfplan

# Wait for completion (typically 10-15 minutes for EKS)
# You should see:
# Apply complete! Resources: 15 added, 0 changed, 0 destroyed.

# Capture outputs for later use
terraform output -json > terraform-outputs.json
```

### Step 2.7: Capture Critical Terraform Outputs

```bash
# Extract and save important values
export EKS_CLUSTER_ENDPOINT=$(terraform output -raw eks_cluster_endpoint)
export EKS_CLUSTER_CERT=$(terraform output -raw eks_cluster_certificate_authority_data)
export ECR_REGISTRY=$(terraform output -raw ecr_registry_url)

# Display outputs
echo "EKS Cluster Endpoint: ${EKS_CLUSTER_ENDPOINT}"
echo "ECR Registry: ${ECR_REGISTRY}"

# View all outputs
terraform output

# Save to file for reference
terraform output -json | tee terraform-outputs.json
```

---

## Part 3: Configure kubectl & Kubernetes Access

### Step 3.1: Update kubeconfig

```bash
# Configure kubectl to access EKS cluster
aws eks update-kubeconfig \
  --region ${AWS_REGION} \
  --name ${EKS_CLUSTER_NAME}

# Verify kubeconfig was updated
kubectl config current-context
# Expected output: arn:aws:eks:us-east-1:123456789012:cluster/cluster
```

### Step 3.2: Verify Kubernetes Connectivity

```bash
# Test cluster connectivity
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://...
# CoreDNS is running at https://...

# Check nodes are ready
kubectl get nodes -o wide

# Expected output:
# NAME                          STATUS   ROLES    AGE   VERSION
# ip-10-0-1-xxx.ec2.internal   Ready    <none>   5m    v1.28.x
# ip-10-0-2-xxx.ec2.internal   Ready    <none>   5m    v1.28.x
```

### Step 3.3: Verify Kubernetes Cluster Status

```bash
# Check all cluster components
kubectl get componentstatuses

# Verify system pods are running
kubectl get pods --all-namespaces

# Check available storage classes
kubectl get storageclass

# View resource quotas and limits
kubectl describe nodes | grep -A 5 "Allocated resources"
```

---

## Part 4: Setup RBAC & Service Accounts

### Step 4.1: Execute RBAC Initialization Script

```bash
# Navigate to setup directory
cd ../../setup

# Run the RBAC setup script
bash init.sh

# Expected output:
# ✅ GitHub Actions service account created
# ✅ RBAC role binding configured
# ✅ kubeconfig secret generated
```

### Step 4.2: Manual RBAC Setup (If init.sh is not available)

```bash
# Create namespace for applications
kubectl create namespace production

# Create service account for deployments
kubectl create serviceaccount github-actions -n production

# Create cluster role for deployment permissions
kubectl create role deployment-role \
  --verb=get,list,watch,create,update,patch,delete \
  --resource=deployments,services,configmaps,secrets \
  --namespace=production

# Bind role to service account
kubectl create rolebinding deployment-binding \
  --clusterrole=deployment-role \
  --serviceaccount=production:github-actions \
  --namespace=production

# Verify RBAC configuration
kubectl get serviceaccounts -n production
kubectl get rolebindings -n production
```

### Step 4.3: Create GitHub Actions Kubeconfig Secret

```bash
# Extract kubeconfig for GitHub Actions
CA_DATA=$(kubectl config view --flatten --minify --raw | base64 | tr -d '\n')

# Create secret in GitHub (via UI or API)
# Secret Name: KUBECONFIG_DATA
# Secret Value: [base64-encoded kubeconfig]

# Or extract specific components
export KUBECONFIG_USER=$(kubectl config view --raw -o jsonpath='{.users[0].name}')
export KUBECONFIG_CLUSTER=$(kubectl config view --raw -o jsonpath='{.clusters[0].name}')

echo "KUBECONFIG User: ${KUBECONFIG_USER}"
echo "KUBECONFIG Cluster: ${KUBECONFIG_CLUSTER}"
```

---

## Part 5: Create ECR Repositories (if not created by Terraform)

### Step 5.1: Verify ECR Repositories

```bash
# List all ECR repositories
aws ecr describe-repositories --region ${AWS_REGION}

# Expected output shows:
# - movie-frontend repository
# - movie-backend repository
```

### Step 5.2: Create ECR Repositories Manually (if needed)

```bash
# Create frontend ECR repository
aws ecr create-repository \
  --repository-name movie-frontend \
  --region ${AWS_REGION} \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability MUTABLE

# Create backend ECR repository
aws ecr create-repository \
  --repository-name movie-backend \
  --region ${AWS_REGION} \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability MUTABLE

# Verify repositories were created
aws ecr describe-repositories --region ${AWS_REGION} --output table
```

### Step 5.3: Set ECR Lifecycle Policies (Cleanup Old Images)

```bash
# Create lifecycle policy JSON
cat > lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images, expire others",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
EOF

# Apply lifecycle policy to repositories
aws ecr put-lifecycle-policy \
  --repository-name movie-frontend \
  --region ${AWS_REGION} \
  --lifecycle-policy-text file://lifecycle-policy.json

aws ecr put-lifecycle-policy \
  --repository-name movie-backend \
  --region ${AWS_REGION} \
  --lifecycle-policy-text file://lifecycle-policy.json
```

---

## Part 6: Deploy Kubernetes Manifests

### Step 6.1: Create Production Namespace & ConfigMaps

```bash
# Ensure production namespace exists
kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -

# Create ConfigMap for application configuration
kubectl create configmap app-config \
  --from-literal=FLASK_ENV=production \
  --from-literal=NODE_ENV=production \
  -n production \
  --dry-run=client -o yaml | kubectl apply -f -

# Verify ConfigMap
kubectl get configmaps -n production
```

### Step 6.2: Apply Kustomize Manifests

```bash
# Deploy frontend (from root directory)
cd starter/frontend/k8s
kustomize build . | kubectl apply -n production -f -

# Deploy backend (from root directory)
cd ../../backend/k8s
kustomize build . | kubectl apply -n production -f -

# Verify deployments
kubectl get deployments -n production
kubectl get pods -n production
```

### Step 6.3: Verify All Resources Created

```bash
# List all resources in production namespace
kubectl get all -n production

# Check specific resource types
kubectl get deployments,services,configmaps,secrets -n production -o wide

# Describe deployments to check status
kubectl describe deployment frontend -n production
kubectl describe deployment backend -n production
```

---

## Part 7: Configure Service Ingress (Optional)

### Step 7.1: Setup AWS Load Balancer Controller (for ingress)

```bash
# Add EKS chart repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${EKS_CLUSTER_NAME} \
  --set serviceAccount.create=true

# Verify installation
kubectl get deployment -n kube-system | grep aws-load-balancer-controller
```

### Step 7.2: Create Ingress Resource (Optional)

```bash
# Create ingress manifest
cat > ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: movie-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 5000
EOF

# Apply ingress
kubectl apply -f ingress.yaml

# Verify ingress created
kubectl get ingress -n production
```

---

## Part 8: Post-Deployment Verification

### Step 8.1: Check All Pods Are Running

```bash
# Verify pods are in Running state
kubectl get pods -n production

# Expected output shows pods with STATUS: Running
# If STATUS is Pending or CrashLoopBackOff, check logs:
kubectl logs -n production -l app=frontend --tail=50
kubectl logs -n production -l app=backend --tail=50
```

### Step 8.2: Verify Services & Load Balancers

```bash
# Check service endpoints
kubectl get svc -n production -o wide

# Get LoadBalancer external IP (may take 2-3 minutes)
kubectl get svc frontend -n production -w

# Once external IP is assigned:
FRONTEND_LB=$(kubectl get svc frontend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Frontend accessible at: http://${FRONTEND_LB}"
```

### Step 8.3: Test Application Endpoints

```bash
# Get backend service endpoint
BACKEND_LB=$(kubectl get svc backend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test backend health endpoint
curl -v http://${BACKEND_LB}:5000/health

# Test API endpoint
curl http://${BACKEND_LB}:5000/movies

# Test frontend (requires external IP)
curl http://${FRONTEND_LB}
```

---

## Part 9: Troubleshooting Infrastructure Issues

### Issue: EKS Cluster Not Accessible

```bash
# Verify kubeconfig is configured
kubectl config current-context

# Update kubeconfig if needed
aws eks update-kubeconfig --name cluster --region us-east-1

# Check if security group allows your IP
aws ec2 describe-security-groups \
  --filters Name=group-name,Values=eks-cluster-sg \
  --region us-east-1
```

### Issue: Pods Not Starting (Pending or CrashLoopBackOff)

```bash
# Check pod events
kubectl describe pod <pod-name> -n production

# View container logs
kubectl logs <pod-name> -n production

# Check resource requests vs. available resources
kubectl top nodes
kubectl top pods -n production

# Increase node capacity if needed (HPA)
kubectl autoscale deployment backend \
  --min=1 --max=5 \
  --cpu-percent=80 \
  -n production
```

### Issue: Services Not Getting External IP

```bash
# Verify service type is LoadBalancer
kubectl get svc -n production

# Check AWS Load Balancer service limits
aws elbv2 describe-load-balancers --region us-east-1

# Manually create load balancer if needed
aws elbv2 create-load-balancer \
  --name movie-frontend-lb \
  --subnets subnet-xxx subnet-yyy \
  --region us-east-1
```

---

## Checklist: Infrastructure Setup Complete

- [ ] AWS credentials configured (`aws configure` successful)
- [ ] Terraform initialized and validated
- [ ] EKS cluster provisioned and accessible
- [ ] ECR repositories created
- [ ] kubectl configured and verified
- [ ] RBAC configured for GitHub Actions
- [ ] Kubernetes manifests deployed
- [ ] All pods in Running state
- [ ] Services have external load balancers
- [ ] Backend health endpoint responding
- [ ] Frontend application accessible

---

## Next Steps

1. **Configure GitHub Secrets** (see `GITHUB_SECRETS_SETUP.md`)
2. **Push workflow files** to `.github/workflows/`
3. **Create a pull request** to trigger frontend-ci and backend-ci workflows
4. **Merge PR** to trigger frontend-cd and backend-cd deployments
5. **Verify deployments** in production namespace
6. **Monitor logs** for any issues
