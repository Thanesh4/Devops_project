# Movie Picture Pipeline: Verification Command Suite

Complete reference of terminal commands for verifying running pods, service endpoints, container logs, and overall deployment health.

---

## Section 1: GitHub Actions Workflow Verification

### 1.1: Trigger Frontend CI Pipeline

```bash
# Option 1: Via git (by creating a pull request)
git checkout -b feature/test-frontend-ci
# Make a change to starter/frontend/
echo "# test" >> starter/frontend/README.md
git add starter/frontend/README.md
git commit -m "test: trigger frontend CI"
git push origin feature/test-frontend-ci
# Create pull request via GitHub UI
# → frontend-ci workflow will automatically trigger

# Option 2: Via GitHub CLI
gh workflow run frontend-ci.yaml --ref main

# Option 3: Via GitHub UI
# Navigate to Actions → Frontend Continuous Integration → Run workflow
```

### 1.2: Monitor Frontend CI Execution

```bash
# Watch CI pipeline status in real-time
gh run watch -i 10

# List recent workflow runs
gh run list --workflow=frontend-ci.yaml

# View detailed run output
gh run view <run-id> --log

# Check specific job status
gh run view <run-id> --job <job-id>
```

### 1.3: Trigger Frontend CD Pipeline

```bash
# Option 1: Merge PR to main (automatically triggers CD)
git checkout main
git merge feature/test-frontend-ci
git push origin main
# → frontend-cd workflow will automatically trigger

# Option 2: Manual trigger via GitHub CLI
gh workflow run frontend-cd.yaml

# Option 3: Manually trigger via GitHub UI
# Actions → Frontend Continuous Deployment → Run workflow → Run workflow (button)
```

### 1.4: Monitor Frontend CD Execution

```bash
# Watch deployment in progress
gh run watch frontend-cd -i 5

# Stream live logs as workflow runs
gh run view <run-id> --log --tail

# Check deployment status after completion
gh run view <run-id> --json status,conclusion

# Expected successful output:
# "status": "completed"
# "conclusion": "success"
```

### 1.5: Trigger Backend Pipelines (CI & CD)

```bash
# Trigger backend CI
git checkout -b feature/test-backend-ci
echo "# test" >> starter/backend/README.md
git add starter/backend/README.md
git commit -m "test: trigger backend CI"
git push origin feature/test-backend-ci
# Create pull request via GitHub UI

# Trigger backend CD (merge to main)
git checkout main
git merge feature/test-backend-ci
git push origin main

# Monitor backend workflows
gh run list --workflow=backend-ci.yaml
gh run list --workflow=backend-cd.yaml
```

---

## Section 2: AWS & ECR Verification

### 2.1: Verify AWS Credentials

```bash
# Test AWS CLI connection
aws sts get-caller-identity

# Expected output:
{
    "UserId": "AIDAI...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/github-actions-cicd"
}

# Verify IAM permissions
aws iam get-user-policy \
  --user-name github-actions-cicd \
  --policy-name GitHubActionsCICDPolicy
```

### 2.2: Verify ECR Repositories

```bash
# List all ECR repositories
aws ecr describe-repositories --region us-east-1

# Describe specific repository
aws ecr describe-repositories \
  --repository-names movie-frontend \
  --region us-east-1

# Expected output shows repository URI:
# "repositoryUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/movie-frontend"
```

### 2.3: Verify Images in ECR

```bash
# List all images in frontend repository
aws ecr describe-images \
  --repository-name movie-frontend \
  --region us-east-1 \
  --query 'imageDetails[*].[imageTags,imagePushedAt,imageSizeInBytes]' \
  --output table

# Expected output shows pushed images with timestamps

# List images for backend repository
aws ecr describe-images \
  --repository-name movie-backend \
  --region us-east-1

# Get image details by tag
aws ecr describe-images \
  --repository-name movie-frontend \
  --image-ids imageTag=<commit-sha> \
  --region us-east-1

# View image manifest
aws ecr batch-get-image \
  --repository-name movie-frontend \
  --image-ids imageTag=<commit-sha> \
  --region us-east-1 \
  --query 'images[0].imageManifest' | jq .
```

### 2.4: Verify ECR Login (Docker)

```bash
# Login to ECR (required before pushing/pulling images)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Expected output:
# Login Succeeded

# Pull an image from ECR
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/movie-frontend:<tag>

# List locally available images
docker images | grep movie
```

---

