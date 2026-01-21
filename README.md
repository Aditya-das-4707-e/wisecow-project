# Wisecow Kubernetes Deployment Project

This project demonstrates the containerization and Kubernetes deployment of the Wisecow application - a fun web server that displays random cow wisdom quotes using `fortune` and `cowsay`.

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Implementation Steps](#implementation-steps)
- [Kubernetes Deployment](#kubernetes-deployment)
- [GitHub Actions CI/CD](#github-actions-cicd)
- [TLS/SSL Configuration](#tlsssl-configuration)
- [Deployment Commands](#deployment-commands)
- [Troubleshooting](#troubleshooting)

## Overview

This project fulfills the following requirements:
1. ✅ Dockerized the Wisecow application
2. ✅ Created Kubernetes manifests for deployment
3. ✅ Implemented GitHub Actions for automated CI/CD
4. ❌ Configured TLS/SSL for secure communication (Its not work in my personal pc)

## Prerequisites

Before deploying this project, ensure you have:

- **Docker** installed (for building images)
- **Kubernetes cluster** (minikube, kind, or cloud-based cluster like EKS/GKE/AKS)
- **kubectl** configured to access your cluster
- **GitHub account** with repository secrets configured
- **Container registry** access (Docker Hub, GitHub Container Registry, etc.)

### Required Tools Installation

```bash
# Install Docker
sudo apt-get update
sudo apt-get install docker.io -y

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install minikube (for local testing)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

## Project Structure

```
wisecow-project/
├── .github/
│   └── workflows/
│       └── docker-image.yml          # GitHub Actions workflow
├── wisecow-k8s/
│   ├── deployment.yaml               # Kubernetes Deployment manifest
│   ├── service.yaml                  # Kubernetes Service manifest
│   ├── ingress.yaml                  # Kubernetes Ingress for TLS (optional)
│   └── tls-secret.yaml               # TLS certificate secret (optional)
├── wisecow.sh                        # Main application script
├── Dockerfile                        # Docker configuration
├── tls.crt                           # TLS certificate
├── tls.key                           # TLS private key
├── LICENSE
└── README.md
```

## Implementation Steps

### Step 1: Dockerfile Creation

The Dockerfile packages the Wisecow application with all dependencies:

```dockerfile
FROM ubuntu:latest

# Install required packages
RUN apt-get update && \
    apt-get install -y fortune-mod cowsay netcat-openbsd && \
    rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy application files
COPY wisecow.sh .

# Make script executable
RUN chmod +x wisecow.sh

# Expose port
EXPOSE 4499

# Run the application
CMD ["./wisecow.sh"]
```

### Step 2: Building Docker Image Locally

```bash
# Build the Docker image
docker build -t wisecow-app:latest .

# Test the image locally
docker run -p 4499:4499 wisecow-app:latest

# Access the application
curl http://localhost:4499
```

### Step 3: Push to Container Registry

```bash
# Tag the image for your registry
docker tag wisecow-app:latest <your-registry>/wisecow-app:latest

# Login to your container registry
docker login

# Push the image
docker push <your-registry>/wisecow-app:latest
```

## Kubernetes Deployment

### Deployment Manifest (`wisecow-k8s/deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wisecow-deployment
  labels:
    app: wisecow
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wisecow
  template:
    metadata:
      labels:
        app: wisecow
    spec:
      containers:
      - name: wisecow
        image: <your-registry>/wisecow-app:latest
        ports:
        - containerPort: 4499
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### Service Manifest (`wisecow-k8s/service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wisecow-service
spec:
  selector:
    app: wisecow
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 4499
```

### Deploying to Kubernetes

```bash
# Apply all Kubernetes manifests
kubectl apply -f wisecow-k8s/

# Or apply individually
kubectl apply -f wisecow-k8s/deployment.yaml
kubectl apply -f wisecow-k8s/service.yaml

# Verify deployment
kubectl get deployments
kubectl get pods
kubectl get services

# Check pod logs
kubectl logs -l app=wisecow

# Get service external IP (for LoadBalancer)
kubectl get svc wisecow-service
```

## GitHub Actions CI/CD

### Workflow Configuration (`.github/workflows/docker-image.yml`)

The GitHub Actions workflow automatically:
1. Triggers on push to main branch
2. Builds Docker image
3. Pushes to container registry
4. Updates Kubernetes deployment (optional)

```yaml
name: CI-CD Pipeline

on:
  push:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and Push Docker Image
      uses: docker/build-push-action@v4
      with:
        context: wisecow-k8s
        push: true
        tags: ${{ secrets.DOCKER_USERNAME }}/wisecow:latest
```

### Setting Up GitHub Secrets

1. Go to your GitHub repository
2. Navigate to: **Settings → Secrets and variables → Actions**
3. Add the following secrets:
   - `DOCKER_USERNAME`: Your Docker Hub username
   - `DOCKER_PASSWORD`: Your Docker Hub password/token
   - `KUBE_CONFIG_DATA`: Base64 encoded kubeconfig (optional, for auto-deployment)

```bash
# Encode kubeconfig for GitHub secret
cat ~/.kube/config | base64 -w 0
```

## TLS/SSL Configuration

### Option 1: Using Kubernetes Secrets

```bash
# Create TLS secret from certificate files
kubectl create secret tls wisecow-tls \
  --cert=tls.crt \
  --key=tls.key
```

### Option 2: Ingress with TLS (`wisecow-k8s/ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wisecow-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - wisecow.example.com
    secretName: wisecow-tls
  rules:
  - host: wisecow.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wisecow-service
            port:
              number: 80
```

### Generating Self-Signed Certificates (for testing)

```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=wisecow.local/O=wisecow"

# Create Kubernetes secret
kubectl create secret tls wisecow-tls \
  --cert=tls.crt \
  --key=tls.key
```

## Deployment Commands

### Complete Deployment Workflow

```bash
# 1. Build and push Docker image
docker build -t <your-username>/wisecow-app:latest .
docker push <your-username>/wisecow-app:latest

# 2. Deploy to Kubernetes
kubectl apply -f wisecow-k8s/

# 3. Verify deployment
kubectl get all -l app=wisecow

# 4. Access the application
kubectl port-forward service/wisecow-service 8080:80

# Visit: http://localhost:8080
```

### Updating the Deployment

```bash
# Update image in deployment
kubectl set image deployment/wisecow-deployment \
  wisecow=<your-username>/wisecow-app:new-tag

# Or edit the deployment directly
kubectl edit deployment wisecow-deployment

# Rollback if needed
kubectl rollout undo deployment/wisecow-deployment
```

### Scaling the Application

```bash
# Scale up replicas
kubectl scale deployment wisecow-deployment --replicas=5

# Enable autoscaling
kubectl autoscale deployment wisecow-deployment \
  --min=2 --max=10 --cpu-percent=80
```

## Troubleshooting

### Common Issues and Solutions

**Pods not starting:**
```bash
# Check pod status
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

**Image pull errors:**
```bash
# Verify image exists
docker pull <your-registry>/wisecow-app:latest

# Check image pull secrets
kubectl get secrets
```

**Service not accessible:**
```bash
# Check service endpoints
kubectl get endpoints wisecow-service

# Test from within cluster
kubectl run test-pod --rm -it --image=busybox -- sh
wget -O- http://wisecow-service
```

**TLS certificate issues:**
```bash
# Verify TLS secret
kubectl describe secret wisecow-tls

# Test certificate
openssl s_client -connect wisecow.example.com:443
```

### Useful Commands

```bash
# View all resources
kubectl get all -n default

# Delete all wisecow resources
kubectl delete -f wisecow-k8s/

# Watch pod status
kubectl get pods -w

# Execute command in pod
kubectl exec -it <pod-name> -- /bin/bash

# View resource usage
kubectl top pods
kubectl top nodes
```

## Testing the Application

### Local Testing
```bash
# Test with curl
curl http://localhost:4499

# Test with browser
xdg-open http://localhost:4499
```

### Testing in Kubernetes
```bash
# Port forward to local machine
kubectl port-forward service/wisecow-service 8080:80

# Test the forwarded port
curl http://localhost:8080
```

## Monitoring and Logs

```bash
# Stream logs from all pods
kubectl logs -f -l app=wisecow

# View logs from specific pod
kubectl logs -f <pod-name>

# View previous container logs
kubectl logs <pod-name> --previous
```

## Clean Up

```bash
# Delete all Kubernetes resources
kubectl delete -f wisecow-k8s/

# Or delete individually
kubectl delete deployment wisecow-deployment
kubectl delete service wisecow-service
kubectl delete ingress wisecow-ingress
kubectl delete secret wisecow-tls

# Delete Docker images
docker rmi <your-registry>/wisecow-app:latest
```

## Contributing

This project is maintained as part of a Kubernetes deployment exercise. Feel free to suggest improvements or report issues.

## License

This project is licensed under the Apache-2.0 License - see the LICENSE file for details.

## Acknowledgments

- Original Wisecow application: [nyrahul/wisecow](https://github.com/nyrahul/wisecow)
- Fortune-mod and Cowsay utilities
- Kubernetes community for excellent documentation
