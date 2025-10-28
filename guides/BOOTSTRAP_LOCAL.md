# Local Bootstrap Guide

Complete step-by-step guide to run GigVault locally on your development machine.

## Prerequisites

### Required Tools

1. **Docker** (>= 24.0)
   ```bash
   docker --version
   ```

2. **kind** (Kubernetes in Docker)
   ```bash
   # macOS
   brew install kind
   
   # Linux
   curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
   chmod +x ./kind
   sudo mv ./kind /usr/local/bin/kind
   ```

3. **kubectl** (>= 1.29)
   ```bash
   # macOS
   brew install kubectl
   
   # Linux
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
   ```

4. **Helm** (>= 3.13)
   ```bash
   # macOS
   brew install helm
   
   # Linux
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

5. **Go** (>= 1.23)
   ```bash
   # Download from https://go.dev/dl/
   go version
   ```

### System Requirements

- CPU: 4 cores minimum
- RAM: 8GB minimum (16GB recommended)
- Disk: 20GB free space

## Step-by-Step Setup

### 1. Clone the Infrastructure Repository

```bash
# Clone infra repo (contains deployment scripts)
git clone https://github.com/gigvault/infra.git
cd infra
```

### 2. Create kind Cluster

```bash
make kind-up
```

This will:
- Create a kind cluster named "gigvault"
- Configure port mappings for services
- Install PostgreSQL
- Create the `gigvault` namespace

**Expected Output:**
```
Creating kind cluster...
✓ Creating cluster "gigvault"
✓ Ensuring node image
✓ Preparing nodes
✓ Writing configuration
✓ Starting control-plane
✓ Installing CNI
✓ Installing StorageClass
✓ Set kubectl context to "kind-gigvault"

Kind cluster 'gigvault' is ready!
```

**Verify:**
```bash
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
```

### 3. Build All Services

```bash
make build-all
```

This script will:
- Build Docker images for all services
- Load images into the kind cluster
- Take approximately 10-15 minutes

**Services Built:**
- shared (library)
- rootca
- ca
- ra
- keymgr
- enroll
- ocsp
- crl
- policy
- auth
- audit
- notify
- cli
- operator

**Expected Output:**
```
Building ca...
✓ ca built and loaded
Building ra...
✓ ra built and loaded
...
✅ All services built successfully!
```

### 4. Deploy Services

```bash
make deploy-local
```

This will:
- Deploy all services using Helm
- Wait for pods to be ready
- Configure service endpoints

**Expected Output:**
```
Deploying ca...
✓ ca deployed
Deploying ra...
✓ ra deployed
...
✅ All services deployed successfully!
```

**Verify Deployment:**
```bash
# Check all pods are running
kubectl get pods -n gigvault

# Expected output:
NAME                        READY   STATUS    RESTARTS   AGE
ca-7f8d9c5b6d-x4h2k        1/1     Running   0          2m
ra-6d5f8c9b7d-p3k1j        1/1     Running   0          2m
postgresql-0               1/1     Running   0          5m
...
```

### 5. Run Smoke Tests

```bash
make smoke-test
```

This verifies:
- All pods are running
- Health endpoints respond
- Services can communicate

**Expected Output:**
```
Checking pod status...
✓ All pods running

Checking service endpoints...
✓ All services reachable

Testing CA service...
✓ CA service is healthy

Testing RA service...
✓ RA service is healthy

✅ Smoke tests passed!
```

## Accessing Services

### Port Forwarding

Access services via port forwarding:

```bash
# CA Service
kubectl port-forward -n gigvault svc/ca 8080:8080

# RA Service
kubectl port-forward -n gigvault svc/ra 8081:8081

# Auth Service
kubectl port-forward -n gigvault svc/auth 8087:8087
```

### Testing API Endpoints

```bash
# Test CA health
curl http://localhost:8080/health

# List certificates
curl http://localhost:8080/api/v1/certificates

# Test RA
curl http://localhost:8081/health
```

## Using the CLI

### Install CLI

```bash
cd ../cli
make build
sudo make install
```

### CLI Commands

```bash
# Generate a key pair
gigvault key generate mykey

# Create a CSR
gigvault csr create example.com

# Create a self-signed cert
gigvault cert create example.com

# Show configuration
gigvault config show
```

## Database Access

### Connect to PostgreSQL

```bash
# Port forward PostgreSQL
kubectl port-forward -n gigvault svc/postgresql 5432:5432

# Connect with psql
PGPASSWORD=changeme psql -h localhost -U gigvault -d gigvault
```

### Run Migrations

```bash
# For CA service
cd ../ca
export DATABASE_URL="postgres://gigvault:changeme@localhost:5432/gigvault_ca?sslmode=disable"
make migrate
```

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n gigvault

# Describe problematic pod
kubectl describe pod <pod-name> -n gigvault

# Check logs
kubectl logs <pod-name> -n gigvault
```

### Database Connection Issues

```bash
# Verify PostgreSQL is running
kubectl get pods -n gigvault -l app=postgresql

# Check PostgreSQL logs
kubectl logs postgresql-0 -n gigvault

# Test connection
kubectl run -it --rm debug --image=postgres:16-alpine --restart=Never -n gigvault -- \
  psql -h postgresql -U gigvault -d gigvault
```

### Image Pull Errors

If you see `ImagePullBackOff`:

```bash
# Rebuild and reload image
cd ../<service>
docker build -t gigvault/<service>:local .
kind load docker-image gigvault/<service>:local --name gigvault

# Restart deployment
kubectl rollout restart deployment/<service> -n gigvault
```

### Port Conflicts

If ports 8080-8089 are in use:

```bash
# Find processes using ports
lsof -i :8080

# Kill process if needed
kill -9 <PID>
```

## Cleanup

### Stop Services (Keep Cluster)

```bash
helm uninstall ca ra keymgr enroll ocsp crl policy auth audit notify -n gigvault
```

### Full Cleanup

```bash
make clean
```

This will:
- Delete the kind cluster
- Remove Docker images
- Clean up all resources

## Next Steps

- [Architecture Documentation](../architecture/ARCHITECTURE.md)
- [Operations Manual](OPSEC.md)
- [API Documentation](../api/CA_API.md)
- [Production Deployment](DEPLOYMENT.md)

## Common Development Workflows

### Rebuild and Redeploy a Single Service

```bash
# Rebuild CA service
cd ../ca
make docker
kind load docker-image gigvault/ca:local --name gigvault
kubectl rollout restart deployment/ca -n gigvault

# Watch deployment
kubectl rollout status deployment/ca -n gigvault
```

### View Logs

```bash
# Tail logs for CA service
kubectl logs -f deployment/ca -n gigvault

# View last 100 lines
kubectl logs --tail=100 deployment/ca -n gigvault

# Follow logs from all pods
kubectl logs -f -l app=ca -n gigvault
```

### Debug Inside a Pod

```bash
kubectl exec -it deployment/ca -n gigvault -- /bin/sh
```

## Support

For issues or questions:
- GitHub Issues: https://github.com/gigvault/infra/issues
- Documentation: https://docs.gigvault.io
- Community Slack: https://gigvault.slack.com