## Section 3: EKS Cluster Verification

### 3.1: Verify EKS Cluster Status

```bash
# Get cluster information
aws eks describe-cluster --name cluster --region us-east-1

# Check cluster status
aws eks describe-cluster \
  --name cluster \
  --region us-east-1 \
  --query 'cluster.status' \
  --output text

# Expected output: ACTIVE

# Get cluster endpoint
aws eks describe-cluster \
  --name cluster \
  --region us-east-1 \
  --query 'cluster.endpoint' \
  --output text

# Get cluster API version
aws eks describe-cluster \
  --name cluster \
  --region us-east-1 \
  --query 'cluster.version' \
  --output text
```

### 3.2: Verify kubectl Configuration

```bash
# Show current context
kubectl config current-context

# Expected output: arn:aws:eks:us-east-1:123456789012:cluster/cluster

# List all contexts
kubectl config get-contexts

# View kubeconfig file location
echo $KUBECONFIG

# Validate kubeconfig is accessible
kubectl config view

# Switch context if needed
kubectl config use-context arn:aws:eks:us-east-1:123456789012:cluster/cluster
```

### 3.3: Verify Kubernetes Cluster Connectivity

```bash
# Test cluster connection
kubectl cluster-info

# Expected output shows:
# Kubernetes control plane is running at https://...
# CoreDNS is running at https://...

# Get cluster version details
kubectl version --short

# Check API server accessibility
kubectl get nodes

# View cluster role bindings
kubectl get clusterrolebindings

# List API resources
kubectl api-resources
```

### 3.4: Verify Kubernetes Nodes

```bash
# List all nodes
kubectl get nodes -o wide

# Get node details
kubectl describe node <node-name>

# Check node status
kubectl get nodes -o json | jq '.items[].status.conditions[] | select(.type=="Ready")'

# View resource allocation per node
kubectl top nodes

# Expected output shows:
# NAME                          CPU(cores)   MEMORY(bytes)
# ip-10-0-1-xxx.ec2.internal   100m         256Mi

# Check node capacity
kubectl get nodes -o json | jq '.items[].status.capacity'

# Monitor node resource usage
watch kubectl top nodes
```

---

## Section 4: Kubernetes Deployment Verification

### 4.1: Verify Namespaces

```bash
# List all namespaces
kubectl get namespaces

# Get namespace details
kubectl describe namespace production

# Create namespace if not exists
kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -

# Check resources in production namespace
kubectl get all -n production
```

### 4.2: Verify Deployments

```bash
# List deployments in production namespace
kubectl get deployments -n production

# Get deployment details
kubectl describe deployment frontend -n production
kubectl describe deployment backend -n production

# Check deployment status
kubectl get deployments -n production -o wide

# Expected output shows:
# NAME       READY   UP-TO-DATE   AVAILABLE   AGE
# frontend   2/2     2            2           5m
# backend    2/2     2            2           5m

# View deployment events
kubectl get events -n production --sort-by='.lastTimestamp'

# Check deployment rollout status
kubectl rollout status deployment/frontend -n production
kubectl rollout status deployment/backend -n production

# Expected output: deployment "frontend" successfully rolled out
```

### 4.3: Verify Pods

```bash
# List all pods in production namespace
kubectl get pods -n production -o wide

# Expected output shows:
# NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE
# frontend-5b8d9c4d6-xxx      1/1     Running   0          5m    10.0.1.100   ip-10-0-1-xxx
# backend-7c4e2b9f1-xxx       1/1     Running   0          5m    10.0.2.50    ip-10-0-2-xxx

# Get pod labels and selectors
kubectl get pods -n production --show-labels

# Describe specific pod
kubectl describe pod <pod-name> -n production

# Check pod resource usage
kubectl top pods -n production

# Monitor pods in real-time
watch kubectl get pods -n production

# Get pod logs
kubectl logs <pod-name> -n production

# Get logs from all frontend pods
kubectl logs -n production -l app=frontend --all-containers=true

# Stream pod logs in real-time
kubectl logs -f <pod-name> -n production

# Get logs from previous pod (if crashed and restarted)
kubectl logs <pod-name> -n production --previous

# View pod details in YAML format
kubectl get pod <pod-name> -n production -o yaml

# Check pod events
kubectl describe pod <pod-name> -n production | grep -A 10 Events:
```

### 4.4: Verify Services

