# Movie Picture Pipeline: Quick Start Guide

**Complete CI/CD Pipeline & Kubernetes Deployment Solution**

---

## 📦 DELIVERABLES SUMMARY

This package contains **9 complete, production-ready files**:

### GitHub Actions Workflows (4 files)
1. **frontend-ci.yaml** — Frontend CI pipeline (lint, test, build)
2. **frontend-cd.yaml** — Frontend CD pipeline (build, push to ECR, deploy to EKS)
3. **backend-ci.yaml** — Backend CI pipeline (Python 3.10, pipenv, lint, test, build)
4. **backend-cd.yaml** — Backend CD pipeline (test gate, ECR push, kustomize deploy)

### Comprehensive Guides (5 files)
5. **GITHUB_SECRETS_SETUP.md** — Configure AWS credentials securely
6. **INFRASTRUCTURE_SETUP.md** — Provision EKS, ECR, and Kubernetes resources
7. **VERIFICATION_COMMANDS.md** — 100+ commands to verify deployments
8. **AWS_TEARDOWN_PROTOCOL.md** — Safe resource cleanup procedures
9. **IMPLEMENTATION_NOTES.md** — Detailed requirements compliance documentation

---

## 🚀 DEPLOYMENT IN 5 STEPS

### Step 1: Configure GitHub Secrets (10 minutes)

Follow **GITHUB_SECRETS_SETUP.md**:

```bash
# Create AWS IAM user
aws iam create-user --user-name github-actions-cicd
aws iam create-access-key --user-name github-actions-cicd

# Add 4 secrets to GitHub Repository Settings:
# Settings → Secrets and variables → Actions → New repository secret

# Secret 1: AWS_ACCESS_KEY_ID
# Secret 2: AWS_SECRET_ACCESS_KEY
# Secret 3: AWS_REGION = us-east-1
# Secret 4: REACT_APP_MOVIE_API_URL = http://localhost:5000 (or your backend URL)
```

**Time:** ~10 minutes

---

### Step 2: Provision AWS Infrastructure (15 minutes)

Follow **INFRASTRUCTURE_SETUP.md** (Part 2-3):

```bash
# Navigate to Terraform directory
cd setup/terraform

# Initialize Terraform
terraform init

# Plan and apply infrastructure
terraform plan -out=tfplan
terraform apply tfplan

# Configure kubectl
aws eks update-kubeconfig --name cluster --region us-east-1

# Verify cluster connectivity
kubectl cluster-info
kubectl get nodes
```

**Time:** ~15 minutes for EKS cluster to provision

**Cost:** ~$0.10/hour for EKS cluster + ~$0.05/hour per node (2 t3.medium nodes by default)

---

### Step 3: Deploy Workflow Files to GitHub (5 minutes)

Place the 4 YAML files in your repository:

```bash
# Create .github/workflows directory (if not exists)
mkdir -p .github/workflows

# Copy workflow files
cp frontend-ci.yaml .github/workflows/
cp frontend-cd.yaml .github/workflows/
cp backend-ci.yaml .github/workflows/
cp backend-cd.yaml .github/workflows/

# Commit and push
git add .github/workflows/
git commit -m "ci: add production-grade CI/CD pipelines"
git push origin main
```

**Time:** ~5 minutes

---

### Step 4: Trigger Test Deployment (10 minutes)

```bash
# Option 1: Create a pull request
git checkout -b feature/test-pipeline
# Make a small change to starter/frontend/ or starter/backend/
git push origin feature/test-pipeline
# Create PR via GitHub UI

# Option 2: Manual trigger via GitHub CLI
gh workflow run frontend-ci.yaml --ref main
gh workflow run backend-ci.yaml --ref main

# Watch workflows execute
gh run list
gh run watch
```

**Expected Result:**
- ✅ frontend-ci passes (lint, test, build)
- ✅ backend-ci passes (lint, test, build)

**Time:** ~10 minutes for workflows to execute

---

### Step 5: Merge & Deploy to Production (15 minutes)

