# Installation Guide

## AI-Powered Kubernetes SRE Assistant (MCP Architecture)

This guide provides step-by-step instructions for installing and setting up the K8s Assistant with MCP architecture.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start)
3. [Detailed Installation](#detailed-installation)
4. [Configuration](#configuration)
5. [Verification](#verification)
6. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### System Requirements

**Minimum Requirements:**
- CPU: 4 cores
- RAM: 8 GB
- Disk: 50 GB
- OS: Linux, macOS, or Windows with WSL2

**Recommended Requirements:**
- CPU: 8+ cores
- RAM: 16+ GB
- Disk: 100+ GB
- OS: Linux or macOS

### Required Software

1. **Python 3.11+**
   ```bash
   # Check Python version
   python --version
   # or
   python3 --version
   ```

2. **pip (Python package manager)**
   ```bash
   # Check pip version
   pip --version
   # or
   pip3 --version
   ```

3. **Kubernetes Cluster**
   - Local: minikube, kind, or Docker Desktop
   - Cloud: EKS, GKE, AKS, or any Kubernetes 1.28+

4. **kubectl**
   ```bash
   # Check kubectl version
   kubectl version --client
   ```

5. **Docker** (optional, for containerized deployment)
   ```bash
   # Check Docker version
   docker --version
   ```

6. **Helm 3+** (optional, for Helm deployment)
   ```bash
   # Check Helm version
   helm version
   ```

### Required API Keys

- **Anthropic API Key** (for Claude AI)
  - Sign up at: https://console.anthropic.com/
  - Or **OpenAI API Key** as alternative
  - Sign up at: https://platform.openai.com/

### Access Requirements

- Kubernetes cluster admin access
- Ability to create namespaces, deployments, and RBAC resources
- Network access to Prometheus (if using existing monitoring)

---

## Quick Start

### 1. Clone Repository

```bash
git clone https://github.com/your-org/k8s-assistant-mcp.git
cd k8s-assistant-mcp
```

### 2. Install Python Dependencies

**Option A: Install All Dependencies**
```bash
pip install -r requirements.txt
```

**Option B: Install Per Component**
```bash
# Orchestrator
pip install -r orchestrator/requirements.txt

# MCP Servers
pip install -r mcp-servers/k8s-monitor/requirements.txt
pip install -r mcp-servers/metrics/requirements.txt
pip install -r mcp-servers/detection/requirements.txt
pip install -r mcp-servers/actions/requirements.txt
pip install -r mcp-servers/knowledge-base/requirements.txt
pip install -r mcp-servers/notifications/requirements.txt
```

### 3. Set Environment Variables

```bash
# Copy example environment file
cp .env.example .env

# Edit with your values
nano .env
```

Required environment variables:
```bash
# AI Provider
ANTHROPIC_API_KEY=your-anthropic-api-key
# OR
OPENAI_API_KEY=your-openai-api-key

# Kubernetes
KUBECONFIG=~/.kube/config

# Database (if using external)
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=k8s_assistant
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your-secure-password

# Redis (if using external)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your-secure-password

# Notifications (optional)
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
EMAIL_SMTP_HOST=smtp.gmail.com
EMAIL_SMTP_PORT=587
EMAIL_USERNAME=your-email@example.com
EMAIL_PASSWORD=your-email-password
```

### 4. Configure MCP Servers

```bash
# Copy example configuration
cp config/mcp-servers.example.json config/mcp-servers.json

# Edit with your settings
nano config/mcp-servers.json
```

### 5. Start MCP Servers

```bash
# Start all MCP servers
./scripts/start-mcp-servers.sh

# Or start individually
python mcp-servers/k8s-monitor/server.py &
python mcp-servers/metrics/server.py &
python mcp-servers/detection/server.py &
python mcp-servers/actions/server.py &
python mcp-servers/knowledge-base/server.py &
python mcp-servers/notifications/server.py &
```

### 6. Start Orchestrator

```bash
python orchestrator/main.py
```

---

## Detailed Installation

### Step 1: System Preparation

#### Ubuntu/Debian

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python 3.11
sudo apt install python3.11 python3.11-venv python3-pip -y

# Install additional dependencies
sudo apt install build-essential libpq-dev -y
```

#### macOS

```bash
# Install Homebrew (if not installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Python 3.11
brew install python@3.11

# Install additional dependencies
brew install postgresql
```

#### Windows (WSL2)

```bash
# Update WSL
wsl --update

# Inside WSL, follow Ubuntu instructions
sudo apt update && sudo apt upgrade -y
sudo apt install python3.11 python3.11-venv python3-pip -y
```

### Step 2: Create Virtual Environment

```bash
# Create virtual environment
python3.11 -m venv venv

# Activate virtual environment
# On Linux/macOS:
source venv/bin/activate

# On Windows:
venv\Scripts\activate
```

### Step 3: Install Dependencies

```bash
# Upgrade pip
pip install --upgrade pip setuptools wheel

# Install all dependencies
pip install -r requirements.txt

# Verify installation
pip list | grep mcp
pip list | grep anthropic
pip list | grep kubernetes
```

### Step 4: Setup Kubernetes Access

```bash
# Verify kubectl access
kubectl cluster-info
kubectl get nodes

# Create namespace
kubectl create namespace k8s-assistant

# Apply RBAC
kubectl apply -f config/rbac.yaml
```

### Step 5: Setup Databases

#### PostgreSQL (Local)

```bash
# Install PostgreSQL
# Ubuntu/Debian:
sudo apt install postgresql postgresql-contrib -y

# macOS:
brew install postgresql@15
brew services start postgresql@15

# Create database
sudo -u postgres psql -c "CREATE DATABASE k8s_assistant;"
sudo -u postgres psql -c "CREATE USER k8s_user WITH PASSWORD 'your-password';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE k8s_assistant TO k8s_user;"

# Run migrations
cd orchestrator
alembic upgrade head
```

#### Redis (Local)

```bash
# Install Redis
# Ubuntu/Debian:
sudo apt install redis-server -y
sudo systemctl start redis-server

# macOS:
brew install redis
brew services start redis

# Test connection
redis-cli ping
```

#### ChromaDB (Vector Database)

```bash
# ChromaDB is installed via requirements.txt
# Start ChromaDB server (optional, can use embedded mode)
chroma run --path ./chroma_data
```

### Step 6: Setup Monitoring (Optional)

#### Install Prometheus

```bash
# Using Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

#### Install Loki (Optional)

```bash
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=true
```

### Step 7: Configure MCP Servers

Create `config/mcp-servers.json`:

```json
{
  "mcpServers": {
    "k8s-monitor": {
      "command": "python",
      "args": ["mcp-servers/k8s-monitor/server.py"],
      "env": {
        "KUBECONFIG": "/path/to/kubeconfig",
        "LOG_LEVEL": "info"
      }
    },
    "metrics": {
      "command": "python",
      "args": ["mcp-servers/metrics/server.py"],
      "env": {
        "PROMETHEUS_URL": "http://prometheus-server:9090",
        "LOG_LEVEL": "info"
      }
    },
    "detection": {
      "command": "python",
      "args": ["mcp-servers/detection/server.py"],
      "env": {
        "LOG_LEVEL": "info"
      }
    },
    "actions": {
      "command": "python",
      "args": ["mcp-servers/actions/server.py"],
      "env": {
        "KUBECONFIG": "/path/to/kubeconfig",
        "DRY_RUN": "false",
        "LOG_LEVEL": "info"
      }
    },
    "knowledge-base": {
      "command": "python",
      "args": ["mcp-servers/knowledge-base/server.py"],
      "env": {
        "DATABASE_URL": "postgresql://k8s_user:password@localhost/k8s_assistant",
        "VECTOR_DB_URL": "http://localhost:8000",
        "LOG_LEVEL": "info"
      }
    },
    "notifications": {
      "command": "python",
      "args": ["mcp-servers/notifications/server.py"],
      "env": {
        "SLACK_WEBHOOK_URL": "https://hooks.slack.com/services/...",
        "LOG_LEVEL": "info"
      }
    }
  }
}
```

### Step 8: Start Services

#### Development Mode

```bash
# Terminal 1: Start MCP Servers
./scripts/start-mcp-servers.sh

# Terminal 2: Start Orchestrator
python orchestrator/main.py

# Terminal 3: Monitor logs
tail -f logs/orchestrator.log
```

#### Production Mode (Docker)

```bash
# Build images
docker-compose build

# Start services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f
```

#### Production Mode (Kubernetes)

```bash
# Create secrets
kubectl create secret generic k8s-assistant-secrets \
  --from-literal=anthropic-api-key=$ANTHROPIC_API_KEY \
  --from-literal=postgres-password=$POSTGRES_PASSWORD \
  --from-literal=redis-password=$REDIS_PASSWORD \
  -n k8s-assistant

# Deploy with Helm
helm install k8s-assistant ./charts/k8s-assistant \
  --namespace k8s-assistant \
  --values config/values-prod.yaml

# Check deployment
kubectl get pods -n k8s-assistant
```

---

## Configuration

### Minimal Configuration

```yaml
# config/config.yaml
global:
  environment: development
  log_level: info

ai:
  provider: anthropic
  model: claude-3-5-sonnet-20241022
  api_key: ${ANTHROPIC_API_KEY}

actions:
  auto_fix:
    enabled: true
    confidence_threshold: 0.90
```

### Production Configuration

See [Configuration Guide](docs/configuration/README.md) for detailed configuration options.

---

## Verification

### 1. Check MCP Servers

```bash
# List MCP servers
k8s-assistant mcp list-servers

# Check server health
k8s-assistant mcp health-check --all

# Expected output:
# ✅ k8s-monitor: healthy
# ✅ metrics: healthy
# ✅ detection: healthy
# ✅ actions: healthy
# ✅ knowledge-base: healthy
# ✅ notifications: healthy
```

### 2. Test MCP Tools

```bash
# Test K8s Monitor
k8s-assistant mcp test-tool \
  --server k8s-monitor \
  --tool watch_pods \
  --args '{"namespace": "default"}'

# Test Metrics
k8s-assistant mcp test-tool \
  --server metrics \
  --tool query_metrics \
  --args '{"query": "up"}'
```

### 3. Check API Health

```bash
# Port forward (if running in Kubernetes)
kubectl port-forward -n k8s-assistant svc/k8s-assistant-api 8080:8080

# Check health endpoint
curl http://localhost:8080/health

# Expected response:
# {
#   "status": "healthy",
#   "components": {
#     "database": "healthy",
#     "redis": "healthy",
#     "llm": "healthy",
#     "mcp_servers": {
#       "k8s-monitor": "healthy",
#       "metrics": "healthy",
#       "detection": "healthy",
#       "actions": "healthy",
#       "knowledge-base": "healthy",
#       "notifications": "healthy"
#     }
#   }
# }
```

### 4. Test End-to-End Workflow

```bash
# Create a test incident
kubectl run test-pod --image=nginx --limits=memory=10Mi -n default

# Wait for OOMKill
kubectl wait --for=condition=Ready pod/test-pod -n default --timeout=60s || true

# Check if incident was detected
curl http://localhost:8080/api/v1/incidents | jq

# Clean up
kubectl delete pod test-pod -n default
```

---

## Troubleshooting

### Common Issues

#### 1. MCP Server Not Starting

```bash
# Check Python version
python --version

# Check dependencies
pip list | grep mcp

# Check logs
tail -f logs/mcp-k8s-monitor.log

# Reinstall dependencies
pip install --force-reinstall -r mcp-servers/k8s-monitor/requirements.txt
```

#### 2. Database Connection Failed

```bash
# Check PostgreSQL status
sudo systemctl status postgresql

# Test connection
psql -h localhost -U k8s_user -d k8s_assistant

# Check connection string
echo $DATABASE_URL
```

#### 3. Kubernetes Access Denied

```bash
# Check kubeconfig
kubectl config view

# Check RBAC
kubectl auth can-i list pods --as=system:serviceaccount:k8s-assistant:k8s-assistant

# Reapply RBAC
kubectl apply -f config/rbac.yaml
```

#### 4. AI API Errors

```bash
# Check API key
echo $ANTHROPIC_API_KEY

# Test API connectivity
curl -H "x-api-key: $ANTHROPIC_API_KEY" \
  https://api.anthropic.com/v1/messages \
  -X POST \
  -d '{"model":"claude-3-5-sonnet-20241022","max_tokens":100,"messages":[{"role":"user","content":"test"}]}'
```

### Getting Help

- **Documentation**: [docs/](docs/)
- **Troubleshooting Guide**: [docs/troubleshooting/README.md](docs/troubleshooting/README.md)
- **GitHub Issues**: https://github.com/your-org/k8s-assistant-mcp/issues
- **Community**: Slack #k8s-assistant

---

## Next Steps

After successful installation:

1. **Configure Auto-Fix**: See [Configuration Guide](docs/configuration/README.md)
2. **Setup Notifications**: Configure Slack/Email alerts
3. **Review Examples**: Check [Examples](docs/examples/README.md)
4. **Monitor System**: Setup Grafana dashboards
5. **Production Deployment**: Follow [Deployment Guide](DEPLOYMENT.md)

---

## Uninstallation

### Local Installation

```bash
# Stop services
./scripts/stop-mcp-servers.sh

# Deactivate virtual environment
deactivate

# Remove virtual environment
rm -rf venv

# Remove data (optional)
rm -rf chroma_data logs
```

### Kubernetes Installation

```bash
# Uninstall Helm release
helm uninstall k8s-assistant -n k8s-assistant

# Delete namespace
kubectl delete namespace k8s-assistant

# Delete CRDs (if any)
kubectl delete crd -l app=k8s-assistant
```

---

## Support

For installation issues:
- Email: support@example.com
- Slack: #k8s-assistant-help
- GitHub: Open an issue with `installation` label