# Production-Ready Microservices with Kubernetes

Enterprise-grade microservices architecture with containerization, Kubernetes orchestration, CI/CD pipeline, and comprehensive security practices.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     NGINX Ingress Controller                 │
│                  (microservices.local)                       │
└────────────┬────────────────────────────┬───────────────────┘
             │                            │
             │ /                          │ /api
             ▼                            ▼
    ┌────────────────┐          ┌────────────────┐
    │   Frontend     │          │    Backend     │
    │   (Flask)      │─────────▶│    (Flask)     │
    │   Port: 8080   │          │   Port: 5000   │
    └────────────────┘          └────────┬───────┘
                                         │
                                         ▼
                                ┌────────────────┐
                                │     Redis      │
                                │   Port: 6379   │
                                └────────────────┘
```

## Features

- Multi-stage Docker builds for optimized image sizes
- Non-root containers with security contexts
- Kubernetes deployments with health checks
- Horizontal Pod Autoscaling (HPA)
- Network policies for traffic restriction
- Pod Disruption Budgets for high availability
- ConfigMaps and Secrets for configuration
- NGINX Ingress for routing
- Complete CI/CD pipeline with GitHub Actions
- Security scanning with Trivy
- Code quality analysis with SonarQube

## Project Structure

```
project/
├── frontend/
│   ├── app.py
│   ├── templates/
│   │   └── index.html
│   ├── Dockerfile
│   ├── .dockerignore
│   └── requirements.txt
├── backend/
│   ├── app.py
│   ├── Dockerfile
│   ├── .dockerignore
│   └── requirements.txt
├── k8s/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── backend-deployment.yaml
│   ├── frontend-deployment.yaml
│   ├── redis-deployment.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── network-policy.yaml
├── .github/
│   └── workflows/
│       └── ci-cd.yaml
└── README.md
```

## Prerequisites

- Docker Desktop or Docker Engine (20.10+)
- Kubernetes cluster (minikube, kind, or cloud provider)
- kubectl (1.25+)
- Python 3.11+
- Git

## Local Development

### 1. Build Docker Images

```bash
# Build backend
docker build -t backend:latest ./backend

# Build frontend
docker build -t frontend:latest ./frontend
```

### 2. Run Locally with Docker

```bash
# Create network
docker network create microservices-net

# Run Redis
docker run -d --name redis --network microservices-net \
  -e REDIS_PASSWORD=redis-secure-password-123 \
  redis:7-alpine redis-server --requirepass redis-secure-password-123

# Run Backend
docker run -d --name backend --network microservices-net \
  -e REDIS_HOST=redis \
  -e REDIS_PORT=6379 \
  -e REDIS_PASSWORD=redis-secure-password-123 \
  -p 5000:5000 backend:latest

# Run Frontend
docker run -d --name frontend --network microservices-net \
  -e BACKEND_URL=http://backend:5000 \
  -p 8080:8080 frontend:latest
```

Access the application at http://localhost:8080

### 3. Test Endpoints

```bash
# Frontend health
curl http://localhost:8080/health

# Backend health
curl http://localhost:5000/health

# Backend API
curl http://localhost:5000/api/data
```

## Kubernetes Deployment

### 1. Install NGINX Ingress Controller

```bash
# For minikube
minikube addons enable ingress

# For kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# For cloud providers (AWS, GCP, Azure)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### 2. Deploy Application

```bash
# Apply all manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/redis-deployment.yaml
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/frontend-deployment.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml
kubectl apply -f k8s/pdb.yaml
kubectl apply -f k8s/network-policy.yaml
```

### 3. Configure Local DNS

Add to /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows):

```
127.0.0.1 microservices.local
```

For minikube, get the IP:
```bash
minikube ip
# Add the output IP instead of 127.0.0.1
```

### 4. Verify Deployment

```bash
# Check pods
kubectl get pods -n microservices

# Check services
kubectl get svc -n microservices

# Check ingress
kubectl get ingress -n microservices

# View logs
kubectl logs -f deployment/backend -n microservices
kubectl logs -f deployment/frontend -n microservices
```