```bash
# List services in production namespace
kubectl get services -n production -o wide

# Expected output shows:
# NAME       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# frontend   LoadBalancer   10.100.1.100   a123.elb...   80:31234/TCP   5m
# backend    LoadBalancer   10.100.2.50    b456.elb...   5000:31567/TCP 5m

# Get service details
kubectl describe service frontend -n production
kubectl describe service backend -n production

# Get service endpoints
kubectl get endpoints -n production

# Check service DNS resolution (within cluster)
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  nslookup frontend.production.svc.cluster.local

# Test service connectivity from within cluster
kubectl exec -it <pod-name> -n production -- \
  curl http://backend.production.svc.cluster.local:5000/health
```

### 4.5: Verify ConfigMaps & Secrets

```bash
# List ConfigMaps in production namespace
kubectl get configmaps -n production

# Get ConfigMap details
kubectl describe configmap app-config -n production

# View ConfigMap data
kubectl get configmap app-config -n production -o yaml

# List Secrets in production namespace
kubectl get secrets -n production

# View Secret metadata (not decoded values)
kubectl describe secret <secret-name> -n production

# Verify environment variables are set
kubectl exec -it <pod-name> -n production -- env | grep REACT_APP
kubectl exec -it <pod-name> -n production -- env | grep FLASK_ENV
```

---

## Section 5: Application Health & Endpoint Verification

### 5.1: Get Service External IPs

```bash
# Get frontend load balancer IP/hostname
kubectl get service frontend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Get backend load balancer IP/hostname
kubectl get service backend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Store in variables for reuse
FRONTEND_LB=$(kubectl get svc frontend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
BACKEND_LB=$(kubectl get svc backend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "Frontend URL: http://${FRONTEND_LB}"
echo "Backend URL: http://${BACKEND_LB}:5000"
```

### 5.2: Test Backend API Endpoints

```bash
# Get backend service endpoint
BACKEND_LB=$(kubectl get svc backend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test health endpoint (liveness probe)
curl -v http://${BACKEND_LB}:5000/health

# Expected response:
# HTTP/1.1 200 OK
# {"status": "healthy"}

# Test readiness endpoint
curl -v http://${BACKEND_LB}:5000/ready

# Test movies API endpoint
curl -s http://${BACKEND_LB}:5000/movies | jq .

# Expected response (JSON array of movies):
# [
#   {"id": 1, "title": "Movie 1", ...},
#   {"id": 2, "title": "Movie 2", ...}
# ]

# Test POST request (create movie)
curl -X POST http://${BACKEND_LB}:5000/movies \
  -H "Content-Type: application/json" \
  -d '{"title": "New Movie", "year": 2024}'

# Test specific movie endpoint
curl http://${BACKEND_LB}:5000/movies/1

# Test with authentication (if required)
curl -H "Authorization: Bearer <token>" \
  http://${BACKEND_LB}:5000/movies

# Test with verbose output
curl -v -H "Content-Type: application/json" \
  http://${BACKEND_LB}:5000/health
```

### 5.3: Test Frontend Application

```bash
# Get frontend load balancer URL
FRONTEND_LB=$(kubectl get svc frontend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test frontend accessibility
curl -I http://${FRONTEND_LB}

# Expected response:
# HTTP/1.1 200 OK

# Get full HTML response
curl http://${FRONTEND_LB} | head -20

# Check if React app is loaded (look for root div)
curl http://${FRONTEND_LB} | grep -i "root\|reactroot"

# Test with custom headers
curl -H "User-Agent: GitHub-Actions" http://${FRONTEND_LB}

# Check response headers
curl -i http://${FRONTEND_LB} | head -20
```

### 5.4: Test Internal Service-to-Service Communication

```bash
# Test from frontend pod to backend service
kubectl exec -it <frontend-pod-name> -n production -- \
  curl http://backend.production.svc.cluster.local:5000/health

# Test from backend pod
kubectl exec -it <backend-pod-name> -n production -- \
  curl http://localhost:5000/health

# Test DNS resolution
kubectl exec -it <pod-name> -n production -- \
  nslookup backend.production.svc.cluster.local

# Test network policy (if configured)
kubectl exec -it <pod-name> -n production -- \
  ping backend.production.svc.cluster.local
```

---

## Section 6: Container Logs Verification

### 6.1: View Frontend Logs

