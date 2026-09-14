# Movie Picture Pipeline: AWS Teardown Protocol

Complete procedures for safely removing AWS resources to prevent account charge depletion after verification or development completion.

---

## ⚠️ WARNING

Executing these commands will **permanently delete** AWS resources including:
- EKS cluster and associated infrastructure
- ECR repositories and all container images
- Load balancers and related networking
- IAM roles and policies
- All related data

**This action is irreversible.** Ensure you have backups or exports of any data you need to retain.

---

## Pre-Teardown Checklist

Before executing teardown commands:

- [ ] Confirm no production workloads are running
- [ ] Export any important data from the cluster
- [ ] Backup ECR images if needed
- [ ] Disable any monitoring/logging (CloudWatch)
- [ ] Notify team members of teardown
- [ ] Verify you have AWS IAM permissions to delete resources
- [ ] Read through the entire teardown procedure

---

## Part 1: Backup Critical Data (Optional but Recommended)

### Step 1.1: Export Kubernetes Manifests

```bash
# Backup current deployments
mkdir -p k8s-backups

# Export all resources from production namespace
kubectl get all -n production -o yaml > k8s-backups/all-resources.yaml

# Export specific deployments
kubectl get deployment -n production -o yaml > k8s-backups/deployments.yaml
kubectl get svc -n production -o yaml > k8s-backups/services.yaml
kubectl get configmaps -n production -o yaml > k8s-backups/configmaps.yaml
kubectl get secrets -n production -o yaml > k8s-backups/secrets.yaml

# Export events and logs
kubectl get events -n production -o yaml > k8s-backups/events.yaml

# Compress backups
tar -czf k8s-backups-$(date +%s).tar.gz k8s-backups/

echo "✅ Kubernetes manifests backed up to k8s-backups-*.tar.gz"
```

### Step 1.2: Export ECR Images (Optional)

```bash
# List all ECR images to a file
aws ecr describe-images \
  --repository-name movie-frontend \
  --region us-east-1 \
  --output json > ecr-frontend-images.json

aws ecr describe-images \
  --repository-name movie-backend \
  --region us-east-1 \
  --output json > ecr-backend-images.json

# If you need to preserve specific images, pull them locally:
FRONTEND_IMAGE=$(aws ecr describe-images \
  --repository-name movie-frontend \
  --region us-east-1 \
  --query 'imageDetails[0].imageTags[0]' \
  --output text)

docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/movie-frontend:${FRONTEND_IMAGE}

# Tag for local registry
docker tag 123456789012.dkr.ecr.us-east-1.amazonaws.com/movie-frontend:${FRONTEND_IMAGE} \
  movie-frontend:${FRONTEND_IMAGE}

# Save locally
docker save movie-frontend:${FRONTEND_IMAGE} -o movie-frontend-${FRONTEND_IMAGE}.tar

echo "✅ Images exported to local Docker"
```

### Step 1.3: Export Terraform State

```bash
# Backup Terraform state file
cd setup/terraform

# Copy state files
mkdir -p terraform-backups
cp terraform.tfstate terraform-backups/terraform.tfstate.backup.$(date +%s)
cp terraform.tfstate.backup terraform-backups/ 2>/dev/null || true

# View current state
terraform show > terraform-backups/terraform-state.$(date +%s).txt

echo "✅ Terraform state backed up"
```

---

## Part 2: Delete Kubernetes Resources

### Step 2.1: Delete Application Deployments

```bash
# Delete frontend deployment
kubectl delete deployment frontend -n production

# Delete backend deployment
kubectl delete deployment backend -n production

# Delete services
kubectl delete service frontend -n production
kubectl delete service backend -n production

# Verify deployments are deleted
kubectl get deployments -n production
# Expected: No resources found

# Wait for load balancers to be removed (may take 1-2 minutes)
sleep 120

echo "✅ Kubernetes deployments deleted"
```

### Step 2.2: Delete ConfigMaps & Secrets

```bash
# Delete ConfigMaps
kubectl delete configmap app-config -n production
kubectl delete configmaps --all -n production

# Delete Secrets
kubectl delete secrets --all -n production

# Verify deletion
kubectl get configmaps,secrets -n production
# Expected: No resources found

echo "✅ ConfigMaps and Secrets deleted"
```

### Step 2.3: Delete Kubernetes Namespace (Optional)

```bash
# List all namespaces
kubectl get namespaces

# Delete production namespace (this deletes everything in it)
kubectl delete namespace production

# Verify namespace is deleted
kubectl get namespaces | grep production
# Expected: namespace not found

# Wait for namespace deletion to complete
sleep 30

echo "✅ Production namespace deleted"
```

---

## Part 3: Clean Up AWS ECR Repositories

### Step 3.1: Delete ECR Images

