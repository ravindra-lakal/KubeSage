# Configuration Guide

## K8s Assistant Configuration

This guide covers all configuration options for the K8s Assistant.

---

## Table of Contents

1. [Configuration Files](#configuration-files)
2. [Environment Variables](#environment-variables)
3. [Feature Flags](#feature-flags)
4. [Detection Configuration](#detection-configuration)
5. [AI Configuration](#ai-configuration)
6. [Action Configuration](#action-configuration)
7. [Notification Configuration](#notification-configuration)
8. [Advanced Configuration](#advanced-configuration)

---

## Configuration Files

### Main Configuration File

Location: `config/config.yaml`

```yaml
# Global settings
global:
  environment: production
  cluster_name: production-cluster
  log_level: info  # debug, info, warn, error
  log_format: json  # json, text

# Kubernetes configuration
kubernetes:
  kubeconfig: ~/.kube/config  # Path to kubeconfig
  context: ""  # Leave empty to use current context
  namespaces:
    watch:
      - production
      - staging
    exclude:
      - kube-system
      - kube-public

# Monitoring configuration
monitoring:
  prometheus:
    url: http://prometheus-server:9090
    scrape_interval: 15s
  
  loki:
    url: http://loki:3100
    batch_size: 1000
  
  metrics_server:
    enabled: true
    url: https://metrics-server.kube-system.svc

# AI configuration
ai:
  provider: openai  # openai, anthropic, local
  api_key: ${OPENAI_API_KEY}
  model: gpt-4
  temperature: 0.2
  max_tokens: 2000
  timeout: 60s
  
  # Fallback configuration
  fallback:
    enabled: true
    provider: local
    model: llama2

# Storage configuration
storage:
  postgres:
    host: postgresql-primary
    port: 5432
    database: k8s_assistant
    user: postgres
    password: ${POSTGRES_PASSWORD}
    ssl_mode: require
    max_connections: 100
    connection_timeout: 30s
  
  redis:
    host: redis-master
    port: 6379
    password: ${REDIS_PASSWORD}
    db: 0
    max_retries: 3
    pool_size: 50
  
  vector_db:
    provider: chromadb  # chromadb, pinecone, weaviate
    url: http://chromadb:8000
    collection: k8s_incidents

# Detection configuration
detection:
  metrics:
    interval: 15s
    retention: 24h
  
  anomaly:
    threshold: 2.0  # Standard deviations
    min_samples: 10
    window: 15m
  
  classifiers:
    - name: oom_kill
      enabled: true
      priority: 1
    - name: crash_loop
      enabled: true
      priority: 2
    - name: high_latency
      enabled: true
      priority: 3

# Action configuration
actions:
  auto_fix:
    enabled: true
    confidence_threshold: 0.90
    dry_run: false
  
  rate_limit:
    enabled: true
    max_per_minute: 10
    max_per_hour: 50
  
  circuit_breaker:
    enabled: true
    failure_threshold: 5
    timeout: 60s
    half_open_requests: 3
  
  validation:
    timeout: 5m
    check_interval: 10s
  
  rollback:
    enabled: true
    automatic: true

# Notification configuration
notifications:
  slack:
    enabled: true
    webhook_url: ${SLACK_WEBHOOK_URL}
    channel: "#sre-alerts"
    username: "K8s Assistant"
    icon_emoji: ":robot_face:"
  
  email:
    enabled: true
    smtp_host: smtp.gmail.com
    smtp_port: 587
    from: alerts@example.com
    to:
      - sre-team@example.com
    username: ${EMAIL_USERNAME}
    password: ${EMAIL_PASSWORD}
  
  pagerduty:
    enabled: false
    integration_key: ${PAGERDUTY_KEY}
    severity_mapping:
      critical: critical
      high: error
      medium: warning
      low: info

# Learning configuration
learning:
  enabled: true
  retrain_interval: 24h
  min_samples: 100
  confidence_adjustment: true
  feedback_weight: 0.3
```

---

## Environment Variables

### Required Variables

```bash
# AI Provider
export OPENAI_API_KEY="sk-..."
# OR
export ANTHROPIC_API_KEY="sk-ant-..."

# Database
export POSTGRES_PASSWORD="your-secure-password"
export REDIS_PASSWORD="your-secure-password"

# Notifications
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."
export EMAIL_PASSWORD="your-email-password"
```

### Optional Variables

```bash
# Override configuration file settings
export K8S_ASSISTANT_LOG_LEVEL="debug"
export K8S_ASSISTANT_AUTO_FIX_ENABLED="false"
export K8S_ASSISTANT_CONFIDENCE_THRESHOLD="0.95"

# Feature flags
export FEATURE_AUTO_FIX="true"
export FEATURE_AI_ANALYSIS="true"
export FEATURE_LEARNING="true"
```

---

## Feature Flags

### Configuration File

```yaml
# config/features.yaml
features:
  auto_fix:
    enabled: true
    allowed_namespaces:
      - production
      - staging
    excluded_namespaces:
      - kube-system
      - monitoring
    allowed_issue_types:
      - OOMKill
      - Crash
      - HighLatency
    excluded_issue_types:
      - SecurityVulnerability
  
  ai_analysis:
    enabled: true
    providers:
      - openai
      - anthropic
    fallback_to_local: true
    cache_results: true
    cache_ttl: 1h
  
  notifications:
    slack: true
    email: true
    pagerduty: false
    webhook: false
  
  learning:
    enabled: true
    retrain_interval: 24h
    auto_update_thresholds: true
    feedback_collection: true
  
  experimental:
    predictive_analysis: false
    multi_cluster: false
    custom_analyzers: true
```

### Runtime Feature Flags

```bash
# Enable/disable features at runtime
kubectl patch configmap k8s-assistant-features -n k8s-assistant -p '
{
  "data": {
    "auto_fix_enabled": "false"
  }
}'

# Restart pods to apply changes
kubectl rollout restart deployment -n k8s-assistant
```

---

## Detection Configuration

### Anomaly Detection

```yaml
detection:
  anomaly:
    # Statistical threshold (standard deviations)
    threshold: 2.0
    
    # Minimum samples before detecting anomalies
    min_samples: 10
    
    # Time window for baseline calculation
    window: 15m
    
    # Algorithms to use
    algorithms:
      - zscore
      - isolation_forest
      - lstm
    
    # Per-metric configuration
    metrics:
      memory_usage:
        threshold: 2.5
        window: 30m
      
      cpu_usage:
        threshold: 2.0
        window: 15m
      
      latency:
        threshold: 3.0
        window: 5m
```

### Issue Classification

```yaml
detection:
  classifiers:
    - name: oom_kill
      enabled: true
      priority: 1
      conditions:
        - event_reason: "OOMKilling"
        - exit_code: 137
      severity: high
    
    - name: crash_loop
      enabled: true
      priority: 2
      conditions:
        - restart_count: "> 3"
        - time_window: "10m"
      severity: high
    
    - name: high_latency
      enabled: true
      priority: 3
      conditions:
        - metric: "p95_latency"
        - threshold: "> baseline * 2"
      severity: medium
    
    - name: memory_leak
      enabled: true
      priority: 4
      conditions:
        - metric: "memory_growth_rate"
        - threshold: "> 0.1"  # 10% per hour
      severity: medium
```

---

## AI Configuration

### LLM Provider Configuration

#### OpenAI

```yaml
ai:
  provider: openai
  api_key: ${OPENAI_API_KEY}
  model: gpt-4
  temperature: 0.2
  max_tokens: 2000
  timeout: 60s
  
  # Advanced options
  options:
    frequency_penalty: 0.0
    presence_penalty: 0.0
    top_p: 1.0
```

#### Anthropic Claude

```yaml
ai:
  provider: anthropic
  api_key: ${ANTHROPIC_API_KEY}
  model: claude-3-opus-20240229
  temperature: 0.2
  max_tokens: 2000
  timeout: 60s
```

#### Local LLM (Ollama)

```yaml
ai:
  provider: local
  base_url: http://ollama:11434
  model: llama2
  temperature: 0.2
  max_tokens: 2000
  timeout: 120s
```

### Context Enrichment

```yaml
ai:
  context:
    # Maximum log lines to include
    max_log_lines: 1000
    
    # Metric window duration
    metric_window: 15m
    
    # Number of similar incidents to include
    similar_incidents: 3
    
    # Include pod configuration
    include_pod_config: true
    
    # Include service dependencies
    include_dependencies: true
```

### Solution Generation

```yaml
ai:
  solutions:
    # Maximum solutions to generate
    max_solutions: 5
    
    # Minimum confidence to consider
    min_confidence: 0.5
    
    # Solution strategies
    strategies:
      - quick_fix
      - optimal_fix
      - preventive_fix
    
    # Risk assessment
    risk_assessment:
      enabled: true
      factors:
        - service_criticality
        - time_of_day
        - recent_changes
```

---

## Action Configuration

### Auto-Fix Settings

```yaml
actions:
  auto_fix:
    enabled: true
    
    # Confidence threshold for auto-fix
    confidence_threshold: 0.90
    
    # Dry run mode (test without applying)
    dry_run: false
    
    # Allowed namespaces
    allowed_namespaces:
      - production
      - staging
    
    # Excluded namespaces
    excluded_namespaces:
      - kube-system
    
    # Time-based restrictions
    restrictions:
      business_hours:
        enabled: true
        timezone: "America/New_York"
        start: "09:00"
        end: "17:00"
        days: ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
        require_higher_confidence: true
        confidence_threshold: 0.95
```

### Rate Limiting

```yaml
actions:
  rate_limit:
    enabled: true
    
    # Per-minute limit
    max_per_minute: 10
    
    # Per-hour limit
    max_per_hour: 50
    
    # Per-day limit
    max_per_day: 200
    
    # Per-action-type limits
    per_type:
      patch_deployment: 5
      scale_deployment: 10
      restart_pod: 20
```

### Circuit Breaker

```yaml
actions:
  circuit_breaker:
    enabled: true
    
    # Number of failures before opening circuit
    failure_threshold: 5
    
    # Time to wait before trying again
    timeout: 60s
    
    # Number of requests to allow in half-open state
    half_open_requests: 3
    
    # Per-action-type configuration
    per_type:
      patch_deployment:
        failure_threshold: 3
        timeout: 120s
```

---

## Notification Configuration

### Slack

```yaml
notifications:
  slack:
    enabled: true
    webhook_url: ${SLACK_WEBHOOK_URL}
    channel: "#sre-alerts"
    username: "K8s Assistant"
    icon_emoji: ":robot_face:"
    
    # Message templates
    templates:
      incident_detected: |
        :warning: *Incident Detected*
        Type: {{.Type}}
        Pod: {{.PodName}}
        Namespace: {{.Namespace}}
      
      incident_resolved: |
        :white_check_mark: *Incident Resolved*
        Type: {{.Type}}
        Solution: {{.Solution}}
        Duration: {{.Duration}}
    
    # Notification rules
    rules:
      - severity: critical
        mention: "@channel"
      - severity: high
        mention: "@sre-team"
      - severity: medium
        mention: ""
```

### Email

```yaml
notifications:
  email:
    enabled: true
    smtp_host: smtp.gmail.com
    smtp_port: 587
    from: alerts@example.com
    
    # Recipients
    to:
      - sre-team@example.com
    cc:
      - management@example.com
    
    # Authentication
    username: ${EMAIL_USERNAME}
    password: ${EMAIL_PASSWORD}
    
    # TLS configuration
    tls:
      enabled: true
      skip_verify: false
    
    # Email templates
    templates:
      subject: "[K8s Assistant] {{.Severity}} - {{.Type}} in {{.Namespace}}"
      body: |
        Incident Details:
        - Type: {{.Type}}
        - Severity: {{.Severity}}
        - Pod: {{.PodName}}
        - Namespace: {{.Namespace}}
        - Detected: {{.DetectedAt}}
        
        Root Cause:
        {{.RootCause}}
        
        Solution Applied:
        {{.Solution}}
```

### PagerDuty

```yaml
notifications:
  pagerduty:
    enabled: true
    integration_key: ${PAGERDUTY_KEY}
    
    # Severity mapping
    severity_mapping:
      critical: critical
      high: error
      medium: warning
      low: info
    
    # Deduplication
    dedup_key_template: "{{.Namespace}}-{{.PodName}}-{{.Type}}"
```

---

## Advanced Configuration

### Multi-Cluster Support

```yaml
clusters:
  - name: production-us-east
    kubeconfig: /configs/prod-us-east.yaml
    context: prod-us-east
    priority: 1
  
  - name: production-eu-west
    kubeconfig: /configs/prod-eu-west.yaml
    context: prod-eu-west
    priority: 2
  
  - name: staging
    kubeconfig: /configs/staging.yaml
    context: staging
    priority: 3
```

### Custom Analyzers

```yaml
analyzers:
  custom:
    - name: database_connection_pool
      type: latency
      enabled: true
      script: /scripts/db_pool_analyzer.py
      config:
        threshold: 0.8
        check_interval: 30s
    
    - name: cache_hit_rate
      type: performance
      enabled: true
      script: /scripts/cache_analyzer.py
      config:
        min_hit_rate: 0.7
```

### Performance Tuning

```yaml
performance:
  # Worker pool sizes
  workers:
    metrics_collector: 5
    event_watcher: 3
    log_processor: 10
    analyzer: 5
  
  # Buffer sizes
  buffers:
    metrics: 10000
    events: 5000
    logs: 20000
  
  # Batch processing
  batch:
    size: 100
    timeout: 5s
  
  # Caching
  cache:
    enabled: true
    ttl: 5m
    max_size: 1000
```

---

## Configuration Validation

Validate your configuration:

```bash
# Validate configuration file
k8s-assistant validate-config --config config/config.yaml

# Test database connection
k8s-assistant test-connection --type postgres

# Test AI provider
k8s-assistant test-connection --type llm

# Dry run deployment
helm install k8s-assistant ./charts/k8s-assistant \
  --dry-run \
  --debug \
  --values config/values-prod.yaml
```

---

## Configuration Best Practices

1. **Use Environment Variables for Secrets**: Never commit secrets to version control
2. **Start Conservative**: Begin with higher confidence thresholds and lower rate limits
3. **Monitor Configuration Changes**: Track all configuration changes in audit logs
4. **Test in Staging First**: Always test configuration changes in staging before production
5. **Use Feature Flags**: Enable new features gradually using feature flags
6. **Document Custom Settings**: Document any custom configuration for your team
7. **Regular Reviews**: Review and update configuration regularly based on system behavior

---

## Troubleshooting Configuration

### Common Issues

1. **Auto-fix not working**: Check confidence threshold and feature flags
2. **High false positive rate**: Adjust anomaly detection threshold
3. **Missing notifications**: Verify webhook URLs and credentials
4. **Performance issues**: Tune worker pool sizes and buffer sizes

### Debug Configuration

```bash
# View current configuration
kubectl get configmap k8s-assistant-config -n k8s-assistant -o yaml

# Check configuration in pod
kubectl exec -it -n k8s-assistant deployment/k8s-assistant-detector -- \
  cat /config/config.yaml

# View effective configuration (with env vars applied)

---

## MCP Server Configuration

### MCP Servers Configuration File

Location: `config/mcp-servers.json`

```json
{
  "mcpServers": {
    "k8s-monitor": {
      "command": "python",
      "args": ["mcp-servers/k8s-monitor/server.py"],
      "env": {
        "KUBECONFIG": "/path/to/kubeconfig",
        "LOG_LEVEL": "info",
        "WATCH_NAMESPACES": "production,staging"
      },
      "transport": "stdio",
      "restart_policy": "always",
      "health_check": {
        "enabled": true,
        "interval": "30s",
        "timeout": "5s"
      }
    },
    "metrics": {
      "command": "python",
      "args": ["mcp-servers/metrics/server.py"],
      "env": {
        "PROMETHEUS_URL": "http://prometheus-server:9090",
        "LOKI_URL": "http://loki:3100",
        "SCRAPE_INTERVAL": "15s",
        "LOG_LEVEL": "info"
      },
      "transport": "stdio",
      "restart_policy": "always",
      "health_check": {
        "enabled": true,
        "interval": "30s",
        "timeout": "5s"
      }
    },
    "detection": {
      "command": "python",
      "args": ["mcp-servers/detection/server.py"],
      "env": {
        "ANOMALY_THRESHOLD": "2.0",
        "MIN_SAMPLES": "10",
        "DETECTION_WINDOW": "15m",
        "LOG_LEVEL": "info"
      },
      "transport": "stdio",
      "restart_policy": "always",
      "health_check": {
        "enabled": true,
        "interval": "30s",
        "timeout": "5s"
      }
    },
    "actions": {
      "command": "python",
      "args": ["mcp-servers/actions/server.py"],
      "env": {
        "KUBECONFIG": "/path/to/kubeconfig",
        "DRY_RUN": "false",
        "RATE_LIMIT_PER_MINUTE": "10",
        "CIRCUIT_BREAKER_THRESHOLD": "5",
        "LOG_LEVEL": "info"
      },
      "transport": "stdio",
      "restart_policy": "always",
      "health_check": {
        "enabled": true,
        "interval": "30s",
        "timeout": "5s"
      }
    },
    "knowledge-base": {
      "command": "python",
      "args": ["mcp-servers/knowledge-base/server.py"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@postgresql:5432/k8s_assistant",
        "VECTOR_DB_URL": "http://chromadb:8000",
        "COLLECTION_NAME": "k8s_incidents",
        "LOG_LEVEL": "info"
      },
      "transport": "stdio",
      "restart_policy": "always",
      "health_check": {
        "enabled": true,
        "interval": "30s",
        "timeout": "5s"
      }
    },
    "notifications": {
      "command": "python",
      "args": ["mcp-servers/notifications/server.py"],
      "env": {
        "SLACK_WEBHOOK_URL": "${SLACK_WEBHOOK_URL}",
        "EMAIL_SMTP_HOST": "smtp.gmail.com",
        "EMAIL_SMTP_PORT": "587",
        "EMAIL_FROM": "alerts@example.com",
        "PAGERDUTY_KEY": "${PAGERDUTY_KEY}",
        "LOG_LEVEL": "info"
      },
      "transport": "stdio",
      "restart_policy": "always",
      "health_check": {
        "enabled": true,
        "interval": "30s",
        "timeout": "5s"
      }
    }
  },
  "orchestrator": {
    "ai_provider": "anthropic",
    "ai_model": "claude-3-5-sonnet-20241022",
    "ai_api_key": "${ANTHROPIC_API_KEY}",
    "max_concurrent_workflows": 10,
    "workflow_timeout": "10m",
    "retry_policy": {
      "max_retries": 3,
      "backoff": "exponential",
      "initial_delay": "1s",
      "max_delay": "30s"
    }
  }
}
```

### MCP Server-Specific Configuration

#### K8s Monitor Server

```yaml
# mcp-servers/k8s-monitor/config.yaml
server:
  name: k8s-monitor
  version: 1.0.0
  description: Kubernetes cluster monitoring MCP server

kubernetes:
  kubeconfig: ${KUBECONFIG}
  context: ""  # Use current context
  namespaces:
    watch:
      - production
      - staging
    exclude:
      - kube-system
      - kube-public

tools:
  watch_pods:
    enabled: true
    batch_size: 100
    timeout: 30s
  
  get_events:
    enabled: true
    max_events: 1000
    lookback: 1h
  
  fetch_logs:
    enabled: true
    max_lines: 10000
    tail_lines: 1000
  
  describe_resource:
    enabled: true
    cache_ttl: 5m

resources:
  pods:
    enabled: true
    cache_ttl: 30s
  
  events:
    enabled: true
    cache_ttl: 1m
  
  logs:
    enabled: true
    cache_ttl: 0s  # No caching for logs

performance:
  max_concurrent_requests: 50
  request_timeout: 30s
  connection_pool_size: 20
```

#### Metrics Server

```yaml
# mcp-servers/metrics/config.yaml
server:
  name: metrics
  version: 1.0.0
  description: Metrics and time-series data MCP server

prometheus:
  url: ${PROMETHEUS_URL}
  timeout: 30s
  max_samples: 10000

loki:
  url: ${LOKI_URL}
  timeout: 30s
  max_lines: 5000

tools:
  query_metrics:
    enabled: true
    default_step: 15s
    max_range: 24h
  
  get_timeseries:
    enabled: true
    default_resolution: 1m
    max_points: 1000
  
  analyze_trends:
    enabled: true
    algorithms:
      - linear_regression
      - moving_average
      - exponential_smoothing
  
  detect_anomalies:
    enabled: true
    algorithms:
      - zscore
      - isolation_forest
      - lstm
    threshold: 2.0

resources:
  metrics:
    enabled: true
    cache_ttl: 15s

performance:
  cache_enabled: true
  cache_size: 1000
  batch_queries: true
  max_concurrent_queries: 20
```

#### Detection Server

```yaml
# mcp-servers/detection/config.yaml
server:
  name: detection
  version: 1.0.0
  description: Issue detection and classification MCP server

detection:
  anomaly:
    threshold: 2.0
    min_samples: 10
    window: 15m
    algorithms:
      - zscore
      - isolation_forest
  
  classifiers:
    - name: oom_kill
      enabled: true
      priority: 1
      confidence_threshold: 0.85
    
    - name: crash_loop
      enabled: true
      priority: 2
      confidence_threshold: 0.85
    
    - name: high_latency
      enabled: true
      priority: 3
      confidence_threshold: 0.80
    
    - name: memory_leak
      enabled: true
      priority: 4
      confidence_threshold: 0.75

tools:
  detect_anomaly:
    enabled: true
    timeout: 5s
  
  classify_issue:
    enabled: true
    timeout: 10s
    use_ml_models: true
  
  analyze_pattern:
    enabled: true
    pattern_library: /data/patterns
  
  correlate_events:
    enabled: true
    correlation_window: 5m

prompts:
  analyze_crash:
    enabled: true
    template: /prompts/analyze_crash.txt
  
  analyze_oom:
    enabled: true
    template: /prompts/analyze_oom.txt
  
  analyze_latency:
    enabled: true
    template: /prompts/analyze_latency.txt

ml_models:
  enabled: true
  model_path: /models
  models:
    - name: issue_classifier
      type: random_forest
      version: 1.0.0
    - name: anomaly_detector
      type: isolation_forest
      version: 1.0.0
```

#### Action Server

```yaml
# mcp-servers/actions/config.yaml
server:
  name: actions
  version: 1.0.0
  description: Remediation actions MCP server

kubernetes:
  kubeconfig: ${KUBECONFIG}
  context: ""
  dry_run: ${DRY_RUN:-false}

safety:
  rate_limit:
    enabled: true
    max_per_minute: 10
    max_per_hour: 50
  
  circuit_breaker:
    enabled: true
    failure_threshold: 5
    timeout: 60s
    half_open_requests: 3
  
  validation:
    enabled: true
    timeout: 5m
    check_interval: 10s
  
  rollback:
    enabled: true
    automatic: true
    timeout: 5m

tools:
  patch_resource:
    enabled: true
    allowed_kinds:
      - Deployment
      - StatefulSet
      - DaemonSet
      - ConfigMap
    excluded_namespaces:
      - kube-system
      - kube-public
  
  scale_workload:
    enabled: true
    min_replicas: 1
    max_replicas: 100
    allowed_kinds:
      - Deployment
      - StatefulSet
  
  restart_pod:
    enabled: true
    grace_period: 30s
  
  rollback_deployment:
    enabled: true
    max_revisions: 10
  
  dry_run:
    enabled: true
    always_available: true

approval:
  required_for:
    - scale_workload  # When scaling > 50%
    - rollback_deployment
  
  approvers:
    - sre-team@example.com
  
  timeout: 5m
```

#### Knowledge Base Server

```yaml
# mcp-servers/knowledge-base/config.yaml
server:
  name: knowledge-base
  version: 1.0.0
  description: Historical incident storage and retrieval MCP server

database:
  postgres:
    url: ${DATABASE_URL}
    max_connections: 50
    connection_timeout: 30s
  
  vector_db:
    provider: chromadb
    url: ${VECTOR_DB_URL}
    collection: ${COLLECTION_NAME:-k8s_incidents}
    embedding_model: text-embedding-ada-002

resources:
  incidents:
    enabled: true
    retention: 90d
    max_results: 100
  
  solutions:
    enabled: true
    retention: 365d
    max_results: 50
  
  patterns:
    enabled: true
    retention: 180d
    max_results: 20

tools:
  search_similar:
    enabled: true
    similarity_threshold: 0.7
    max_results: 5
    use_vector_search: true
  
  store_incident:
    enabled: true
    auto_embed: true
    validate_schema: true
  
  get_solution:
    enabled: true
    cache_ttl: 1h
  
  update_effectiveness:
    enabled: true
    track_metrics: true

learning:
  enabled: true
  retrain_interval: 24h
  min_samples: 100
  confidence_adjustment: true
  feedback_weight: 0.3

performance:
  cache_enabled: true
  cache_size: 500
  batch_operations: true
  max_batch_size: 100
```

#### Notification Server

```yaml
# mcp-servers/notifications/config.yaml
server:
  name: notifications
  version: 1.0.0
  description: Multi-channel notification MCP server

channels:
  slack:
    enabled: ${SLACK_ENABLED:-true}
    webhook_url: ${SLACK_WEBHOOK_URL}
    channel: "#sre-alerts"
    username: "K8s Assistant"
    icon_emoji: ":robot_face:"
    rate_limit: 10  # per minute
  
  email:
    enabled: ${EMAIL_ENABLED:-true}
    smtp_host: ${EMAIL_SMTP_HOST}
    smtp_port: ${EMAIL_SMTP_PORT}
    from: ${EMAIL_FROM}
    to:
      - sre-team@example.com
    tls: true
    rate_limit: 20  # per minute
  
  pagerduty:
    enabled: ${PAGERDUTY_ENABLED:-false}
    integration_key: ${PAGERDUTY_KEY}
    severity_mapping:
      critical: critical
      high: error
      medium: warning
      low: info
    rate_limit: 5  # per minute
  
  webhook:
    enabled: ${WEBHOOK_ENABLED:-false}
    url: ${WEBHOOK_URL}
    headers:
      Content-Type: application/json
    rate_limit: 30  # per minute

tools:
  send_slack:
    enabled: true
    timeout: 10s
    retry_count: 3
  
  send_email:
    enabled: true
    timeout: 30s
    retry_count: 3
  
  create_ticket:
    enabled: true
    timeout: 30s
    retry_count: 3
  
  send_webhook:
    enabled: true
    timeout: 10s
    retry_count: 3

templates:
  incident_detected: /templates/incident_detected.tmpl
  incident_resolved: /templates/incident_resolved.tmpl
  action_applied: /templates/action_applied.tmpl
  validation_failed: /templates/validation_failed.tmpl

filtering:
  enabled: true
  rules:
    - severity: low
      channels: [slack]
    - severity: medium
      channels: [slack, email]
    - severity: high
      channels: [slack, email, pagerduty]
    - severity: critical
      channels: [slack, email, pagerduty]
      mention: "@channel"
```

---

## MCP Transport Configuration

### stdio Transport (Default)

```json
{
  "mcpServers": {
    "k8s-monitor": {
      "command": "python",
      "args": ["mcp-servers/k8s-monitor/server.py"],
      "transport": "stdio"
    }
  }
}
```

### SSE Transport

```json
{
  "mcpServers": {
    "k8s-monitor": {
      "url": "http://k8s-monitor-server:8080/sse",
      "transport": "sse",
      "headers": {
        "Authorization": "Bearer ${MCP_TOKEN}"
      }
    }
  }
}
```

### WebSocket Transport

```json
{
  "mcpServers": {
    "k8s-monitor": {
      "url": "ws://k8s-monitor-server:8080/ws",
      "transport": "websocket",
      "headers": {
        "Authorization": "Bearer ${MCP_TOKEN}"
      },
      "reconnect": {
        "enabled": true,
        "max_attempts": 5,
        "backoff": "exponential"
      }
    }
  }
}
```

---

## MCP Orchestrator Configuration

```yaml
# config/orchestrator.yaml
orchestrator:
  name: k8s-assistant-orchestrator
  version: 1.0.0

ai:
  provider: anthropic
  api_key: ${ANTHROPIC_API_KEY}
  model: claude-3-5-sonnet-20241022
  temperature: 0.2
  max_tokens: 4000
  timeout: 60s
  
  fallback:
    enabled: true
    provider: openai
    model: gpt-4
    api_key: ${OPENAI_API_KEY}

mcp:
  servers_config: /config/mcp-servers.json
  connection_timeout: 30s
  request_timeout: 60s
  max_retries: 3
  
  discovery:
    enabled: true
    interval: 5m
  
  health_check:
    enabled: true
    interval: 30s
    timeout: 5s
    unhealthy_threshold: 3

workflow:
  max_concurrent: 10
  timeout: 10m
  max_steps: 20
  
  retry_policy:
    enabled: true
    max_retries: 3
    backoff: exponential
    initial_delay: 1s
    max_delay: 30s
  
  validation:
    enabled: true
    timeout: 5m
    check_interval: 10s

decision:
  auto_fix:
    enabled: true
    confidence_threshold: 0.90
    
  approval:
    required_for_high_risk: true
    timeout: 5m
  
  rate_limit:
    enabled: true
    max_per_minute: 10
    max_per_hour: 50

logging:
  level: info
  format: json
  output: stdout
  
  audit:
    enabled: true
    output: /var/log/audit.log
    include_tool_calls: true
    include_ai_responses: true

metrics:
  enabled: true
  port: 9090
  path: /metrics
  
  collect:
    - workflow_duration
    - tool_call_duration
    - ai_request_duration
    - success_rate
    - error_rate
```

---

## Environment-Specific MCP Configuration

### Development

```json
{
  "mcpServers": {
    "k8s-monitor": {
      "command": "python",
      "args": ["mcp-servers/k8s-monitor/server.py"],
      "env": {
        "LOG_LEVEL": "debug",
        "KUBECONFIG": "~/.kube/config"
      }
    },
    "actions": {
      "command": "python",
      "args": ["mcp-servers/actions/server.py"],
      "env": {
        "DRY_RUN": "true",
        "LOG_LEVEL": "debug"
      }
    }
  },
  "orchestrator": {
    "ai_provider": "openai",
    "ai_model": "gpt-3.5-turbo",
    "max_concurrent_workflows": 3
  }
}
```

### Production

```json
{
  "mcpServers": {
    "k8s-monitor": {
      "command": "python",
      "args": ["mcp-servers/k8s-monitor/server.py"],
      "env": {
        "LOG_LEVEL": "info",
        "KUBECONFIG": "/etc/kubernetes/kubeconfig"
      },
      "restart_policy": "always",
      "health_check": {
        "enabled": true,
        "interval": "30s"
      }
    },
    "actions": {
      "command": "python",
      "args": ["mcp-servers/actions/server.py"],
      "env": {
        "DRY_RUN": "false",
        "LOG_LEVEL": "info",
        "RATE_LIMIT_PER_MINUTE": "10"
      },
      "restart_policy": "always",
      "health_check": {
        "enabled": true,
        "interval": "30s"
      }
    }
  },
  "orchestrator": {
    "ai_provider": "anthropic",
    "ai_model": "claude-3-5-sonnet-20241022",
    "max_concurrent_workflows": 10,
    "retry_policy": {
      "max_retries": 3,
      "backoff": "exponential"
    }
  }
}
```

---

## MCP Configuration Validation

```bash
# Validate MCP servers configuration
k8s-assistant mcp validate-config --config config/mcp-servers.json

# Test MCP server connectivity
k8s-assistant mcp test-server --server k8s-monitor

# List available tools
k8s-assistant mcp list-tools --server k8s-monitor

# Test tool execution
k8s-assistant mcp test-tool \
  --server k8s-monitor \
  --tool watch_pods \
  --args '{"namespace": "default"}'

# Check MCP server health
k8s-assistant mcp health-check --all
```

---

## MCP Configuration Best Practices

1. **Use Environment Variables**: Store sensitive data in environment variables
2. **Enable Health Checks**: Always enable health checks for production
3. **Configure Timeouts**: Set appropriate timeouts for each server
4. **Enable Retry Logic**: Configure retry policies for resilience
5. **Monitor Performance**: Track tool execution times and success rates
6. **Use Transport Wisely**: stdio for local, SSE/WebSocket for remote
7. **Implement Rate Limiting**: Protect servers from overload
8. **Enable Audit Logging**: Track all MCP interactions
9. **Test Configuration**: Validate before deploying to production
10. **Document Custom Settings**: Maintain documentation for team

kubectl logs -n k8s-assistant deployment/k8s-assistant-detector | grep "Configuration loaded"