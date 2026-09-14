# Movie Picture Pipeline - DevOps CI/CD Project

A production-grade CI/CD pipeline for a Movie Picture web application using GitHub Actions, Docker, AWS ECR, and Kubernetes (EKS).

## Project Structure

```
.
â”œâ”€â”€ .github/workflows/        # CI/CD Pipeline Workflows
â”‚   â”œâ”€â”€ frontend-ci.yaml      # Frontend CI (lint, test, build)
â”‚   â”œâ”€â”€ frontend-cd.yaml      # Frontend CD (build, push to ECR, deploy to EKS)
â”‚   â”œâ”€â”€ backend-ci.yaml       # Backend CI (lint, test, build)
â”‚   â””â”€â”€ backend-cd.yaml       # Backend CD (build, push to ECR, deploy to EKS)
â”œâ”€â”€ starter/
â”‚   â”œâ”€â”€ frontend/             # React.js Frontend Application
â”‚   â”‚   â”œâ”€â”€ k8s/              # Kubernetes manifests (Kustomize)
â”‚   â”‚   â””â”€â”€ Dockerfile        # Frontend Docker image
â”‚   â””â”€â”€ backend/              # Python Flask Backend Application
â”‚       â”œâ”€â”€ k8s/              # Kubernetes manifests (Kustomize)
â”‚       â””â”€â”€ Dockerfile        # Backend Docker image
â”œâ”€â”€ setup/
â”‚   â””â”€â”€ terraform/            # Terraform IaC for AWS EKS + ECR
â””â”€â”€ docs/                     # Complete documentation guides
```

## CI/CD Pipeline

| Workflow | Trigger | Actions |
|----------|---------|---------|
| frontend-ci | PR to main (frontend changes) | Lint, Test, Build Docker |
| frontend-cd | Push to main (frontend changes) | Build, Push ECR, Deploy EKS |
| backend-ci | PR to main (backend changes) | Lint, Test, Build Docker |
| backend-cd | Push to main (backend changes) | Build, Push ECR, Deploy EKS |

## Quick Start

See [docs/QUICK_START_GUIDE.md](docs/QUICK_START_GUIDE.md) to get started.

## Required GitHub Secrets

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION` (= us-east-1)
- `REACT_APP_MOVIE_API_URL`

## Tech Stack

- **Frontend:** React.js (Node 18)
- **Backend:** Python Flask (Python 3.10, Pipenv)
- **Containers:** Docker + AWS ECR
- **Orchestration:** Kubernetes on AWS EKS
- **IaC:** Terraform
- **CI/CD:** GitHub Actions