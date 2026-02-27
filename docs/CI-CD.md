# CI/CD Pipeline Documentation

## Overview

The CI/CD pipeline is implemented using GitHub Actions and consists of three main stages:

1. Code Quality & Security Scan
2. Build and Push Docker Images
3. Deploy to Kubernetes

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Actions Workflow                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Stage 1: Code Quality & Security                           │
│  - Checkout code                                            │
│  - Setup Python                                             │
│  - Install dependencies                                     │
│  - Lint with flake8                                         │
│  - Trivy filesystem scan                                    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Stage 2: Build & Push                                      │
│  - Setup Docker Buildx                                      │
│  - Login to GHCR                                            │
│  - Build multi-stage images                                 │
│  - Push to registry                                         │
│  - Scan images with Trivy                                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Stage 3: Deploy (main branch only)                         │
│  - Setup kubectl                                            │
│  - Configure kubeconfig                                     │
│  - Update image tags                                        │
│  - Apply Kubernetes manifests                               │
│  - Wait for rollout                                         │
│  - Verify deployment                                        │
└─────────────────────────────────────────────────────────────┘
```

## Stage Details

### Stage 1: Code Quality & Security

#### Linting
- Uses flake8 for Python code quality
- Checks for syntax errors and code style
- Fails on critical issues

#### Trivy Filesystem Scan
- Scans dependencies for vulnerabilities
- Checks both backend and frontend
- Fails on CRITICAL/HIGH severity
- Outputs SARIF format for GitHub Security

### Stage 2: Build & Push

#### Docker Buildx
- Multi-platform builds
- Layer caching with GitHub Actions cache
- Optimized build performance

#### Image Tagging Strategy
- `latest` - Latest main branch build
- `<branch>` - Branch name
- `<branch>-<sha>` - Branch with commit SHA
- `<version>` - Semantic version tags

#### Registry
- GitHub Container Registry (GHCR)
- Automatic authentication with GITHUB_TOKEN
- Public or private images

#### Image Scanning
- Trivy scans built images
- Detects vulnerabilities in final images
- Fails pipeline on critical issues

### Stage 3: Deploy

#### Deployment Strategy
- Rolling updates (zero downtime)
- Only deploys from main branch
- Automatic rollback on failure

#### Deployment Steps
1. Configure kubectl with cluster credentials
2. Update image references in manifests
3. Apply manifests in order
4. Wait for rollout completion
5. Verify pod status

## Configuration

### Required Secrets

Add these in GitHub repository settings → Secrets and variables → Actions:

#### GITHUB_TOKEN
- Automatically provided by GitHub
- Used for GHCR authentication
- No configuration needed

#### KUBECONFIG
- Base64 encoded kubeconfig file
- Provides cluster access
- Generate with:
```bash
cat ~/.kube/config | base64 -w 0
```

### Environment Variables

Configured in workflow file:

```yaml
env:
  REGISTRY: ghcr.io
  BACKEND_IMAGE: ${{ github.repository }}/backend
  FRONTEND_IMAGE: ${{ github.repository }}/frontend
```

## Triggering the Pipeline

### Automatic Triggers

#### Push to main or develop
```bash
git push origin main
```

#### Pull Request to main
```bash
git checkout -b feature/new-feature
git push origin feature/new-feature
# Create PR on GitHub
```

### Manual Trigger

1. Go to GitHub repository
2. Click "Actions" tab
3. Select "CI/CD Pipeline"
4. Click "Run workflow"

## Pipeline Optimization

### Caching Strategy

#### Docker Layer Caching
```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```
- Caches Docker layers between builds
- Reduces build time by 50-70%

#### Python Dependencies
```yaml
- uses: actions/setup-python@v5
  with:
    cache: 'pip'
```
- Caches pip packages
- Faster dependency installation

### Parallel Execution

Jobs run in parallel when possible:
- Code quality checks run independently
- Image builds can run concurrently
- Deployment waits for all previous stages

## Monitoring and Debugging

### View Pipeline Status

```bash
# Using GitHub CLI
gh run list
gh run view <run-id>
gh run watch
```

### Check Logs

1. Go to Actions tab
2. Click on workflow run
3. Click on job name
4. Expand step to view logs

### Common Issues

#### Authentication Failure
```
Error: failed to authorize: failed to fetch anonymous token
```
Solution: Check GITHUB_TOKEN permissions

#### Kubeconfig Error
```
Error: error loading config file: invalid configuration
```
Solution: Verify KUBECONFIG secret is base64 encoded correctly

#### Image Pull Error
```
Error: ErrImagePull
```
Solution: Ensure images are pushed and accessible

#### Trivy Scan Failure
```
Error: vulnerabilities found
```
Solution: Update dependencies or add exceptions

## Best Practices

### Security
- Never commit secrets to repository
- Use GitHub Secrets for sensitive data
- Scan images before deployment
- Fail pipeline on critical vulnerabilities

### Performance
- Use caching for dependencies and layers
- Minimize image size with multi-stage builds
- Run tests in parallel when possible

### Reliability
- Implement health checks
- Use rolling updates
- Configure automatic rollback
- Monitor deployment status

### Maintenance
- Keep actions up to date
- Review and update dependencies
- Monitor pipeline performance
- Document changes

## Advanced Configuration

### Custom Deployment Environments

```yaml
deploy-staging:
  if: github.ref == 'refs/heads/develop'
  environment: staging
  
deploy-production:
  if: github.ref == 'refs/heads/main'
  environment: production
```

### Approval Gates

```yaml
environment:
  name: production
  url: https://microservices.example.com
```

### Notifications

```yaml
- name: Notify on failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## Troubleshooting Guide

### Pipeline Fails at Linting
- Check Python syntax errors
- Review flake8 output
- Fix code style issues

### Pipeline Fails at Security Scan
- Review Trivy output
- Update vulnerable dependencies
- Add exceptions if needed

### Pipeline Fails at Build
- Check Dockerfile syntax
- Verify base images are accessible
- Review build logs

### Pipeline Fails at Deploy
- Verify kubeconfig is correct
- Check cluster connectivity
- Review Kubernetes events

## Metrics and Reporting

### Pipeline Metrics
- Build duration
- Success rate
- Deployment frequency
- Mean time to recovery (MTTR)

### Security Metrics
- Vulnerabilities detected
- Vulnerabilities fixed
- Time to patch

### Quality Metrics
- Code quality (flake8)
- Security vulnerabilities
- Build success rate
