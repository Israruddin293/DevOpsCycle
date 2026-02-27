# Troubleshooting Checklist for Kubeadm Deployment

## Quick Diagnostic Commands

Run these commands on your EC2 instance to diagnose issues:

```bash
# 1. Check pod status
kubectl get pods -n microservices

# 2. Check detailed pod info
kubectl get pods -n microservices -o wide

# 3. Check events (most important for troubleshooting)
kubectl get events -n microservices --sort-by='.lastTimestamp'

# 4. Check specific pod details
kubectl describe pod <pod-name> -n microservices

# 5. Check pod logs
kubectl logs <pod-name> -n microservices

# 6. Check all resources
kubectl get all -n microservices
```

## Common Issues and Solutions

### Issue 1: Pods in ImagePullBackOff

**Symptoms:**
```
NAME                        READY   STATUS             RESTARTS   AGE
backend-xxx                 0/1     ImagePullBackOff   0          2m
frontend-xxx                0/1     ImagePullBackOff   0          2m
```

**Reason:** Kubernetes can't pull images from GitHub Container Registry (GHCR)

**Solutions:**

#### Option A: Make Images Public (Easiest)
```bash
# On GitHub:
# 1. Go to https://github.com/Israruddin293?tab=packages
# 2. Click on "backend" package
# 3. Package settings → Change visibility → Public
# 4. Repeat for "frontend" package

# Then restart pods:
kubectl rollout restart deployment/backend -n microservices
kubectl rollout restart deployment/frontend -n microservices
```

#### Option B: Create Image Pull Secret
```bash
# Generate GitHub token:
# GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
# Generate new token with 'read:packages' scope

# Create secret
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=Israruddin293 \
  --docker-password=<YOUR-GITHUB-TOKEN> \
  --docker-email=<YOUR-EMAIL> \
  -n microservices

# Add to default service account
kubectl patch serviceaccount default -n microservices \
  -p '{"imagePullSecrets": [{"name": "ghcr-secret"}]}'

# Restart deployments
kubectl rollout restart deployment/backend -n microservices
kubectl rollout restart deployment/frontend -n microservices
```

### Issue 2: Pods in CrashLoopBackOff

**Symptoms:**
```
NAME                        READY   STATUS             RESTARTS   AGE
backend-xxx                 0/1     CrashLoopBackOff   5          5m
```

**Reason:** Application is crashing after starting

**Check logs:**
```bash
kubectl logs <pod-name> -n microservices
kubectl logs <pod-name> -n microservices --previous
```

**Common causes:**
1. **Redis connection failed** - Backend can't connect to Redis
2. **Missing dependencies** - Python packages not installed
3. **Port already in use** - Unlikely in Kubernetes

**Solutions:**

```bash
# Check if Redis is running
kubectl get pods -n microservices | grep redis

# Check Redis logs
kubectl logs deployment/redis -n microservices

# Check backend logs
kubectl logs deployment/backend -n microservices

# Check if services exist
kubectl get svc -n microservices
```

### Issue 3: Pods in Pending State

**Symptoms:**
```
NAME                        READY   STATUS    RESTARTS   AGE
backend-xxx                 0/1     Pending   0          5m
```

**Reason:** Not enough resources or scheduling issues

**Check:**
```bash
# Describe pod to see reason
kubectl describe pod <pod-name> -n microservices

# Check node resources
kubectl top nodes
kubectl describe nodes
```

**Solutions:**
```bash
# If insufficient CPU/memory, reduce resource requests
kubectl edit deployment backend -n microservices
# Reduce requests.cpu and requests.memory

# Or scale down replicas
kubectl scale deployment backend --replicas=1 -n microservices
kubectl scale deployment frontend --replicas=1 -n microservices
```

### Issue 4: ReadOnlyRootFilesystem Error

**Symptoms:**
```
Error: failed to create containerd task: failed to create shim task: 
OCI runtime create failed: runc create failed: unable to start container process: 
error during container init: error mounting "/tmp" to rootfs at "/tmp": 
read-only file system
```

**Reason:** Backend/Frontend containers have `readOnlyRootFilesystem: true` but need write access

