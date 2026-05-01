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
kubectl logs -n k8s-assistant deployment/k8s-assistant-detector | grep "Configuration loaded"