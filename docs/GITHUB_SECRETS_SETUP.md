# GitHub Repository Secrets Configuration Guide

## Overview
This guide provides exact key-value pairs required in your GitHub Repository Secrets for secure CI/CD pipeline operation. All credentials must be stored as repository secrets (not environment variables) to prevent accidental exposure.

---

## Prerequisites
- GitHub repository access with admin privileges
- AWS account with IAM user credentials
- AWS account configured with appropriate permissions

---

## Step 1: Access GitHub Repository Secrets

1. Navigate to your GitHub repository
2. Click **Settings** (top navigation bar)
3. In the left sidebar, click **Secrets and variables** → **Actions**
4. Click **New repository secret** button

---

## Step 2: Configure Required Secrets

### Secret #1: AWS_ACCESS_KEY_ID

| Property | Value |
|----------|-------|
| **Secret Name** | `AWS_ACCESS_KEY_ID` |
| **Description** | AWS IAM Access Key ID for CI/CD automation |
| **Value** | Your actual AWS Access Key ID (20 characters, alphanumeric) |
| **Source** | AWS IAM Console → Users → Your CI/CD User → Security Credentials → Access Keys |
| **Required** | YES - Used in all CI/CD workflows |

**How to obtain:**
```bash
# Via AWS CLI (if already configured)
aws configure get aws_access_key_id

# Or retrieve from AWS Console:
# 1. Sign in to AWS Management Console
# 2. Navigate to IAM > Users > Select your CI/CD user
# 3. Click "Security credentials" tab
# 4. Under "Access keys", click "Create access key"
# 5. Copy the "Access key ID" value
```

---

### Secret #2: AWS_SECRET_ACCESS_KEY

| Property | Value |
|----------|-------|
| **Secret Name** | `AWS_SECRET_ACCESS_KEY` |
| **Description** | AWS IAM Secret Access Key for CI/CD automation |
| **Value** | Your actual AWS Secret Access Key (40 characters) |
| **Source** | AWS IAM Console (shown only once during creation) |
| **Required** | YES - Used in all CI/CD workflows |

**How to obtain:**
```bash
# Via AWS CLI
aws configure get aws_secret_access_key

# Or from AWS Console:
# When creating access key in IAM, copy the "Secret access key" immediately
# (This value is only shown once and cannot be retrieved later)
# If lost, create a new access key pair
```

---

### Secret #3: AWS_REGION

| Property | Value |
|----------|-------|
| **Secret Name** | `AWS_REGION` |
| **Description** | AWS region for EKS cluster and ECR repositories |
| **Value** | `us-east-1` |
| **Source** | Terraform configuration (`setup/terraform`) |
| **Required** | NO - Already defined in workflow env, but recommended for consistency |

---

### Secret #4: REACT_APP_MOVIE_API_URL

| Property | Value |
|----------|-------|
| **Secret Name** | `REACT_APP_MOVIE_API_URL` |
| **Description** | Backend API URL exposed to React frontend |
| **Value Examples** | `http://localhost:5000` (dev) OR `https://api.yourdomain.com` (prod) |
| **Default Fallback** | `http://localhost:5000` (if secret not set) |
| **Required** | NO - Frontend CD has fallback |
| **Notes** | Must start with `http://` or `https://` for browser compatibility |

**Configuration options:**
```yaml
# Development (local testing)
http://localhost:5000

# Production (EKS service endpoint)
# After deploying backend, run:
# kubectl get svc backend -n production
# Use LoadBalancer external IP or internal service DNS:
# http://backend.production.svc.cluster.local:5000

# Custom domain (with DNS configured)
https://api.yourdomain.com
```

---

## Step 3: Required IAM Policy for CI/CD User

Create an IAM user or role with the following permissions for secure CI/CD operation:

### IAM Policy JSON

Save this as `ci-cd-policy.json` and attach to your CI/CD IAM user:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRRepositoryAccess",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:DescribeImages",
        "ecr:ListImages"
      ],
      "Resource": "arn:aws:ecr:us-east-1:ACCOUNT_ID:repository/movie-*"
    },
    {
      "Sid": "EKSClusterAccess",
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "arn:aws:eks:us-east-1:ACCOUNT_ID:cluster/cluster"
    },
    {
      "Sid": "IAMPassRoleAccess",
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": [
        "arn:aws:iam::ACCOUNT_ID:role/eks-service-role",
        "arn:aws:iam::ACCOUNT_ID:role/eks-node-role"
      ]
    }
  ]
}
```

**Replace `ACCOUNT_ID` with your AWS account ID:**
```bash
# Retrieve your AWS Account ID
aws sts get-caller-identity --query "Account" --output text
```

---

## Step 4: Create AWS IAM User for CI/CD

Execute these commands in AWS CLI:

```bash
# Set your AWS account ID and region
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export AWS_REGION="us-east-1"
export CI_CD_USER="github-actions-cicd"