### 5. Access Application

Open browser: http://microservices.local

## CI/CD Pipeline

### GitHub Actions Workflow

The pipeline includes:

1. Code Quality & Security Scan
   - Linting with flake8
   - Trivy filesystem vulnerability scanning

2. Build and Push Docker Images
   - Multi-stage builds
   - Push to GitHub Container Registry (GHCR)
   - Docker layer caching for faster builds
   - Trivy image scanning

3. Deploy to Kubernetes
   - Automatic deployment on main branch
   - Rolling updates
   - Deployment verification

### Required GitHub Secrets

Configure these in GitHub repository settings:

```
GITHUB_TOKEN          # Automatically provided
KUBECONFIG            # Base64 encoded kubeconfig file
```

### Generate KUBECONFIG Secret

```bash
# Encode your kubeconfig
cat ~/.kube/config | base64 -w 0

# Add the output to GitHub Secrets as KUBECONFIG
```

### Trigger Pipeline

```bash
git add .
git commit -m "Deploy microservices"
git push origin main
```

## Security Best Practices

### Container Security

- Non-root users in all containers
- Read-only root filesystem where possible
- Minimal base images (Alpine Linux)
- Multi-stage builds to reduce attack surface
- No hardcoded secrets
- Security scanning with Trivy

### Kubernetes Security

- SecurityContext with runAsNonRoot
- Resource limits to prevent resource exhaustion
- Network policies to restrict traffic
- Secrets for sensitive data
- RBAC (Role-Based Access Control)
- Pod Security Standards

### Image Optimization

- Multi-stage builds reduce image size by 60-70%
- .dockerignore to exclude unnecessary files
- Layer caching for faster builds
- Minimal dependencies

## Monitoring and Scaling

### Horizontal Pod Autoscaling

HPA automatically scales based on CPU/memory:

```bash
# View HPA status
kubectl get hpa -n microservices

# Describe HPA
kubectl describe hpa backend-hpa -n microservices
```

### Resource Monitoring

```bash
# View resource usage
kubectl top pods -n microservices
kubectl top nodes

# View events
kubectl get events -n microservices --sort-by='.lastTimestamp'
```

## Troubleshooting

### Pod Issues

```bash
# Check pod status
kubectl get pods -n microservices

# Describe pod
kubectl describe pod <pod-name> -n microservices

# View logs
kubectl logs <pod-name> -n microservices

# Execute into pod
kubectl exec -it <pod-name> -n microservices -- /bin/sh
```

### Service Issues

```bash
# Check service endpoints
kubectl get endpoints -n microservices

# Test service connectivity
kubectl run test-pod --rm -it --image=busybox -n microservices -- sh
wget -O- http://backend:5000/health
```

### Ingress Issues

```bash
# Check ingress
kubectl describe ingress microservices-ingress -n microservices

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

## Advanced Features

### Network Policies

Restrict traffic between pods:
- Frontend can only access Backend
- Backend can only access Redis
- Redis accepts connections only from Backend

```bash
kubectl get networkpolicies -n microservices
```

### Pod Disruption Budgets

Ensure minimum availability during voluntary disruptions:

```bash
kubectl get pdb -n microservices
```

### Persistent Storage (Optional)

For Redis data persistence, create a PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: microservices
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Performance Optimization

- Gunicorn with multiple workers
- Redis for caching and session storage
- Resource requests and limits
- Horizontal Pod Autoscaling
- Docker layer caching in CI/CD

## Clean Up

```bash
# Delete all resources
kubectl delete namespace microservices

# Delete ingress controller (if needed)
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Stop local Docker containers
docker stop frontend backend redis
docker rm frontend backend redis
docker network rm microservices-net
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## License

MIT License - see LICENSE file for details

## Support

For issues and questions:
- Create an issue in GitHub
- Check existing documentation
- Review Kubernetes logs

## References

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [GitHub Actions](https://docs.github.com/en/actions)