```bash
# Set environment variables
export AWS_REGION="us-east-1"
export ECR_FRONTEND_REPO="movie-frontend"
export ECR_BACKEND_REPO="movie-backend"

# List all images in frontend repository
aws ecr describe-images \
  --repository-name ${ECR_FRONTEND_REPO} \
  --region ${AWS_REGION}

# Delete all images in frontend repository
aws ecr batch-delete-image \
  --repository-name ${ECR_FRONTEND_REPO} \
  --region ${AWS_REGION} \
  --image-ids "$(aws ecr describe-images \
    --repository-name ${ECR_FRONTEND_REPO} \
    --region ${AWS_REGION} \
    --query 'imageDetails[*].[{imageTag:imageTags[0],imageDigest:imageDigest}]' \
    --output json | jq -r '.[] | "\(.imageTag//\"NONE\")" | "imageTag=\(.)"')"

# Alternative: Delete all images by pushing empty set
aws ecr batch-delete-image \
  --repository-name ${ECR_FRONTEND_REPO} \
  --region ${AWS_REGION} \
  --image-ids "$(aws ecr list-images \
    --repository-name ${ECR_FRONTEND_REPO} \
    --region ${AWS_REGION} \
    --query 'imageIds' \
    --output json)" || true

# Repeat for backend repository
aws ecr batch-delete-image \
  --repository-name ${ECR_BACKEND_REPO} \
  --region ${AWS_REGION} \
  --image-ids "$(aws ecr list-images \
    --repository-name ${ECR_BACKEND_REPO} \
    --region ${AWS_REGION} \
    --query 'imageIds' \
    --output json)" || true

# Verify repositories are empty
aws ecr describe-images \
  --repository-name ${ECR_FRONTEND_REPO} \
  --region ${AWS_REGION} | jq '.imageDetails | length'

echo "✅ ECR images deleted"
```

### Step 3.2: Delete ECR Repositories

```bash
# Delete frontend repository
aws ecr delete-repository \
  --repository-name movie-frontend \
  --region us-east-1 \
  --force

# Delete backend repository
aws ecr delete-repository \
  --repository-name movie-backend \
  --region us-east-1 \
  --force

# Verify repositories are deleted
aws ecr describe-repositories --region us-east-1
# Expected: No repositories found

echo "✅ ECR repositories deleted"
```

---

## Part 4: Destroy Terraform Infrastructure

### Step 4.1: Plan Terraform Destruction

```bash
# Navigate to Terraform directory
cd setup/terraform

# Plan the destruction (review what will be deleted)
terraform plan -destroy -out=destroy-plan

# Review the output carefully - should show:
# - EKS cluster deletion
# - EC2 node groups deletion
# - IAM roles deletion
# - ECR repositories (if managed by Terraform)
# - VPC and subnets deletion

echo "Review the plan above before proceeding to Step 4.2"
```

### Step 4.2: Apply Terraform Destruction

```bash
# Apply the destruction plan
# ⚠️ This is the point of no return
terraform destroy -auto-approve -lock=false

# Expected output:
# Destroy complete! Resources: 15 destroyed.

# Verify state file is updated
terraform show
# Expected: No resources shown

echo "✅ Terraform infrastructure destroyed"
```

### Step 4.3: Verify AWS Resources Deleted

```bash
# Verify EKS cluster is deleted
aws eks describe-cluster --name cluster --region us-east-1 2>&1 | grep -i "not found\|error"

# Verify EC2 instances are terminated
aws ec2 describe-instances \
  --filters "Name=tag:aws:eks:cluster-name,Values=cluster" \
  --region us-east-1 | jq '.Reservations | length'
# Expected: 0

# Verify security groups are removed
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=eks-cluster-sg" \
  --region us-east-1 | jq '.SecurityGroups | length'
# Expected: 0

# Verify VPC is removed (if created by Terraform)
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=*movie*" \
  --region us-east-1 | jq '.Vpcs | length'

# Verify load balancers are removed
aws elbv2 describe-load-balancers --region us-east-1 | jq '.LoadBalancers | length'

echo "✅ AWS resources cleanup verified"
```

---

## Part 5: Clean Up IAM (Optional)

### Step 5.1: Delete IAM User for CI/CD

```bash
# Set IAM user name
IAM_USER="github-actions-cicd"

# List all access keys for the user
aws iam list-access-keys --user-name ${IAM_USER}

# Delete access keys (one at a time)
aws iam delete-access-key \
  --user-name ${IAM_USER} \
  --access-key-id <ACCESS_KEY_ID>

# Delete all inline policies
aws iam list-user-policies --user-name ${IAM_USER}

# Delete each inline policy
aws iam delete-user-policy \
  --user-name ${IAM_USER} \
  --policy-name GitHubActionsCICDPolicy

# Delete the user
aws iam delete-user --user-name ${IAM_USER}

# Verify user is deleted
aws iam get-user --user-name ${IAM_USER} 2>&1 | grep -i "not found\|error"

echo "✅ IAM user deleted"
```