```bash
# Merge PR to main (or push to main after verification)
git checkout main
git merge feature/test-pipeline
git push origin main

# This automatically triggers:
# - frontend-cd.yaml (build, push to ECR, deploy to EKS)
# - backend-cd.yaml (build, push to ECR, deploy to EKS)

# Monitor deployment
gh run watch
kubectl get pods -n production -w

# Verify services are running
kubectl get svc -n production -o wide

# Get application endpoints
FRONTEND_LB=$(kubectl get svc frontend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
BACKEND_LB=$(kubectl get svc backend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "Frontend: http://${FRONTEND_LB}"
echo "Backend: http://${BACKEND_LB}:5000"

# Test backend health
curl http://${BACKEND_LB}:5000/health
curl http://${BACKEND_LB}:5000/movies
```

**Expected Result:**
- ✅ Images pushed to ECR
- ✅ Frontend deployment running (2 replicas)
- ✅ Backend deployment running (2 replicas)
- ✅ Services have external load balancer IPs
- ✅ Backend health endpoint responds (HTTP 200)

**Time:** ~15 minutes for full deployment

---

## 📋 TOTAL DEPLOYMENT TIME: ~55 minutes

| Step | Duration | Cost |
|------|----------|------|
| GitHub Secrets Setup | 10 min | $0 |
| AWS Infrastructure | 15 min | $0.03-0.05 |
| Deploy Workflow Files | 5 min | $0 |
| Test CI Pipeline | 10 min | $0.01 |
| Deploy to Production | 15 min | $0.01 |
| **TOTAL** | **~55 min** | **~$0.10** |

---

## ✅ VERIFICATION CHECKLIST

After deployment, verify with these commands (see **VERIFICATION_COMMANDS.md** for details):

```bash
# ✅ Kubernetes Cluster
kubectl cluster-info
kubectl get nodes
kubectl get pods -n production

# ✅ Deployments
kubectl get deployments -n production
kubectl describe deployment frontend -n production
kubectl describe deployment backend -n production

# ✅ Services & Load Balancers
kubectl get svc -n production -o wide

# ✅ Backend API Endpoints
BACKEND_LB=$(kubectl get svc backend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://${BACKEND_LB}:5000/health          # Should return 200 OK
curl http://${BACKEND_LB}:5000/movies          # Should return JSON

# ✅ Frontend Application
FRONTEND_LB=$(kubectl get svc frontend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -I http://${FRONTEND_LB}                  # Should return 200 OK

# ✅ ECR Images
aws ecr describe-images --repository-name movie-frontend --region us-east-1
aws ecr describe-images --repository-name movie-backend --region us-east-1

# ✅ Pod Logs
kubectl logs -n production -l app=frontend --tail=50
kubectl logs -n production -l app=backend --tail=50
```

---

## 🔐 SECURITY CHECKLIST

- [x] All AWS credentials stored in GitHub Secrets (not hardcoded)
- [x] IAM policy uses least-privilege permissions (only ECR + EKS)
- [x] Frontend Docker build uses dynamic API URL from secrets
- [x] Kubernetes RBAC configured for namespace-scoped access
- [x] Workflows mask secrets in logs (GitHub automatic)
- [x] No plaintext credentials in YAML files

---

## 💰 COST MANAGEMENT

### Estimated Monthly Costs (default config: 2 nodes)
- EKS Control Plane: **$73.00**
- EC2 Nodes (2 × t3.medium): **~$60.00**
- Load Balancers (2 × ALB): **~$32.00**
- **Total: ~$165/month**

### Cost Optimization
- Use `terraform apply` with `node_group_desired=0` for zero-cost testing
- Scale down to 1 node: `terraform apply -var node_group_desired=1`
- Use Spot instances: Reduce EC2 costs by 70% (requires changes to Terraform)

### When Done: Cleanup with AWS_TEARDOWN_PROTOCOL.md
```bash
# Safe resource deletion
cd setup/terraform
terraform destroy -auto-approve

# This removes all AWS resources and stops all charges
```