# Create IAM user
aws iam create-user --user-name "${CI_CD_USER}"

# Create access key (this will output ACCESS_KEY_ID and SECRET_ACCESS_KEY)
aws iam create-access-key --user-name "${CI_CD_USER}"

# Attach the CI/CD policy (after saving the policy file above)
aws iam put-user-policy \
  --user-name "${CI_CD_USER}" \
  --policy-name "GitHubActionsCICDPolicy" \
  --policy-document file://ci-cd-policy.json

# Verify the policy was attached
aws iam get-user-policy \
  --user-name "${CI_CD_USER}" \
  --policy-name "GitHubActionsCICDPolicy"
```

---

## Step 5: Verify Secrets in GitHub

After adding all secrets, verify they're accessible to workflows:

```bash
# In GitHub Actions workflow logs, you should see:
# ✅ AWS credentials configured
# ✅ Logged in to Amazon ECR
# ✅ Frontend image pushed successfully

# Secrets will NOT display in logs due to GitHub masking
# You will see [***] instead of actual values
```

---

## Security Best Practices

### ✅ Do's
- ✅ Rotate AWS access keys every 90 days
- ✅ Use separate IAM users for CI/CD vs. manual operations
- ✅ Enable MFA on the CI/CD IAM user account
- ✅ Audit GitHub Actions logs regularly
- ✅ Use least-privilege IAM policies (restrict to required resources)
- ✅ Delete old/unused access keys immediately
- ✅ Store secrets only as GitHub Secrets, never in code

### ❌ Don'ts
- ❌ NEVER commit AWS credentials to git
- ❌ NEVER hardcode secrets in YAML files
- ❌ NEVER share access keys via email or Slack
- ❌ NEVER log or print secrets in workflow output
- ❌ NEVER use personal AWS credentials for CI/CD
- ❌ NEVER disable GitHub secret masking

---

## Troubleshooting

### Issue: "AWS_ACCESS_KEY_ID not found"
**Solution:** Ensure the secret name matches exactly (case-sensitive):
```bash
# ✅ Correct
AWS_ACCESS_KEY_ID

# ❌ Wrong
aws_access_key_id
Aws_Access_Key_Id
```

### Issue: "UnauthorizedOperation: You are not authorized to perform"
**Solution:** Verify IAM policy is attached to the CI/CD user:
```bash
aws iam list-user-policies --user-name github-actions-cicd
aws iam get-user-policy --user-name github-actions-cicd --policy-name GitHubActionsCICDPolicy
```

### Issue: "Unable to connect to the Docker daemon"
**Solution:** This occurs in CI/CD; the Docker daemon runs in GitHub Actions runners, not your local machine.

### Issue: GitHub Action skipped (no secrets detected)
**Solution:** Ensure secrets are set BEFORE pushing code that references them. GitHub Actions caches secrets.

---

## Checklist: Secrets Configuration Complete

- [ ] `AWS_ACCESS_KEY_ID` created and added to GitHub Secrets
- [ ] `AWS_SECRET_ACCESS_KEY` created and added to GitHub Secrets
- [ ] `AWS_REGION` added (value: `us-east-1`)
- [ ] `REACT_APP_MOVIE_API_URL` configured (with backend endpoint URL)
- [ ] IAM policy attached to CI/CD user
- [ ] IAM user has ECR and EKS permissions
- [ ] Secrets tested in a workflow run (verify no auth errors)
- [ ] All secrets masked in GitHub Actions logs
- [ ] MFA enabled on CI/CD IAM user (optional but recommended)

---

## Quick Reference: Secret Names

Use this table for copy-paste accuracy:

| Environment Variable | GitHub Secret Name | Example Value |
|----------------------|-------------------|---------------|
| `AWS_ACCESS_KEY_ID` | `AWS_ACCESS_KEY_ID` | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_SECRET_ACCESS_KEY` | `AWS_SECRET_ACCESS_KEY` | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `AWS_REGION` | `AWS_REGION` | `us-east-1` |
| `REACT_APP_MOVIE_API_URL` | `REACT_APP_MOVIE_API_URL` | `http://backend.production.svc.cluster.local:5000` |