### Step 5.2: Delete IAM Roles (if created)

```bash
# List all roles
aws iam list-roles --query 'Roles[?contains(RoleName, `movie`)].RoleName' --output text

# Delete role policies
aws iam list-role-policies --role-name eks-service-role | \
  jq -r '.PolicyNames[]' | \
  xargs -I {} aws iam delete-role-policy --role-name eks-service-role --policy-name {}

# Delete the role
aws iam delete-role --role-name eks-service-role

# Repeat for other roles as needed
aws iam delete-role --role-name eks-node-role

echo "✅ IAM roles deleted"
```

---

## Part 6: Final Cleanup & Verification

### Step 6.1: Clear Local Kubernetes Configuration

```bash
# Remove EKS cluster from kubeconfig
kubectl config delete-context arn:aws:eks:us-east-1:123456789012:cluster/cluster

# Remove cluster from kubeconfig
kubectl config delete-cluster arn:aws:eks:us-east-1:123456789012:cluster/cluster

# Remove user credentials
kubectl config delete-user arn:aws:eks:us-east-1:123456789012:cluster/cluster

# Verify kubeconfig is cleaned
kubectl config view

echo "✅ Local kubeconfig cleaned"
```

### Step 6.2: Remove GitHub Actions Secrets (Optional)

```bash
# Using GitHub CLI
gh secret delete AWS_ACCESS_KEY_ID
gh secret delete AWS_SECRET_ACCESS_KEY
gh secret delete AWS_REGION
gh secret delete REACT_APP_MOVIE_API_URL

# Or manually via GitHub UI:
# Settings → Secrets and variables → Actions → Delete each secret

echo "✅ GitHub secrets cleaned (optional)"
```

### Step 6.3: Remove Workflow Files (Optional)

```bash
# Backup workflows (optional)
mkdir -p workflows-backup
cp .github/workflows/*.yaml workflows-backup/

# Delete workflow files
rm .github/workflows/frontend-ci.yaml
rm .github/workflows/frontend-cd.yaml
rm .github/workflows/backend-ci.yaml
rm .github/workflows/backend-cd.yaml

# Commit and push changes
git add .github/workflows/
git commit -m "chore: remove CI/CD workflows for project cleanup"
git push origin main

echo "✅ Workflow files removed"
```

### Step 6.4: Clean Up Local Terraform State

```bash
# Navigate to Terraform directory
cd setup/terraform

# Backup state directory
mkdir -p terraform-backups-final
cp -r .terraform terraform-backups-final/

# Remove Terraform state and lock files (use carefully)
rm -f .terraform.lock.hcl
rm -f terraform.tfstate
rm -f terraform.tfstate.backup

# Remove .terraform directory
rm -rf .terraform

# Verify clean state
ls -la
# Should NOT show .terraform or *.tfstate files

echo "✅ Local Terraform state cleaned"
```

---

## Part 7: Final AWS Bill Verification

### Step 7.1: Check for Remaining AWS Resources

```bash
# List all EC2 instances
aws ec2 describe-instances --region us-east-1 | jq '.Reservations[].Instances[] | {InstanceId, State}'

# List all load balancers
aws elbv2 describe-load-balancers --region us-east-1 | jq '.LoadBalancers[] | {LoadBalancerName, State}'

# List all NAT gateways (these have hourly charges)
aws ec2 describe-nat-gateways --region us-east-1 | jq '.NatGateways[] | {NatGatewayId, State}'

# List all elastic IPs
aws ec2 describe-addresses --region us-east-1 | jq '.Addresses[] | {PublicIp, AssociationId}'

# List all RDS instances
aws rds describe-db-instances --region us-east-1 | jq '.DBInstances[] | {DBInstanceIdentifier, DBInstanceStatus}'

# List all S3 buckets (these have storage charges)
aws s3 ls | grep "movie\|pipeline"

# Check current AWS spending (requires AWS Cost Explorer access)
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=$(date +%Y-%m-%d) \
  --granularity DAILY \
  --metrics BlendedCost \
  --region us-east-1 | jq '.ResultsByTime[] | {TimePeriod, Total}'
```

### Step 7.2: Set Up AWS Budget Alerts (Recommended)

```bash
# Create a budget alert to notify if costs exceed threshold
aws budgets create-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget file://budget-alert.json \
  --notifications-with-subscribers file://notifications.json

# Where budget-alert.json contains:
cat > budget-alert.json << 'EOF'
{
  "BudgetName": "MonthlySpendAlert",
  "BudgetLimit": {
    "Amount": "10.00",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
EOF

echo "✅ AWS billing verification complete"
```