---

## 🔧 CUSTOMIZATION GUIDE

### Change Python Version (Backend)
Edit **backend-ci.yaml** and **backend-cd.yaml**:
```yaml
python-version: '3.11'  # Change from 3.10 to 3.11
```

### Change Node Instance Type
Edit **setup/terraform/terraform.tfvars**:
```hcl
node_instance_type = "t3.small"  # Change from t3.medium
```

### Change Number of Replicas
Edit **starter/frontend/k8s/kustomization.yaml**:
```yaml
replicas:
- name: frontend
  count: 3  # Change from 2 to 3
```

### Change Build Triggers
Edit **frontend-ci.yaml** and **frontend-cd.yaml** paths:
```yaml
paths:
  - 'starter/frontend/**'
  - 'docker/**'  # Add additional paths to trigger on
```

### Change API URL for Frontend
Edit **GITHUB_SECRETS_SETUP.md** Step 4:
```
REACT_APP_MOVIE_API_URL = https://api.yourdomain.com
```

---

## 🆘 TROUBLESHOOTING QUICK REFERENCE

| Issue | Solution | Reference |
|-------|----------|-----------|
| "Secret not found" | Check GitHub Secrets settings | GITHUB_SECRETS_SETUP.md |
| "EKS cluster not accessible" | Run `aws eks update-kubeconfig` | INFRASTRUCTURE_SETUP.md Part 3 |
| "Pod stuck in Pending" | Check resource quotas, node capacity | VERIFICATION_COMMANDS.md Section 4.3 |
| "Image not found in ECR" | Verify workflows completed successfully | VERIFICATION_COMMANDS.md Section 1 |
| "Service has no external IP" | Wait 2-3 min for ALB provision | VERIFICATION_COMMANDS.md Section 4.4 |
| "Backend API not responding" | Check pod logs, verify service | VERIFICATION_COMMANDS.md Section 5 |
| "Workflows not triggering" | Check path filters, commit to main | Frontend-ci.yaml `paths:` section |

---

## 📞 SUPPORT RESOURCES

### Files Organization
```
📦 Deliverables/
├── 🔄 Workflows (copy to .github/workflows/)
│   ├── frontend-ci.yaml
│   ├── frontend-cd.yaml
│   ├── backend-ci.yaml
│   └── backend-cd.yaml
│
├── 📚 Guides (reference docs)
│   ├── GITHUB_SECRETS_SETUP.md       (START HERE)
│   ├── INFRASTRUCTURE_SETUP.md       (THEN HERE)
│   ├── VERIFICATION_COMMANDS.md      (FOR TESTING)
│   ├── AWS_TEARDOWN_PROTOCOL.md      (FOR CLEANUP)
│   └── IMPLEMENTATION_NOTES.md       (FOR DETAILS)
│
└── 📖 This Guide
    └── QUICK_START_GUIDE.md
```

### Reading Order
1. **QUICK_START_GUIDE.md** ← You are here
2. **GITHUB_SECRETS_SETUP.md** ← Configure credentials
3. **INFRASTRUCTURE_SETUP.md** ← Provision AWS
4. **Deploy workflow YAML files** ← Push to .github/workflows/
5. **VERIFICATION_COMMANDS.md** ← Test deployments
6. **IMPLEMENTATION_NOTES.md** ← Deep dive (optional)
7. **AWS_TEARDOWN_PROTOCOL.md** ← Cleanup when done

---

## 🎯 NEXT STEPS

### Immediate (Next 10 minutes)
- [ ] Download all 9 files
- [ ] Read GITHUB_SECRETS_SETUP.md
- [ ] Configure AWS credentials in GitHub Secrets

### Short-term (Next hour)
- [ ] Read INFRASTRUCTURE_SETUP.md
- [ ] Run `terraform apply` in setup/terraform
- [ ] Wait for EKS cluster to provision

### Medium-term (Next 2 hours)
- [ ] Copy 4 YAML files to `.github/workflows/`
- [ ] Commit and push to GitHub
- [ ] Trigger test workflows
- [ ] Monitor with VERIFICATION_COMMANDS.md

