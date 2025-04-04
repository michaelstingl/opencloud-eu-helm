# Getting Started with OpenCloud Helm Charts on macOS

This guide helps you set up a local Kubernetes environment on macOS for developing and testing OpenCloud using the community Helm charts.

> **Note:** These charts are intended for local development and testing purposes only, not for production deployments.

## Prerequisites

Install the required tools:

```bash
# Install Homebrew if needed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install kubectl and Helm
brew install kubectl helm
```

## Set up a local Kubernetes cluster

Choose one of these options:

### Option 1: Docker Desktop

```bash
# Install Docker Desktop
brew install --cask docker

# Start Docker Desktop, then enable Kubernetes in Preferences > Kubernetes
```

### Option 2: minikube

```bash
brew install minikube
minikube start
```

### Option 3: k3d

```bash
brew install k3d
k3d cluster create opencloud-cluster
```

Verify your cluster:
```bash
kubectl get nodes
```

## Install OpenCloud

```bash
# Create namespace
kubectl create namespace opencloud

# Install chart with default settings
helm install opencloud -n opencloud ./charts/opencloud

# Or customize with your own settings
helm install opencloud -n opencloud ./charts/opencloud \
  --set adminPassword="YourSecurePassword" \
  --set url="https://localhost:9200"
```

Wait for deployment:
```bash
kubectl get pods -n opencloud -w
```

## Access OpenCloud

Use port-forwarding to access your local instance:

```bash
kubectl port-forward -n opencloud svc/opencloud-service 9200:443
```

Open https://localhost:9200 in your browser and accept the self-signed certificate warning.

Login with:
- Username: `admin`
- Password: `admin` (or your custom password)

## Development Limitations

The current Helm charts are simplified for local development:
- Uses default insecure settings
- No resource limits/requests
- No health probes
- Single-replica deployment
- Self-signed certificates

## Clean Up

```bash
# Uninstall release
helm uninstall -n opencloud opencloud

# Delete namespace
kubectl delete namespace opencloud

# Stop cluster (if using minikube/k3d)
minikube stop
# or
k3d cluster delete opencloud-cluster
```

## Contributing

To improve these development charts:
- Fork the repository
- Submit a Pull Request
- File issues for bugs or feature requests

Future versions will include production-ready features with proper security, resource management, and high availability options.