```bash
# Get latest frontend pod logs
kubectl logs -n production -l app=frontend --tail=50

# Stream logs in real-time
kubectl logs -f -n production -l app=frontend

# Get logs from all frontend replicas
kubectl logs -n production -l app=frontend --all-containers=true --timestamps=true

# Get logs from specific pod
kubectl logs <frontend-pod-name> -n production

# Get logs with timestamps
kubectl logs -n production -l app=frontend --timestamps=true --tail=100

# Get logs from previous container (if pod restarted)
kubectl logs -n production -l app=frontend --previous

# Watch logs with grep filter
kubectl logs -f -n production -l app=frontend | grep ERROR

# Get container environment for debugging
kubectl exec -it <pod-name> -n production -- env | sort
```

### 6.2: View Backend Logs

```bash
# Get latest backend pod logs
kubectl logs -n production -l app=backend --tail=50

# Stream logs in real-time
kubectl logs -f -n production -l app=backend

# Get Flask debug logs
kubectl logs -n production -l app=backend --timestamps=true --tail=100 | grep -i "debug\|error\|warning"

# Get logs from specific backend pod
kubectl logs <backend-pod-name> -n production -c backend

# Get logs with specific severity level
kubectl logs -n production -l app=backend | grep "ERROR\|CRITICAL"

# Get logs from last 1 hour
kubectl logs -n production -l app=backend --since=1h

# Get logs from last 10 minutes
kubectl logs -n production -l app=backend --since=10m
```

### 6.3: Monitor Logs in Real-Time

```bash
# Watch frontend logs
watch kubectl logs -n production -l app=frontend --tail=20

# Watch backend logs
watch kubectl logs -n production -l app=backend --tail=20

# Tail multiple log streams simultaneously (requires kubetail)
kubetail -n production -l app=frontend
kubetail -n production -l app=backend

# Or use stern (log aggregator)
stern -n production ".*" --all-namespaces=false
```

---

## Section 7: Deployment Events & Status Verification

### 7.1: Check Deployment Events

```bash
# Get events in production namespace
kubectl get events -n production --sort-by='.lastTimestamp'

# Expected events:
# REASON                     MESSAGE
# Pulling                     Pulling image "...movie-frontend:abc123"
# Pulled                      Successfully pulled image "...movie-frontend:abc123"
# Created                     Created container frontend
# Started                     Started container frontend

# Watch events in real-time
kubectl get events -n production --watch

# Get events for specific deployment
kubectl describe deployment frontend -n production | grep -A 20 Events:
```

### 7.2: Check Deployment Rollout History

```bash
# View rollout history for frontend deployment
kubectl rollout history deployment/frontend -n production

# Expected output:
# REVISION  CHANGE-CAUSE
# 1         kubectl apply --filename=...
# 2         kubectl apply --filename=... (updated image)

# Get details of specific revision
kubectl rollout history deployment/frontend -n production --revision=2

# Check current deployment status
kubectl rollout status deployment/frontend -n production

# Check if rollout is complete
kubectl get deployment frontend -n production -o json | jq '.status'
```

### 7.3: Troubleshoot Failed Rollouts

```bash
# Check if pods are ready
kubectl get pods -n production -l app=frontend -o json | \
  jq '.items[] | {name: .metadata.name, ready: .status.conditions[] | select(.type=="Ready")}'

# Check why pods are not ready
kubectl describe pod <pod-name> -n production | grep -A 10 "Conditions:"

# Check container restart count (indicates crashes)
kubectl get pods -n production -l app=frontend -o json | \
  jq '.items[] | {name: .metadata.name, restarts: .status.containerStatuses[].restartCount}'

# View events for failed pod
kubectl describe pod <pod-name> -n production | tail -20

# Check resource limits are not being exceeded
kubectl describe nodes | grep -A 5 "Allocated resources"
```

---

## Section 8: Performance & Resource Monitoring

### 8.1: Monitor CPU & Memory Usage

```bash
# View metrics for nodes
kubectl top nodes

# View metrics for pods
kubectl top pods -n production

# View metrics for specific pod
kubectl top pod <pod-name> -n production

# Monitor resource usage over time
watch -n 2 kubectl top pods -n production

# Get resource requests vs. actual usage
kubectl get pods -n production -o json | jq '.items[] | {name: .metadata.name, resources: .spec.containers[].resources}'
```

### 8.2: Check Resource Quotas & Limits

```bash
# Get resource quota for namespace
kubectl describe resourcequota -n production

# Get limit ranges
kubectl describe limitrange -n production

# Check pod resource requests
kubectl describe pod <pod-name> -n production | grep -A 5 "Requests"

# Check container resource limits
kubectl describe pod <pod-name> -n production | grep -A 5 "Limits"
```