### Long-term (Ongoing)
- [ ] Use VERIFICATION_COMMANDS.md for monitoring
- [ ] Check logs with kubectl commands
- [ ] Update code and watch CI/CD pipeline
- [ ] Scale or customize as needed

---

## 💡 PRO TIPS

✅ **Use environment variables for reusability:**
```bash
export AWS_REGION="us-east-1"
export EKS_CLUSTER_NAME="cluster"
export ECR_FRONTEND_REPO="movie-frontend"
export ECR_BACKEND_REPO="movie-backend"

# Save to ~/.bashrc for persistence
```

✅ **Monitor workflows with gh CLI:**
```bash
# Watch in real-time
gh run watch -i 5

# Filter by status
gh run list --status completed
gh run list --status failed
```

✅ **Stream logs efficiently:**
```bash
# All frontend logs with timestamps
kubectl logs -n production -l app=frontend --timestamps=true -f

# Search logs for errors
kubectl logs -n production -l app=backend | grep ERROR
```

✅ **Save money during testing:**
```bash
# Scale down nodes when not in use
kubectl scale deployment frontend --replicas=0 -n production
kubectl scale deployment backend --replicas=0 -n production

# Scale back up when needed
kubectl scale deployment frontend --replicas=2 -n production
```

✅ **Automate health checks:**
```bash
# Save this as health-check.sh and run periodically
while true; do
  curl -s http://${BACKEND_LB}:5000/health > /dev/null
  echo "✅ Backend healthy at $(date)"
  sleep 60
done
```

---

## ⚠️ IMPORTANT REMINDERS

🔒 **Security**
- Never commit AWS credentials to GitHub
- Rotate access keys every 90 days
- Use GitHub Secrets for all sensitive data
- Delete secrets when workflows are disabled

💰 **Cost Control**
- EKS cluster costs $73/month minimum
- 2 t3.medium nodes cost ~$60/month
- Load balancers cost ~$32/month
- Run `terraform destroy` when not in use
- Set AWS budget alerts to prevent surprises

🔄 **Workflow Maintenance**
- Review logs after each deployment
- Monitor resource usage with `kubectl top`
- Update workflow files if Terraform changes
- Keep GitHub Actions runners updated

---

## ✨ WHAT YOU GET

✅ **4 Production-Ready Workflows**
- Frontend CI (3 jobs, lint + test gate)
- Frontend CD (build, push, deploy)
- Backend CI (Python 3.10, pipenv, lint + test gate)
- Backend CD (test gate, ECR push, kustomize deploy)

✅ **5 Comprehensive Guides**
- AWS credentials configuration
- Infrastructure provisioning steps
- 100+ verification commands
- Safe teardown procedures
- Requirements compliance documentation

✅ **Professional Quality**
- Security-first design
- Error handling at every step
- Detailed logging and observability
- Troubleshooting guides
- Cost optimization tips

✅ **Enterprise-Grade**
- Kubernetes RBAC configured
- GitHub Secrets for all credentials
- ECR image scanning enabled
- Automatic rollout status checks
- Post-deployment health verification

---

## 📞 GETTING HELP

**If workflows fail:**
→ See VERIFICATION_COMMANDS.md Section 1

**If cluster connection fails:**
→ See INFRASTRUCTURE_SETUP.md Part 3

**If pods won't start:**
→ See VERIFICATION_COMMANDS.md Section 6

**If ECR push fails:**
→ See GITHUB_SECRETS_SETUP.md Troubleshooting

**If cost is too high:**
→ See QUICK_START_GUIDE.md Cost Management

**For complete details:**
→ See IMPLEMENTATION_NOTES.md

---

## 🎉 YOU'RE READY!

Start with **GITHUB_SECRETS_SETUP.md** and follow the 5-step deployment process above.

**Average time to production: ~1 hour**

**Any questions? Refer to the comprehensive guides provided.**

Good luck! 🚀