**Solution:**
```bash
# Edit deployments to set readOnlyRootFilesystem: false
kubectl edit deployment backend -n microservices
# Find: readOnlyRootFilesystem: true
# Change to: readOnlyRootFilesystem: false

kubectl edit deployment frontend -n microservices
# Find: readOnlyRootFilesystem: true
# Change to: readOnlyRootFilesystem: false
```

### Issue 5: Ingress Not Working

**Symptoms:**
- Pods are running
- Can't access application via browser

**Check:**
```bash
# Check ingress controller
kubectl get pods -n ingress-nginx

# Check ingress resource
kubectl get ingress -n microservices
kubectl describe ingress microservices-ingress -n microservices

# Check services
kubectl get svc -n microservices
kubectl get svc -n ingress-nginx
```

**Get NodePort:**
```bash
kubectl get svc -n ingress-nginx

# Look for ingress-nginx-controller
# Example output:
# NAME                       TYPE       PORT(S)
# ingress-nginx-controller   NodePort   80:30080/TCP,443:30443/TCP
```

**Access application:**
```
http://<EC2-PUBLIC-IP>:30080
```

**If ingress controller not installed:**
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml

# Wait for it to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### Issue 6: HPA Not Working

**Symptoms:**
```bash
kubectl get hpa -n microservices
# Shows: <unknown>/70%
```

**Reason:** Metrics server not installed or not working

**Solution:**
```bash
# Install metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For kubeadm, add insecure TLS flag
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Wait and check
kubectl top nodes
kubectl top pods -n microservices
```

### Issue 7: Network Policy Blocking Traffic

**Symptoms:**
- Pods running
- Frontend can't reach backend
- Backend can't reach Redis

**Solution:**
```bash
# Temporarily disable network policies
kubectl delete networkpolicy --all -n microservices

# Test if application works
# If yes, network policies are too restrictive

# Re-apply with modifications if needed
kubectl apply -f k8s/network-policy.yaml
```

## Complete Diagnostic Script

Save this as `diagnose.sh` and run on EC2:

```bash
#!/bin/bash

echo "=== Kubernetes Cluster Status ==="
kubectl get nodes

echo -e "\n=== Namespace Resources ==="
kubectl get all -n microservices

echo -e "\n=== Pod Details ==="
kubectl get pods -n microservices -o wide

echo -e "\n=== Recent Events ==="
kubectl get events -n microservices --sort-by='.lastTimestamp' | tail -20

echo -e "\n=== Ingress Status ==="
kubectl get ingress -n microservices
kubectl get svc -n ingress-nginx

echo -e "\n=== HPA Status ==="
kubectl get hpa -n microservices

echo -e "\n=== ConfigMap and Secrets ==="
kubectl get configmap -n microservices
kubectl get secrets -n microservices

echo -e "\n=== Node Resources ==="
kubectl top nodes 2>/dev/null || echo "Metrics server not available"

echo -e "\n=== Pod Resources ==="
kubectl top pods -n microservices 2>/dev/null || echo "Metrics server not available"

echo -e "\n=== Checking for Failed Pods ==="
kubectl get pods -n microservices | grep -E "Error|CrashLoop|ImagePull|Pending"

echo -e "\n=== Done! ==="
```

Run it:
```bash
chmod +x diagnose.sh
./diagnose.sh
```

## Quick Fixes

### Restart Everything
```bash
kubectl rollout restart deployment/backend -n microservices
kubectl rollout restart deployment/frontend -n microservices
kubectl rollout restart deployment/redis -n microservices
```

### Delete and Recreate
```bash
kubectl delete namespace microservices
kubectl apply -f k8s/
```

### Check Logs of All Pods
```bash
# Backend logs
kubectl logs -f deployment/backend -n microservices

# Frontend logs
kubectl logs -f deployment/frontend -n microservices

# Redis logs
kubectl logs -f deployment/redis -n microservices
```

## Next Steps After Fixing

1. ✅ Verify all pods are running: `kubectl get pods -n microservices`
2. ✅ Check application health: `curl http://localhost:30080/health`
3. ✅ Access in browser: `http://<EC2-PUBLIC-IP>:30080`
4. ✅ Test backend API: `curl http://<EC2-PUBLIC-IP>:30080/api/health`

## Need More Help?

Share the output of:
```bash
kubectl get pods -n microservices
kubectl get events -n microservices --sort-by='.lastTimestamp' | tail -20
kubectl logs <failing-pod-name> -n microservices
```
