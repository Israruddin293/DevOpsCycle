# Kubeadm + GitHub Actions - Ultra Quick Start

## 5-Minute Setup for Kubeadm Cluster

### 1. Get Kubeconfig (On EC2)

```bash
cat ~/.kube/config > kubeconfig-github.yaml
```

### 2. Change Server IP

```bash
# Replace with your EC2 PUBLIC IP
sed -i 's/127.0.0.1/<EC2-PUBLIC-IP>/g' kubeconfig-github.yaml
```

### 3. Open Port 6443

```bash
# In AWS Console: EC2 → Security Groups → Add Inbound Rule
# Type: Custom TCP, Port: 6443, Source: 0.0.0.0/0
```

### 4. Encode for GitHub

```bash
cat kubeconfig-github.yaml | base64 -w 0 to get encoded file
# Copy the output
```

### 5. Add to GitHub

```
GitHub Repo → Settings → Secrets → Actions → New secret
Name: KUBECONFIG
Value: <paste base64 string>
```

### 6. Push Code

```bash
git push origin main
```

### 7. Access App

```
http://<EC2-PUBLIC-IP>:30080
```

## Troubleshooting

### Can't connect to cluster?

```bash
# On EC2, check API server
sudo netstat -tlnp | grep 6443

# Should show: 0.0.0.0:6443
# If shows 127.0.0.1:6443, edit:
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
# Add: --bind-address=0.0.0.0
```

### Certificate error?

```bash
# On EC2
sudo kubeadm init phase certs apiserver --apiserver-cert-extra-sans=<EC2-PUBLIC-IP>
sudo systemctl restart kubelet
```

### ImagePullBackOff?

```bash
# Make images public in GitHub Packages
# OR create secret:
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<username> \
  --docker-password=<token> \
  -n microservices
```

## Done! 🎉

See [KUBEADM-GITHUB-ACTIONS.md](docs/KUBEADM-GITHUB-ACTIONS.md) for detailed guide.
