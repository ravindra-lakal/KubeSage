# Deployment Guide

## AI-Powered Kubernetes SRE Assistant

This document provides comprehensive deployment instructions for the AI-Powered Kubernetes SRE Assistant.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start)
3. [Production Deployment](#production-deployment)
4. [Configuration](#configuration)
5. [Monitoring Setup](#monitoring-setup)
6. [Scaling](#scaling)
7. [Backup & Recovery](#backup--recovery)
8. [Troubleshooting](#troubleshooting)
9. [Upgrade Guide](#upgrade-guide)

---

## Prerequisites

### Infrastructure Requirements

#### Minimum Requirements (Development)
```yaml
Kubernetes Cluster:
  Version: 1.28+
  Nodes: 3
  CPU: 8 cores total
  Memory: 16 GB total
  Storage: 50 GB

Control Plane:
  CPU: 2 cores
  Memory: 4 GB
```

#### Recommended Requirements (Production)
```yaml
Kubernetes Cluster:
  Version: 1.28+
  Nodes: 5+ (with autoscaling)
  CPU: 32+ cores total
  Memory: 64+ GB total
  Storage: 200+ GB (with dynamic provisioning)

Control Plane:
  CPU: 4 cores
  Memory: 8 GB
  High Availability: 3 replicas
```

### Required Tools

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installations
kubectl version --client
helm version
```

### Access Requirements

- Kubernetes cluster admin access
- Container registry access (Docker Hub, ECR, GCR, etc.)
- API keys for LLM provider (OpenAI, Anthropic, or local LLM)
- (Optional) Cloud provider credentials for managed services

---

### Python Dependencies

Install all required Python packages:

```bash
# Install all dependencies at once
pip install -r requirements.txt

# Or install component-specific dependencies
pip install -r orchestrator/requirements.txt
pip install -r mcp-servers/k8s-monitor/requirements.txt
pip install -r mcp-servers/metrics/requirements.txt
pip install -r mcp-servers/detection/requirements.txt
pip install -r mcp-servers/actions/requirements.txt
pip install -r mcp-servers/knowledge-base/requirements.txt
pip install -r mcp-servers/notifications/requirements.txt
```

**Key Dependencies**:
- MCP SDK (1.0.0)
- Anthropic Claude (0.39.0)
- Kubernetes Client (31.0.0)
- FastAPI (0.115.5)
- ChromaDB (0.5.20)
- Prometheus Client (0.21.0)


## Quick Start

### Local Development Deployment

```bash
# 1. Create local cluster
kind create cluster --name k8s-assistant --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# 2. Clone repository
git clone https://github.com/your-org/k8s-assistant.git
cd k8s-assistant

# 3. Create namespace
kubectl create namespace k8s-assistant

# 4. Install monitoring stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false

# 5. Create secrets
kubectl create secret generic k8s-assistant-secrets \
  --namespace k8s-assistant \
  --from-literal=openai-api-key=YOUR_OPENAI_API_KEY \
  --from-literal=postgres-password=YOUR_POSTGRES_PASSWORD \
  --from-literal=redis-password=YOUR_REDIS_PASSWORD

# 6. Install dependencies
helm install postgresql bitnami/postgresql \
  --namespace k8s-assistant \
  --set auth.password=YOUR_POSTGRES_PASSWORD \
  --set primary.persistence.size=10Gi

helm install redis bitnami/redis \
  --namespace k8s-assistant \
  --set auth.password=YOUR_REDIS_PASSWORD

# 7. Deploy k8s-assistant
helm install k8s-assistant ./charts/k8s-assistant \
  --namespace k8s-assistant \
  --values config/values-dev.yaml

# 8. Verify deployment
kubectl get pods -n k8s-assistant
kubectl logs -n k8s-assistant -l app=k8s-assistant --tail=50
```

### Verify Installation

```bash
# Check all pods are running
kubectl get pods -n k8s-assistant

# Expected output:
# NAME                                READY   STATUS    RESTARTS   AGE
# k8s-assistant-watcher-xxx           1/1     Running   0          2m
# k8s-assistant-detector-xxx          1/1     Running   0          2m
# k8s-assistant-ai-service-xxx        1/1     Running   0          2m
# k8s-assistant-action-controller-xxx 1/1     Running   0          2m
# k8s-assistant-api-xxx               1/1     Running   0          2m
# postgresql-xxx                      1/1     Running   0          3m
# redis-xxx                           1/1     Running   0          3m

# Check services
kubectl get svc -n k8s-assistant

# Test API endpoint
kubectl port-forward -n k8s-assistant svc/k8s-assistant-api 8080:8080 &
curl http://localhost:8080/health
```

---

## Production Deployment

### Step 1: Prepare Infrastructure

#### Cloud Provider Setup (AWS Example)

```bash
# Create EKS cluster
eksctl create cluster \
  --name k8s-assistant-prod \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.xlarge \
  --nodes 5 \
  --nodes-min 3 \
  --nodes-max 10 \
  --managed

# Configure kubectl
aws eks update-kubeconfig --name k8s-assistant-prod --region us-east-1

# Create storage class for persistent volumes
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
EOF
```

### Step 2: Install Monitoring Stack

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus Operator
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values - <<EOF
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    resources:
      requests:
        cpu: 2
        memory: 4Gi
      limits:
        cpu: 4
        memory: 8Gi

grafana:
  adminPassword: CHANGE_ME
  persistence:
    enabled: true
    storageClassName: fast-ssd
    size: 10Gi
  
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
EOF

# Install Loki for logs
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --values - <<EOF
loki:
  persistence:
    enabled: true
    storageClassName: fast-ssd
    size: 50Gi
  resources:
    requests:
      cpu: 1
      memory: 2Gi
    limits:
      cpu: 2
      memory: 4Gi

promtail:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
EOF
```

### Step 3: Setup Databases

#### PostgreSQL (Production)

```bash
# Install PostgreSQL with replication
helm install postgresql bitnami/postgresql \
  --namespace k8s-assistant \
  --create-namespace \
  --values - <<EOF
auth:
  postgresPassword: CHANGE_ME
  database: k8s_assistant

architecture: replication
replication:
  enabled: true
  numSynchronousReplicas: 1

primary:
  persistence:
    enabled: true
    storageClass: fast-ssd
    size: 100Gi
  resources:
    requests:
      cpu: 2
      memory: 4Gi
    limits:
      cpu: 4
      memory: 8Gi

readReplicas:
  replicaCount: 2
  persistence:
    enabled: true
    storageClass: fast-ssd
    size: 100Gi
  resources:
    requests:
      cpu: 1
      memory: 2Gi
    limits:
      cpu: 2
      memory: 4Gi

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

backup:
  enabled: true
  cronjob:
    schedule: "0 2 * * *"
    storage:
      size: 50Gi
EOF
```

#### Redis (Production)

```bash
# Install Redis with sentinel
helm install redis bitnami/redis \
  --namespace k8s-assistant \
  --values - <<EOF
architecture: replication

auth:
  password: CHANGE_ME

master:
  persistence:
    enabled: true
    storageClass: fast-ssd
    size: 20Gi
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1
      memory: 2Gi

replica:
  replicaCount: 2
  persistence:
    enabled: true
    storageClass: fast-ssd
    size: 20Gi
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1
      memory: 2Gi

sentinel:
  enabled: true
  quorum: 2

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
EOF
```

### Step 4: Create Secrets

```bash
# Create secrets from files
kubectl create secret generic k8s-assistant-secrets \
  --namespace k8s-assistant \
  --from-literal=openai-api-key="${OPENAI_API_KEY}" \
  --from-literal=postgres-password="${POSTGRES_PASSWORD}" \
  --from-literal=redis-password="${REDIS_PASSWORD}" \
  --from-literal=slack-webhook-url="${SLACK_WEBHOOK_URL}"

# Or use sealed secrets for GitOps
kubectl apply -f - <<EOF
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: k8s-assistant-secrets
  namespace: k8s-assistant
spec:
  encryptedData:
    openai-api-key: AgBx... # encrypted value
    postgres-password: AgBy... # encrypted value
    redis-password: AgBz... # encrypted value
EOF
```

### Step 5: Deploy Application

```bash
# Create production values file
cat > values-prod.yaml <<EOF
global:
  environment: production
  clusterName: k8s-assistant-prod

watcher:
  replicaCount: 3
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1
      memory: 2Gi
  
  metrics:
    interval: 15s
  
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - watcher
          topologyKey: kubernetes.io/hostname

detector:
  replicaCount: 2
  resources:
    requests:
      cpu: 1
      memory: 2Gi
    limits:
      cpu: 2
      memory: 4Gi
  
  anomalyThreshold: 2.0
  minSamples: 10

aiService:
  replicaCount: 3
  resources:
    requests:
      cpu: 2
      memory: 4Gi
    limits:
      cpu: 4
      memory: 8Gi
  
  llm:
    provider: openai
    model: gpt-4
    temperature: 0.2
    maxTokens: 2000
  
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70

actionController:
  replicaCount: 1  # Single active instance
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1
      memory: 2Gi
  
  autoFix:
    enabled: true
    confidenceThreshold: 0.90
  
  rateLimit:
    maxPerMinute: 10
    maxPerHour: 50
  
  circuitBreaker:
    failureThreshold: 5
    timeout: 60s

api:
  replicaCount: 2
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 1
      memory: 1Gi
  
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb

postgresql:
  enabled: false  # Using external PostgreSQL
  host: postgresql-primary.k8s-assistant.svc.cluster.local
  port: 5432
  database: k8s_assistant

redis:
  enabled: false  # Using external Redis
  host: redis-master.k8s-assistant.svc.cluster.local
  port: 6379

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: k8s-assistant.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: k8s-assistant-tls
      hosts:
        - k8s-assistant.example.com

monitoring:
  serviceMonitor:
    enabled: true
  
  prometheusRules:
    enabled: true

notifications:
  slack:
    enabled: true
    channel: "#sre-alerts"
  
  email:
    enabled: true
    smtpHost: smtp.gmail.com
    smtpPort: 587
    from: alerts@example.com
EOF

# Deploy with Helm
helm install k8s-assistant ./charts/k8s-assistant \
  --namespace k8s-assistant \
  --values values-prod.yaml \
  --wait \
  --timeout 10m

# Verify deployment
kubectl get pods -n k8s-assistant -w
```

### Step 6: Configure RBAC

```bash
# Apply RBAC configuration
kubectl apply -f config/rbac.yaml

# Verify service account
kubectl get serviceaccount k8s-assistant -n k8s-assistant
kubectl describe clusterrole k8s-assistant
kubectl describe clusterrolebinding k8s-assistant
```

### Step 7: Setup Ingress & TLS

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ClusterIssuer
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Install nginx ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

---

## Configuration

### Environment-Specific Configuration

#### Development
```yaml
# config/values-dev.yaml
global:
  environment: development
  logLevel: debug

watcher:
  replicaCount: 1
  resources:
    requests:
      cpu: 100m
      memory: 256Mi

detector:
  replicaCount: 1
  resources:
    requests:
      cpu: 200m
      memory: 512Mi

aiService:
  replicaCount: 1
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
  llm:
    provider: local  # Use local LLM for dev
    model: llama2

actionController:
  autoFix:
    enabled: false  # Disable auto-fix in dev
```

#### Staging
```yaml
# config/values-staging.yaml
global:
  environment: staging
  logLevel: info

watcher:
  replicaCount: 2

detector:
  replicaCount: 1

aiService:
  replicaCount: 2
  llm:
    provider: openai
    model: gpt-3.5-turbo  # Use cheaper model for staging

actionController:
  autoFix:
    enabled: true
    confidenceThreshold: 0.95  # Higher threshold for staging
```

### Feature Flags

```yaml
# config/features.yaml
features:
  autoFix:
    enabled: true
    allowedNamespaces:
      - production
      - staging
    excludedNamespaces:
      - kube-system
      - monitoring
  
  aiAnalysis:
    enabled: true
    providers:
      - openai
      - anthropic
    fallbackToLocal: true
  
  notifications:
    slack: true
    email: true
    pagerduty: false
  
  learning:
    enabled: true
    retrainInterval: 24h
```

---

## Monitoring Setup

### Grafana Dashboards

```bash
# Import dashboards
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: k8s-assistant-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  k8s-assistant.json: |
    {
      "dashboard": {
        "title": "K8s Assistant Overview",
        "panels": [
          {
            "title": "Issues Detected",
            "targets": [
              {
                "expr": "rate(k8s_assistant_issues_detected_total[5m])"
              }
            ]
          },
          {
            "title": "Auto-Fix Success Rate",
            "targets": [
              {
                "expr": "rate(k8s_assistant_autofix_success_total[5m]) / rate(k8s_assistant_autofix_attempts_total[5m])"
              }
            ]
          }
        ]
      }
    }
EOF
```

### Prometheus Rules

```yaml
# config/prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: k8s-assistant-alerts
  namespace: monitoring
spec:
  groups:
  - name: k8s-assistant
    interval: 30s
    rules:
    - alert: K8sAssistantDown
      expr: up{job="k8s-assistant"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "K8s Assistant is down"
        description: "K8s Assistant has been down for more than 5 minutes"
    
    - alert: HighDetectionLatency
      expr: histogram_quantile(0.95, rate(k8s_assistant_detection_duration_seconds_bucket[5m])) > 60
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High detection latency"
        description: "P95 detection latency is above 60 seconds"
    
    - alert: LowAutoFixSuccessRate
      expr: rate(k8s_assistant_autofix_success_total[1h]) / rate(k8s_assistant_autofix_attempts_total[1h]) < 0.8
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Low auto-fix success rate"
        description: "Auto-fix success rate is below 80%"
```

### Alertmanager Configuration

```yaml
# config/alertmanager.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      slack_api_url: 'YOUR_SLACK_WEBHOOK_URL'
    
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'slack-notifications'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty'
    
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#sre-alerts'
        title: 'K8s Assistant Alert'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    
    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
```

---

## Scaling

### Horizontal Pod Autoscaling

```yaml
# Apply HPA for AI service
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: k8s-assistant-ai-service
  namespace: k8s-assistant
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: k8s-assistant-ai-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 2
        periodSeconds: 30
      selectPolicy: Max
```

### Cluster Autoscaling

```bash
# AWS EKS example
eksctl create nodegroup \
  --cluster k8s-assistant-prod \
  --name autoscaling-workers \
  --node-type t3.xlarge \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 20 \
  --asg-access \
  --managed

# Enable cluster autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

---

## Backup & Recovery

### Database Backup

```bash
# PostgreSQL backup script
cat > backup-postgres.sh <<'EOF'
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/postgres"
NAMESPACE="k8s-assistant"

kubectl exec -n $NAMESPACE postgresql-primary-0 -- \
  pg_dump -U postgres k8s_assistant | \
  gzip > $BACKUP_DIR/k8s_assistant_$TIMESTAMP.sql.gz

# Upload to S3
aws s3 cp $BACKUP_DIR/k8s_assistant_$TIMESTAMP.sql.gz \
  s3://your-backup-bucket/postgres/

# Keep only last 30 days
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete
EOF

chmod +x backup-postgres.sh

# Create CronJob
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: k8s-assistant
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command: ["/bin/bash", "-c"]
            args:
            - |
              pg_dump -h postgresql-primary -U postgres k8s_assistant | \
              gzip > /backup/k8s_assistant_$(date +%Y%m%d_%H%M%S).sql.gz
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: postgres-backup-pvc
          restartPolicy: OnFailure
EOF
```

### Disaster Recovery

```bash
# Create disaster recovery plan
cat > disaster-recovery.md <<'EOF'
# Disaster Recovery Plan

## RTO (Recovery Time Objective): 1 hour
## RPO (Recovery Point Objective): 1 hour

### Recovery Steps:

1. **Restore Database**
   ```bash
   # Download latest backup
   aws s3 cp s3://your-backup-bucket/postgres/latest.sql.gz /tmp/
   
   # Restore to PostgreSQL
   gunzip < /tmp/latest.sql.gz | \
   kubectl exec -i -n k8s-assistant postgresql-primary-0 -- \
     psql -U postgres k8s_assistant
   ```

2. **Redeploy Application**
   ```bash
   helm upgrade k8s-assistant ./charts/k8s-assistant \
     --namespace k8s-assistant \
     --values values-prod.yaml \
     --force
   ```

3. **Verify Services**
   ```bash
   kubectl get pods -n k8s-assistant
   kubectl logs -n k8s-assistant -l app=k8s-assistant
   ```

4. **Test Functionality**
   - Check API health endpoint
   - Verify issue detection
   - Test auto-fix capability
EOF
```

---

## Troubleshooting

### Common Issues

#### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n k8s-assistant

# Describe pod for events
kubectl describe pod <pod-name> -n k8s-assistant

# Check logs
kubectl logs <pod-name> -n k8s-assistant --previous

# Common fixes:
# 1. Check resource limits
kubectl top pods -n k8s-assistant

# 2. Check secrets
kubectl get secrets -n k8s-assistant
kubectl describe secret k8s-assistant-secrets -n k8s-assistant

# 3. Check RBAC
kubectl auth can-i --list --as=system:serviceaccount:k8s-assistant:k8s-assistant
```

#### High Memory Usage

```bash
# Check memory usage
kubectl top pods -n k8s-assistant

# Increase memory limits
kubectl patch deployment k8s-assistant-ai-service -n k8s-assistant -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "ai-service",
          "resources": {
            "limits": {
              "memory": "8Gi"
            }
          }
        }]
      }
    }
  }
}'
```

#### LLM API Errors

```bash
# Check API key
kubectl get secret k8s-assistant-secrets -n k8s-assistant -o jsonpath='{.data.openai-api-key}' | base64 -d

# Check logs for API errors
kubectl logs -n k8s-assistant -l app=ai-service | grep -i "api error"

# Test API connectivity
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://api.openai.com/v1/models
```

### Debug Mode

```bash
# Enable debug logging
kubectl set env deployment/k8s-assistant-detector -n k8s-assistant LOG_LEVEL=debug

# Watch logs in real-time
kubectl logs -f -n k8s-assistant -l app=detector

# Disable auto-fix for debugging
kubectl patch configmap k8s-assistant-config -n k8s-assistant -p '
{
  "data": {
    "AUTO_FIX_ENABLED": "false"
  }
}'
```

---

## Upgrade Guide

### Minor Version Upgrade

```bash
# 1. Backup current state
kubectl get all -n k8s-assistant -o yaml > backup-$(date +%Y%m%d).yaml

# 2. Update Helm chart
helm repo update

# 3. Check what will change
helm diff upgrade k8s-assistant ./charts/k8s-assistant \
  --namespace k8s-assistant \
  --values values-prod.yaml

# 4. Perform upgrade
helm upgrade k8s-assistant ./charts/k8s-assistant \
  --namespace k8s-assistant \
  --values values-prod.yaml \
  --wait \
  --timeout 10m

# 5. Verify upgrade
kubectl rollout status deployment -n k8s-assistant
kubectl get pods -n k8s-assistant
```

### Major Version Upgrade

```bash
# 1. Read release notes
# 2. Backup database
./backup-postgres.sh

# 3. Run migration scripts
kubectl apply -f migrations/v2.0.0/

# 4. Upgrade with new values
helm upgrade k8s-assistant ./charts/k8s-assistant \
  --namespace k8s-assistant \
  --values values-prod-v2.yaml \
  --wait

# 5. Verify and test
./tests/e2e/test_full_workflow.sh
```

### Rollback

```bash
# List releases
helm history k8s-assistant -n k8s-assistant

# Rollback to previous version
helm rollback k8s-assistant -n k8s-assistant

# Rollback to specific revision
helm rollback k8s-assistant 3 -n k8s-assistant
```

---

## Production Checklist

Before going to production, ensure:

- [ ] All secrets are properly configured
- [ ] RBAC permissions are minimal and correct
- [ ] Resource limits are set appropriately
- [ ] Monitoring and alerting are configured
- [ ] Backup strategy is in place
- [ ] Disaster recovery plan is documented
- [ ] High availability is configured (multiple replicas)
- [ ] Auto-scaling is configured
- [ ] Network policies are applied
- [ ] TLS/SSL certificates are configured
- [ ] Logging is centralized
- [ ] Performance testing is completed
- [ ] Security scanning is performed
- [ ] Documentation is up to date
- [ ] Team is trained on the system

---

## Support

For issues and questions:
- GitHub Issues: https://github.com/your-org/k8s-assistant/issues
- Slack: #k8s-assistant
- Email: support@example.com

---

## Next Steps

After deployment:
1. Review the [Configuration Guide](docs/configuration/)
2. Set up [Custom Analyzers](docs/examples/custom-analyzers.md)
3. Configure [Notification Channels](docs/configuration/notifications.md)
4. Train the system with [Historical Data](docs/examples/training.md)
5. Review [Best Practices](docs/best-practices.md)