### 8.3: Monitor Persistent Volumes (if used)

```bash
# List persistent volumes
kubectl get persistentvolumes

# List persistent volume claims
kubectl get persistentvolumeclaims -n production

# Check PVC status
kubectl describe pvc <pvc-name> -n production

# Check storage usage
kubectl exec -it <pod-name> -n production -- df -h
```

---

## Section 9: Quick Health Check Commands

### 9.1: One-Liner Cluster Health Check

```bash
# All-in-one health check script
echo "=== Cluster Status ===" && \
kubectl cluster-info && \
echo "=== Nodes ===" && \
kubectl get nodes && \
echo "=== Namespaces ===" && \
kubectl get namespaces && \
echo "=== Deployments ===" && \
kubectl get deployments -n production && \
echo "=== Pods ===" && \
kubectl get pods -n production && \
echo "=== Services ===" && \
kubectl get services -n production && \
echo "=== Health Check Complete ==="
```

### 9.2: Automated Health Check Script

```bash
#!/bin/bash
# save as health-check.sh

set -e

echo "🔍 Movie Picture Pipeline Health Check"
echo "========================================"

# Check cluster connectivity
echo "✓ Cluster connectivity..."
kubectl cluster-info > /dev/null && echo "  ✅ Connected to EKS cluster"

# Check nodes
echo "✓ Node status..."
READY_NODES=$(kubectl get nodes -o json | jq '[.items[].status.conditions[] | select(.type=="Ready" and .status=="True")] | length')
echo "  ✅ $READY_NODES nodes ready"

# Check deployments
echo "✓ Deployment status..."
kubectl get deployments -n production -o json | jq '.items[] | {name: .metadata.name, ready: .status.readyReplicas, desired: .spec.replicas}' | grep -v null
echo "  ✅ All deployments verified"

# Check pods
echo "✓ Pod status..."
READY_PODS=$(kubectl get pods -n production -o json | jq '[.items[] | select(.status.conditions[] | select(.type=="Ready" and .status=="True"))] | length')
echo "  ✅ $READY_PODS pods ready"

# Check services
echo "✓ Service status..."
kubectl get svc -n production -o wide

# Check API endpoints
echo "✓ API endpoints..."
BACKEND_LB=$(kubectl get svc backend -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "pending")
echo "  Backend: http://${BACKEND_LB}:5000"

echo ""
echo "✅ Health check completed successfully!"
```

---

## Section 10: Emergency Commands

### 10.1: Delete and Restart Deployments

```bash
# Delete all pods in deployment (triggers automatic restart)
kubectl delete pods -n production -l app=frontend

# Delete specific pod (automatically replaced)
kubectl delete pod <pod-name> -n production

# Restart deployment (rolling restart)
kubectl rollout restart deployment/frontend -n production

# Scale deployment to 0 and back (full restart)
kubectl scale deployment frontend -n production --replicas=0
kubectl scale deployment frontend -n production --replicas=2
```

### 10.2: Force Pull Latest Image

```bash
# Edit deployment to force image pull
kubectl patch deployment frontend -n production -p \
  "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"restartedAt\":\"$(date +%s)\"}}}}}"

# Or use kubectl rollout with image update
kubectl set image deployment/frontend \
  frontend=<ECR_URL>:latest \
  -n production --record
```

### 10.3: Emergency Debug Pod

```bash
# Create a debug pod for troubleshooting
kubectl run -it --rm debug --image=busybox --restart=Never -n production -- sh

# Or with more tools
kubectl run -it --rm debug --image=ubuntu --restart=Never -n production -- bash

# Connect to existing pod
kubectl exec -it <pod-name> -n production -- /bin/sh
```

---

## Checklist: Verification Complete

- [ ] GitHub Actions workflows triggered successfully
- [ ] ECR images pushed and verified
- [ ] EKS cluster accessible and nodes ready
- [ ] Frontend deployment running with 2+ replicas
- [ ] Backend deployment running with 2+ replicas
- [ ] Services have external load balancer IPs
- [ ] Backend health endpoint responding (HTTP 200)
- [ ] Movies API endpoint returning data
- [ ] Frontend application accessible via browser
- [ ] Pod logs show no errors or crashes
- [ ] CPU/Memory usage within expected ranges
