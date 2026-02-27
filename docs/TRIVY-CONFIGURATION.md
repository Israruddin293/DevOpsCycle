# Trivy Security Scanning Configuration

## Overview

Trivy is configured in the CI/CD pipeline to scan for vulnerabilities in both the filesystem (dependencies) and Docker images. The current configuration is set to **report vulnerabilities without blocking the pipeline**.

## Current Configuration

### Filesystem Scanning

```yaml
- name: Run Trivy vulnerability scanner (Backend)
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    scan-ref: './backend'
    format: 'table'
    severity: 'CRITICAL,HIGH'
    exit-code: '0'
  continue-on-error: true
```

### Image Scanning

```yaml
- name: Scan Backend image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.BACKEND_IMAGE }}:latest
    format: 'table'
    severity: 'CRITICAL,HIGH'
    exit-code: '0'
  continue-on-error: true
```

## Configuration Options

### Exit Code Behavior

- `exit-code: '0'` - **Current setting**: Reports vulnerabilities but doesn't fail the pipeline
- `exit-code: '1'` - **Strict mode**: Fails the pipeline if CRITICAL/HIGH vulnerabilities are found

### Severity Levels

- `CRITICAL` - Most severe vulnerabilities
- `HIGH` - High severity vulnerabilities
- `MEDIUM` - Medium severity vulnerabilities
- `LOW` - Low severity vulnerabilities
- `UNKNOWN` - Unknown severity

Current setting: `CRITICAL,HIGH`

### Output Formats

- `table` - **Current setting**: Human-readable table format in logs
- `sarif` - SARIF format for GitHub Security tab integration
- `json` - JSON format for programmatic processing

## Switching to Strict Mode

To make the pipeline fail on vulnerabilities (recommended for production):

1. Change `exit-code: '0'` to `exit-code: '1'`
2. Remove `continue-on-error: true`
3. Ensure all dependencies are up to date

```yaml
- name: Run Trivy vulnerability scanner (Backend)
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    scan-ref: './backend'
    format: 'sarif'
    output: 'trivy-backend-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

## Dependency Management

### Current Versions (Updated for Security)

#### Backend
```
Flask==3.0.3
redis==5.0.8
gunicorn==23.0.0
werkzeug==3.0.4
```

#### Frontend
```
Flask==3.0.3
requests==2.32.3
gunicorn==23.0.0
werkzeug==3.0.4
urllib3==2.2.3
certifi==2024.8.30
```

### Updating Dependencies

```bash
# Check for outdated packages
pip list --outdated

# Update specific package
pip install --upgrade <package-name>

# Update requirements file
pip freeze > requirements.txt
```

## Handling Vulnerabilities

### 1. Review Scan Results

Check the pipeline logs for Trivy output:
```bash
gh run view <run-id> --log
```

### 2. Update Dependencies

Update vulnerable packages to patched versions:
```bash
cd backend
pip install --upgrade Flask redis gunicorn
pip freeze > requirements.txt
```

### 3. Test Locally

```bash
# Install Trivy locally
# macOS
brew install trivy

# Linux
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# Scan locally
trivy fs ./backend
trivy fs ./frontend
trivy image backend:latest
```

### 4. Exceptions and Suppressions

If a vulnerability cannot be fixed immediately, create a `.trivyignore` file:

```bash
# .trivyignore
# CVE-2024-XXXXX - False positive, not applicable to our use case
CVE-2024-XXXXX

# CVE-2024-YYYYY - Waiting for upstream fix, tracked in issue #123
CVE-2024-YYYYY
```

## Best Practices

### Regular Updates
- Update dependencies monthly
- Monitor security advisories
- Use automated dependency updates (Dependabot)

### Monitoring
- Review Trivy reports in every pipeline run
- Track vulnerability trends
- Document exceptions

### Production Readiness
- Enable strict mode (`exit-code: '1'`) before production
- Ensure all CRITICAL vulnerabilities are resolved
- Keep HIGH vulnerabilities to a minimum

## GitHub Security Integration

To enable SARIF upload to GitHub Security tab:

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    scan-ref: './backend'
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload Trivy results to GitHub Security
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

## Troubleshooting

### Pipeline Fails on Trivy Scan

**Issue**: Pipeline fails with exit code 1

**Solution**:
1. Check which vulnerabilities were found
2. Update dependencies
3. Or temporarily set `exit-code: '0'`

### False Positives

**Issue**: Trivy reports vulnerabilities that don't apply

**Solution**:
1. Add to `.trivyignore`
2. Document the reason
3. Review periodically

### Scan Takes Too Long

**Issue**: Trivy scan is slow

**Solution**:
1. Use `--skip-dirs` to exclude unnecessary directories
2. Cache Trivy database
3. Scan only changed files in PRs

## Additional Resources

- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Trivy GitHub Action](https://github.com/aquasecurity/trivy-action)
- [CVE Database](https://cve.mitre.org/)
- [National Vulnerability Database](https://nvd.nist.gov/)