---

## Troubleshooting Teardown Issues

### Issue: "Cannot delete EKS cluster - resources still running"

```bash
# Check for load balancers still attached
aws elbv2 describe-load-balancers --region us-east-1

# Delete orphaned load balancers manually
aws elbv2 delete-load-balancer --load-balancer-arn <ARN> --region us-east-1

# Check for volumes still attached
aws ec2 describe-volumes --region us-east-1 --query 'Volumes[?State==`available`]'

# Delete orphaned volumes
aws ec2 delete-volume --volume-id <VOLUME_ID> --region us-east-1

# Then retry Terraform destroy
terraform destroy -auto-approve
```

### Issue: "Terraform state lock - operation already in progress"

```bash
# Force unlock (use with caution)
terraform force-unlock <LOCK_ID>

# Or manually remove lock file
rm -f .terraform.tfstate.lock.info

# Retry operation
terraform destroy -auto-approve
```

### Issue: "IAM role still has attached policies"

```bash
# List all attached policies
aws iam list-attached-role-policies --role-name eks-service-role

# Detach policies
aws iam detach-role-policy \
  --role-name eks-service-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSServiceRolePolicy

# List inline policies
aws iam list-role-policies --role-name eks-service-role

# Delete inline policies
aws iam delete-role-policy \
  --role-name eks-service-role \
  --policy-name <POLICY_NAME>

# Then retry deletion
aws iam delete-role --role-name eks-service-role
```

### Issue: "ECR repository not empty - cannot delete"

```bash
# List all images
aws ecr list-images \
  --repository-name movie-frontend \
  --region us-east-1

# Delete all images forcefully
aws ecr batch-delete-image \
  --repository-name movie-frontend \
  --region us-east-1 \
  --image-ids \
    imageDigest=$(aws ecr list-images \
      --repository-name movie-frontend \
      --region us-east-1 \
      --query 'imageIds[].imageDigest' \
      --output text | tr '\t' ' ' | sed 's/ / imageDigest=/g')

# Then delete repository
aws ecr delete-repository \
  --repository-name movie-frontend \
  --region us-east-1 \
  --force
```

---

## Complete Teardown Checklist

- [ ] Exported and backed up all Kubernetes manifests
- [ ] Backed up ECR images (if needed)
- [ ] Backed up Terraform state files
- [ ] Deleted all Kubernetes deployments
- [ ] Deleted all Kubernetes services
- [ ] Deleted production namespace
- [ ] Deleted all ECR images
- [ ] Deleted ECR repositories
- [ ] Executed `terraform destroy` successfully
- [ ] Verified EKS cluster is gone
- [ ] Verified EC2 instances are terminated
- [ ] Verified load balancers are removed
- [ ] Deleted CI/CD IAM user
- [ ] Cleaned local kubeconfig
- [ ] Deleted GitHub Actions secrets (optional)
- [ ] Removed workflow files (optional)
- [ ] Verified no remaining AWS resources
- [ ] Confirmed all charges have stopped

---

## Final Verification Command

```bash
# Run this command to confirm all resources are deleted
cat << 'SCRIPT' > final-verification.sh
#!/bin/bash

echo "🔍 Final AWS Resource Cleanup Verification"
echo "=========================================="

AWS_REGION="us-east-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Check EKS
echo "EKS Clusters:"
aws eks list-clusters --region $AWS_REGION | jq '.clusters | length'

# Check EC2
echo "EC2 Instances:"
aws ec2 describe-instances --region $AWS_REGION --query 'Reservations[].Instances[] | length(@)' --output text

# Check Load Balancers
echo "Load Balancers:"
aws elbv2 describe-load-balancers --region $AWS_REGION | jq '.LoadBalancers | length'

# Check ECR
echo "ECR Repositories:"
aws ecr describe-repositories --region $AWS_REGION | jq '.repositories | length'

# Check NAT Gateways
echo "NAT Gateways:"
aws ec2 describe-nat-gateways --region $AWS_REGION --query 'NatGateways[] | length(@)' --output text

# Check RDS
echo "RDS Instances:"
aws rds describe-db-instances --region $AWS_REGION | jq '.DBInstances | length'

echo ""
echo "✅ Verification complete!"
echo "All counts above should be 0"
SCRIPT

chmod +x final-verification.sh
./final-verification.sh
```

---

## Post-Teardown Steps

1. **Confirm AWS Billing**: Wait 24-48 hours and verify no additional charges appear
2. **Cancel AWS Resources**: If accounts were created solely for this project, consider deactivating them
3. **Archive Documentation**: Keep a copy of this guide and all backup files for reference
4. **Notify Team**: Inform team members that resources have been cleaned up
5. **Update Repository**: Remove references to AWS resources from documentation
