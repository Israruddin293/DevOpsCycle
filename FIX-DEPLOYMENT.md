# Quick Fix for InvalidImageName Error

## Problem
Your deployments have `image: your-registry/backend:latest` instead of the actual GHCR image path.

## Solution

### On EC2, run these commands:

```bash
# 1. Update backend deployment
kubectl set image deployment/backend \
  backend=ghcr.io/israruddin293/devopscycle/backend:latest \
  -n microservices

# 2. Update frontend deployment
kubectl set image deployment/frontend \
  frontend=ghcr.io/israruddin293/devopscycle/frontend:latest \
  -n microservices

# 3. Fix readOnlyRootFilesystem (if needed)
kubectl patch deployment backend -n microservices --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/securityContext/readOnlyRootFilesystem", "value": false}]'

kubectl patch deployment frontend -n microservices --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/securityContext/readOnlyRootFilesystem", "value": false}]'

# 4. Check status
kubectl get pods -n microservices -w
```

## If Images are Private

Create image pull secret:

```bash
# Generate GitHub token first:
# GitHub → Settings → Developer settings → Personal access tokens → Generate new token
# Select scope: read:packages

kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=Israruddin293 \
  --docker-password=<YOUR-GITHUB-TOKEN> \
  --docker-email=<YOUR-EMAIL> \
  -n microservices

# Add to service account
kubectl patch serviceaccount default -n microservices \
  -p '{"imagePullSecrets": [{"name": "ghcr-secret"}]}'

# Restart deployments
kubectl rollout restart deployment/backend -n microservices
kubectl rollout restart deployment/frontend -n microservices
```

## Verify

```bash
# Check pods
kubectl get pods -n microservices

# Should show:
# NAME                        READY   STATUS    RESTARTS   AGE
# backend-xxx                 1/1     Running   0          2m
# frontend-xxx                1/1     Running   0          2m
# redis-xxx                   1/1     Running   0          5m

# Check logs
kubectl logs -f deployment/backend -n microservices
kubectl logs -f deployment/frontend -n microservices
```

## Access Application

```bash
# Get NodePort
kubectl get svc -n ingress-nginx

# Access in browser
http://<EC2-PUBLIC-IP>:30080
```

## For Future Deployments

Update your local k8s files and commit:

```bash
# On your local machine
cd project

# Files are already updated in the repository
git pull origin main

# Or manually update:
# k8s/backend-deployment.yaml - line 24
# k8s/frontend-deployment.yaml - line 24
```
