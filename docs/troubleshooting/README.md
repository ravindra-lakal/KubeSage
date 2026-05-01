# Troubleshooting Guide

## K8s Assistant Troubleshooting

This guide helps you diagnose and resolve common issues with the K8s Assistant.

---

## Table of Contents

1. [Quick Diagnostics](#quick-diagnostics)
2. [Common Issues](#common-issues)
3. [Component-Specific Issues](#component-specific-issues)
4. [Performance Issues](#performance-issues)
5. [Integration Issues](#integration-issues)
6. [Debug Mode](#debug-mode)

---

## Quick Diagnostics

### Health Check Commands

```bash
# Check all pods status
kubectl get pods -n k8s-assistant

# Check pod logs
kubectl logs -n k8s-assistant -l app=k8s-assistant --tail=100

# Check service endpoints
kubectl get endpoints -n k8s-assistant

# Check configmaps
kubectl get configmap -n k8s-assistant

# Check secrets
kubectl get secrets -n k8s-assistant

# Check resource usage
kubectl top pods -n k8s-assistant
kubectl top nodes
```

### API Health Check

```bash
# Port forward to API
kubectl port-forward -n k8s-assistant svc/k8s-assistant-api 8080:8080 &

# Check health endpoint
curl http://localhost:8080/health

# Expected response:
# {
#   "status": "healthy",
#   "components": {
#     "database": "healthy",
#     "redis": "healthy",
#     "llm": "healthy"
#   }
# }
```

---

## Common Issues

### Issue 1: Pods Not Starting

**Symptoms**:
- Pods stuck in `Pending` or `CrashLoopBackOff`
- Pods showing `ImagePullBackOff`

**Diagnosis**:
```bash
# Check pod status
kubectl describe pod <pod-name> -n k8s-assistant

# Check events
kubectl get events -n k8s-assistant --sort-by='.lastTimestamp'

# Check logs
kubectl logs <pod-name> -n k8s-assistant --previous
```

**Common Causes & Solutions**:

1. **Insufficient Resources**
   ```bash
   # Check node resources
   kubectl describe nodes
   
   # Solution: Add more nodes or reduce resource requests
   kubectl patch deployment <deployment> -n k8s-assistant -p '
   {
     "spec": {
       "template": {
         "spec": {
           "containers": [{
             "name": "container-name",
             "resources": {
               "requests": {
                 "cpu": "100m",
                 "memory": "256Mi"
               }
             }
           }]
         }
       }
     }
   }'
   ```

2. **Image Pull Errors**
   ```bash
   # Check image pull secrets
   kubectl get secrets -n k8s-assistant
   
   # Create image pull secret if missing
   kubectl create secret docker-registry regcred \
     --docker-server=<registry> \
     --docker-username=<username> \
     --docker-password=<password> \
     -n k8s-assistant
   ```

3. **Missing ConfigMaps/Secrets**
   ```bash
   # Check if secrets exist
   kubectl get secret k8s-assistant-secrets -n k8s-assistant
   
   # Create missing secrets
   kubectl create secret generic k8s-assistant-secrets \
     --from-literal=openai-api-key=YOUR_KEY \
     -n k8s-assistant
   ```

---

### Issue 2: Auto-Fix Not Working

**Symptoms**:
- Issues detected but not automatically fixed
- Manual intervention always required

**Diagnosis**:
```bash
# Check auto-fix configuration
kubectl get configmap k8s-assistant-config -n k8s-assistant -o yaml | grep auto_fix

# Check logs for auto-fix decisions
kubectl logs -n k8s-assistant deployment/k8s-assistant-action-controller | grep "auto-fix"
```

**Common Causes & Solutions**:

1. **Auto-Fix Disabled**
   ```bash
   # Enable auto-fix
   kubectl patch configmap k8s-assistant-config -n k8s-assistant -p '
   {
     "data": {
       "AUTO_FIX_ENABLED": "true"
     }
   }'
   
   # Restart pods
   kubectl rollout restart deployment -n k8s-assistant
   ```

2. **Confidence Threshold Too High**
   ```bash
   # Lower confidence threshold
   kubectl patch configmap k8s-assistant-config -n k8s-assistant -p '
   {
     "data": {
       "CONFIDENCE_THRESHOLD": "0.85"
     }
   }'
   ```

3. **Rate Limit Exceeded**
   ```bash
   # Check rate limit status
   kubectl logs -n k8s-assistant deployment/k8s-assistant-action-controller | grep "rate limit"
   
   # Increase rate limits
   kubectl patch configmap k8s-assistant-config -n k8s-assistant -p '
   {
     "data": {
       "RATE_LIMIT_PER_MINUTE": "20"
     }
   }'
   ```

4. **Circuit Breaker Open**
   ```bash
   # Check circuit breaker status
   kubectl logs -n k8s-assistant deployment/k8s-assistant-action-controller | grep "circuit breaker"
   
   # Reset circuit breaker (restart controller)
   kubectl rollout restart deployment/k8s-assistant-action-controller -n k8s-assistant
   ```

---

### Issue 3: High False Positive Rate

**Symptoms**:
- Too many alerts for non-issues
- Alerts for normal behavior

**Diagnosis**:
```bash
# Check detection metrics
kubectl logs -n k8s-assistant deployment/k8s-assistant-detector | grep "false positive"

# Review recent incidents
curl http://localhost:8080/api/v1/incidents?status=false_positive
```

**Solutions**:

1. **Adjust Anomaly Threshold**
   ```yaml
   # Edit config
   detection:
     anomaly:
       threshold: 3.0  # Increase from 2.0
       min_samples: 20  # Increase from 10
   ```

2. **Add Exclusions**
   ```yaml
   detection:
     exclude:
       namespaces:
         - test
         - development
       pods:
         - ".*-test-.*"
   ```

3. **Tune Per-Metric Thresholds**
   ```yaml
   detection:
     metrics:
       memory_usage:
         threshold: 2.5  # More lenient
       cpu_usage:
         threshold: 2.0
   ```

---

### Issue 4: LLM API Errors

**Symptoms**:
- Analysis failing
- Timeout errors
- API rate limit errors

**Diagnosis**:
```bash
# Check AI service logs
kubectl logs -n k8s-assistant deployment/k8s-assistant-ai-service | grep -i "error"

# Test API connectivity
kubectl exec -it -n k8s-assistant deployment/k8s-assistant-ai-service -- \
  curl -H "Authorization: Bearer $OPENAI_API_KEY" \
  https://api.openai.com/v1/models
```

**Solutions**:

1. **Invalid API Key**
   ```bash
   # Update API key
   kubectl create secret generic k8s-assistant-secrets \
     --from-literal=openai-api-key=NEW_KEY \
     --dry-run=client -o yaml | kubectl apply -f -
   
   # Restart AI service
   kubectl rollout restart deployment/k8s-assistant-ai-service -n k8s-assistant
   ```

2. **Rate Limit Exceeded**
   ```yaml
   # Switch to different model or add fallback
   ai:
     provider: openai
     model: gpt-3.5-turbo  # Cheaper, higher limits
     fallback:
       enabled: true
       provider: local
   ```

3. **Timeout Issues**
   ```yaml
   # Increase timeout
   ai:
     timeout: 120s  # Increase from 60s
     max_retries: 3
   ```

---

## Component-Specific Issues

### Watcher Service Issues

**Problem**: Metrics not being collected

```bash
# Check watcher logs
kubectl logs -n k8s-assistant deployment/k8s-assistant-watcher

# Check Prometheus connectivity
kubectl exec -it -n k8s-assistant deployment/k8s-assistant-watcher -- \
  curl http://prometheus-server:9090/-/healthy

# Verify RBAC permissions
kubectl auth can-i list pods --as=system:serviceaccount:k8s-assistant:k8s-assistant
```

**Solutions**:
```bash
# Fix RBAC if needed
kubectl apply -f config/rbac.yaml

# Restart watcher
kubectl rollout restart deployment/k8s-assistant-watcher -n k8s-assistant
```

---

### Detector Service Issues

**Problem**: Issues not being detected

```bash
# Check detector logs
kubectl logs -n k8s-assistant deployment/k8s-assistant-detector -f

# Check if receiving metrics
kubectl logs -n k8s-assistant deployment/k8s-assistant-detector | grep "metrics received"

# Check detection rules
kubectl get configmap k8s-assistant-detection-rules -n k8s-assistant -o yaml
```

**Solutions**:
```bash
# Enable debug logging
kubectl set env deployment/k8s-assistant-detector -n k8s-assistant LOG_LEVEL=debug

# Verify detection configuration
kubectl exec -it -n k8s-assistant deployment/k8s-assistant-detector -- \
  cat /config/detection.yaml
```

---

### AI Service Issues

**Problem**: Analysis taking too long

```bash
# Check AI service performance
kubectl top pod -n k8s-assistant -l app=ai-service

# Check queue depth
kubectl logs -n k8s-assistant deployment/k8s-assistant-ai-service | grep "queue"
```

**Solutions**:
```bash
# Scale AI service
kubectl scale deployment k8s-assistant-ai-service -n k8s-assistant --replicas=5

# Enable HPA
kubectl autoscale deployment k8s-assistant-ai-service \
  -n k8s-assistant \
  --min=3 --max=10 \
  --cpu-percent=70
```

---

### Action Controller Issues

**Problem**: Actions failing to apply

```bash
# Check action controller logs
kubectl logs -n k8s-assistant deployment/k8s-assistant-action-controller

# Check RBAC permissions
kubectl auth can-i patch deployments \
  --as=system:serviceaccount:k8s-assistant:k8s-assistant

# Check dry-run mode
kubectl get configmap k8s-assistant-config -n k8s-assistant -o yaml | grep dry_run
```

**Solutions**:
```bash
# Fix RBAC permissions
kubectl apply -f config/rbac.yaml

# Disable dry-run if enabled
kubectl patch configmap k8s-assistant-config -n k8s-assistant -p '
{
  "data": {
    "DRY_RUN": "false"
  }
}'
```

---

## Performance Issues

### High Memory Usage

**Diagnosis**:
```bash
# Check memory usage
kubectl top pods -n k8s-assistant

# Check for memory leaks
kubectl logs -n k8s-assistant <pod-name> | grep -i "memory"
```

**Solutions**:
```bash
# Increase memory limits
kubectl patch deployment <deployment> -n k8s-assistant -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "container-name",
          "resources": {
            "limits": {
              "memory": "4Gi"
            }
          }
        }]
      }
    }
  }
}'

# Enable memory profiling
kubectl set env deployment/<deployment> -n k8s-assistant \
  ENABLE_PROFILING=true \
  PROFILING_PORT=6060
```

---

### High CPU Usage

**Diagnosis**:
```bash
# Check CPU usage
kubectl top pods -n k8s-assistant

# Check for CPU throttling
kubectl describe pod <pod-name> -n k8s-assistant | grep -i throttl
```

**Solutions**:
```bash
# Increase CPU limits
kubectl patch deployment <deployment> -n k8s-assistant -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "container-name",
          "resources": {
            "limits": {
              "cpu": "2"
            }
          }
        }]
      }
    }
  }
}'

# Scale horizontally
kubectl scale deployment <deployment> -n k8s-assistant --replicas=3
```

---

### Slow Response Times

**Diagnosis**:
```bash
# Check API latency
kubectl logs -n k8s-assistant deployment/k8s-assistant-api | grep "latency"

# Check database performance
kubectl exec -it -n k8s-assistant postgresql-primary-0 -- \
  psql -U postgres -c "SELECT * FROM pg_stat_activity;"
```

**Solutions**:
```bash
# Enable caching
kubectl patch configmap k8s-assistant-config -n k8s-assistant -p '
{
  "data": {
    "CACHE_ENABLED": "true",
    "CACHE_TTL": "300"
  }
}'

# Optimize database
kubectl exec -it -n k8s-assistant postgresql-primary-0 -- \
  psql -U postgres -c "VACUUM ANALYZE;"

# Add database indexes
kubectl exec -it -n k8s-assistant postgresql-primary-0 -- \
  psql -U postgres k8s_assistant -c "CREATE INDEX idx_incidents_namespace ON incidents(namespace);"
```

---

## Integration Issues

### Slack Notifications Not Working

**Diagnosis**:
```bash
# Check Slack configuration
kubectl get secret k8s-assistant-secrets -n k8s-assistant -o jsonpath='{.data.slack-webhook-url}' | base64 -d

# Test webhook
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"Test from K8s Assistant"}' \
  YOUR_WEBHOOK_URL
```

**Solutions**:
```bash
# Update webhook URL
kubectl create secret generic k8s-assistant-secrets \
  --from-literal=slack-webhook-url=NEW_URL \
  --dry-run=client -o yaml | kubectl apply -f -

# Enable Slack notifications
kubectl patch configmap k8s-assistant-config -n k8s-assistant -p '
{
  "data": {
    "SLACK_ENABLED": "true"
  }
}'
```

---

### Prometheus Metrics Not Available

**Diagnosis**:
```bash
# Check Prometheus connectivity
kubectl exec -it -n k8s-assistant deployment/k8s-assistant-watcher -- \
  curl http://prometheus-server:9090/api/v1/query?query=up

# Check ServiceMonitor
kubectl get servicemonitor -n monitoring
```

**Solutions**:
```bash
# Create ServiceMonitor
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: k8s-assistant
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: k8s-assistant
  endpoints:
  - port: metrics
    interval: 30s
EOF
```

---

## Debug Mode

### Enable Debug Logging

```bash
# Enable debug for all components
kubectl set env deployment -n k8s-assistant --all LOG_LEVEL=debug

# Enable debug for specific component
kubectl set env deployment/k8s-assistant-detector -n k8s-assistant LOG_LEVEL=debug

# View debug logs
kubectl logs -f -n k8s-assistant deployment/k8s-assistant-detector
```

### Enable Profiling

```bash
# Enable CPU/Memory profiling
kubectl set env deployment/<deployment> -n k8s-assistant \
  ENABLE_PROFILING=true \
  PROFILING_PORT=6060

# Port forward to profiling endpoint
kubectl port-forward -n k8s-assistant deployment/<deployment> 6060:6060 &

# View CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile

# View memory profile
go tool pprof http://localhost:6060/debug/pprof/heap
```

### Trace Requests

```bash
# Enable request tracing
kubectl set env deployment/k8s-assistant-api -n k8s-assistant \
  ENABLE_TRACING=true \
  JAEGER_ENDPOINT=http://jaeger:14268/api/traces

# View traces in Jaeger UI
kubectl port-forward -n monitoring svc/jaeger-query 16686:16686
# Open http://localhost:16686
```

---

## Getting Help

If you're still experiencing issues:

1. **Collect Diagnostics**:
   ```bash
   # Run diagnostic script
   ./scripts/collect-diagnostics.sh
   
   # This creates a diagnostics bundle with:
   # - Pod logs
   # - Events
   # - Resource usage
   # - Configuration
   ```

2. **Check Documentation**:
   - [Architecture](../../ARCHITECTURE.md)
   - [Configuration Guide](../configuration/README.md)
   - [API Documentation](../api/README.md)

3. **Community Support**:
   - GitHub Issues: https://github.com/your-org/k8s-assistant/issues
   - Slack: #k8s-assistant
   - Stack Overflow: Tag `k8s-assistant`

4. **Professional Support**:
   - Email: support@example.com

---

## MCP-Specific Troubleshooting

### MCP Server Issues

#### Issue: MCP Server Not Starting

**Symptoms**:
- Server process exits immediately
- Connection refused errors
- Timeout when connecting to server

**Diagnosis**:
```bash
# Check MCP server logs
kubectl logs -n k8s-assistant deployment/k8s-assistant-orchestrator | grep "mcp-server"

# Test server manually
python mcp-servers/k8s-monitor/server.py

# Check server configuration
cat config/mcp-servers.json | jq '.mcpServers["k8s-monitor"]'
```

**Solutions**:

1. **Missing Dependencies**
   ```bash
   # Install MCP SDK
   pip install mcp
   
   # Install server-specific dependencies
   cd mcp-servers/k8s-monitor
   pip install -r requirements.txt
   ```

2. **Configuration Errors**
   ```bash
   # Validate MCP server config
   k8s-assistant mcp validate-config --config config/mcp-servers.json
   
   # Check environment variables
   kubectl get configmap k8s-assistant-config -n k8s-assistant -o yaml
   ```

3. **Permission Issues**
   ```bash
   # Check RBAC permissions
   kubectl auth can-i list pods --as=system:serviceaccount:k8s-assistant:k8s-assistant
   
   # Apply RBAC if needed
   kubectl apply -f config/rbac.yaml
   ```

---

#### Issue: MCP Tool Calls Failing

**Symptoms**:
- Tool execution returns errors
- Timeout errors
- Invalid response format

**Diagnosis**:
```bash
# Check tool execution logs
kubectl logs -n k8s-assistant deployment/k8s-assistant-orchestrator | grep "tool_call"

# Test tool manually
k8s-assistant mcp test-tool \
  --server k8s-monitor \
  --tool watch_pods \
  --args '{"namespace": "default"}'

# Check tool schema
k8s-assistant mcp list-tools --server k8s-monitor
```

**Solutions**:

1. **Invalid Arguments**
   ```python
   # Check tool schema
   async with session:
       tools = await session.list_tools()
       for tool in tools.tools:
           if tool.name == "watch_pods":
               print(tool.inputSchema)
   
   # Use correct argument format
   result = await session.call_tool(
       "watch_pods",
       arguments={
           "namespace": "production",  # Required
           "label_selector": "app=api"  # Optional
       }
   )
   ```

2. **Timeout Issues**
   ```json
   {
     "mcpServers": {
       "k8s-monitor": {
         "env": {
           "TOOL_TIMEOUT": "60s"
         }
       }
     }
   }
   ```

3. **Server Overload**
   ```bash
   # Check server metrics
   kubectl top pod -n k8s-assistant -l app=mcp-server
   
   # Scale MCP server
   kubectl scale deployment mcp-k8s-monitor -n k8s-assistant --replicas=3
   ```

---

#### Issue: MCP Resource Reading Fails

**Symptoms**:
- Resource not found errors
- Invalid URI format
- Permission denied

**Diagnosis**:
```bash
# List available resources
k8s-assistant mcp list-resources --server k8s-monitor

# Test resource reading
k8s-assistant mcp read-resource \
  --uri "pods://production/api-server-abc123"

# Check resource permissions
kubectl auth can-i get pods --as=system:serviceaccount:k8s-assistant:k8s-assistant
```

**Solutions**:

1. **Invalid URI Format**
   ```python
   # Correct URI formats
   pod_uri = "pods://namespace/pod-name"
   events_uri = "events://namespace"
   logs_uri = "logs://namespace/pod-name/container"
   incidents_uri = "incidents://"
   
   # Read resource
   resource = await session.read_resource(pod_uri)
   ```

2. **Resource Not Found**
   ```python
   # Check if resource exists first
   try:
       resource = await session.read_resource(uri)
   except McpError as e:
       if e.error.code == -32002:  # Resource not found
           print(f"Resource not found: {uri}")
   ```

3. **Permission Issues**
   ```bash
   # Grant necessary permissions
   kubectl apply -f - <<EOF
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: k8s-assistant-reader
   rules:
   - apiGroups: [""]
     resources: ["pods", "events", "pods/log"]
     verbs: ["get", "list", "watch"]
   EOF
   ```

---

### MCP Workflow Issues

#### Issue: Workflow Stuck or Hanging

**Symptoms**:
- Workflow doesn't complete
- No progress updates
- Timeout errors

**Diagnosis**:
```bash
# Check workflow status
curl -H "Authorization: Bearer $TOKEN" \
  https://k8s-assistant.example.com/api/v1/mcp/workflows

# Check orchestrator logs
kubectl logs -n k8s-assistant deployment/k8s-assistant-orchestrator -f

# Check MCP server connectivity
k8s-assistant mcp health-check --all
```

**Solutions**:

1. **Increase Timeouts**
   ```yaml
   # config/orchestrator.yaml
   workflow:
     timeout: 15m  # Increase from 10m
     max_steps: 30  # Increase from 20
   ```

2. **Check Server Health**
   ```bash
   # Restart unhealthy servers
   kubectl rollout restart deployment/mcp-k8s-monitor -n k8s-assistant
   
   # Check server logs
   kubectl logs -n k8s-assistant deployment/mcp-k8s-monitor
   ```

3. **Enable Debug Logging**
   ```bash
   # Enable debug mode
   kubectl set env deployment/k8s-assistant-orchestrator \
     -n k8s-assistant \
     LOG_LEVEL=debug \
     MCP_DEBUG=true
   ```

---

#### Issue: AI Analysis Failing

**Symptoms**:
- Claude API errors
- Analysis timeout
- Invalid responses

**Diagnosis**:
```bash
# Check AI service logs
kubectl logs -n k8s-assistant deployment/k8s-assistant-orchestrator | grep "claude"

# Test API connectivity
curl -H "Authorization: Bearer $ANTHROPIC_API_KEY" \
  https://api.anthropic.com/v1/messages \
  -X POST \
  -d '{"model":"claude-3-5-sonnet-20241022","max_tokens":100,"messages":[{"role":"user","content":"test"}]}'
```

**Solutions**:

1. **API Key Issues**
   ```bash
   # Update API key
   kubectl create secret generic k8s-assistant-secrets \
     --from-literal=anthropic-api-key=NEW_KEY \
     --dry-run=client -o yaml | kubectl apply -f -
   
   # Restart orchestrator
   kubectl rollout restart deployment/k8s-assistant-orchestrator -n k8s-assistant
   ```

2. **Rate Limiting**
   ```yaml
   # config/orchestrator.yaml
   ai:
     provider: anthropic
     model: claude-3-5-sonnet-20241022
     fallback:
       enabled: true
       provider: openai
       model: gpt-4
   ```

3. **Context Too Large**
   ```yaml
   # Reduce context size
   ai:
     context:
       max_log_lines: 500  # Reduce from 1000
       metric_window: 10m  # Reduce from 15m
       similar_incidents: 2  # Reduce from 3
   ```

---

### MCP Communication Issues

#### Issue: Server Connection Drops

**Symptoms**:
- Intermittent connection failures
- "Connection reset" errors
- Workflow failures mid-execution

**Diagnosis**:
```bash
# Check network connectivity
kubectl exec -it -n k8s-assistant deployment/k8s-assistant-orchestrator -- \
  ping mcp-k8s-monitor

# Check server health
kubectl get pods -n k8s-assistant -l app=mcp-server

# Check logs for connection errors
kubectl logs -n k8s-assistant deployment/k8s-assistant-orchestrator | grep "connection"
```

**Solutions**:

1. **Enable Reconnection**
   ```json
   {
     "mcpServers": {
       "k8s-monitor": {
         "transport": "websocket",
         "reconnect": {
           "enabled": true,
           "max_attempts": 5,
           "backoff": "exponential"
         }
       }
     }
   }
   ```

2. **Increase Timeouts**
   ```json
   {
     "mcp": {
       "connection_timeout": "60s",
       "request_timeout": "120s"
     }
   }
   ```

3. **Use Persistent Connections**
   ```python
   # Keep session alive
   async with session:
       # Perform multiple operations
       result1 = await session.call_tool("tool1", {})
       result2 = await session.call_tool("tool2", {})
       # Session stays open
   ```

---

#### Issue: High MCP Server Latency

**Symptoms**:
- Slow tool execution
- Workflow taking too long
- Timeout warnings

**Diagnosis**:
```bash
# Check server metrics
kubectl top pod -n k8s-assistant -l app=mcp-server

# Check tool execution times
curl -H "Authorization: Bearer $TOKEN" \
  https://k8s-assistant.example.com/api/v1/mcp/metrics/tools

# Profile server performance
kubectl port-forward -n k8s-assistant deployment/mcp-k8s-monitor 6060:6060
go tool pprof http://localhost:6060/debug/pprof/profile
```

**Solutions**:

1. **Scale MCP Servers**
   ```bash
   # Horizontal scaling
   kubectl scale deployment mcp-k8s-monitor -n k8s-assistant --replicas=5
   
   # Enable HPA
   kubectl autoscale deployment mcp-k8s-monitor \
     -n k8s-assistant \
     --min=3 --max=10 \
     --cpu-percent=70
   ```

2. **Enable Caching**
   ```yaml
   # mcp-servers/k8s-monitor/config.yaml
   resources:
     pods:
       cache_ttl: 30s
     events:
       cache_ttl: 1m
   
   performance:
     cache_enabled: true
     cache_size: 1000
   ```

3. **Optimize Queries**
   ```python
   # Use batch operations
   results = await asyncio.gather(
       session1.call_tool("tool1", {}),
       session2.call_tool("tool2", {}),
       session3.call_tool("tool3", {})
   )
   
   # Use resource caching
   resource = await session.read_resource(
       "pods://production/api-server",
       cache=True
   )
   ```

---

### MCP Orchestrator Issues

#### Issue: Orchestrator Not Discovering Servers

**Symptoms**:
- Servers not listed
- "Server not found" errors
- Tools not available

**Diagnosis**:
```bash
# Check orchestrator logs
kubectl logs -n k8s-assistant deployment/k8s-assistant-orchestrator | grep "discovery"

# List configured servers
cat config/mcp-servers.json | jq '.mcpServers | keys'

# Test server connectivity
k8s-assistant mcp test-server --server k8s-monitor
```

**Solutions**:

1. **Enable Discovery**
   ```yaml
   # config/orchestrator.yaml
   mcp:
     discovery:
       enabled: true
       interval: 5m
   ```

2. **Fix Server Configuration**
   ```json
   {
     "mcpServers": {
       "k8s-monitor": {
         "command": "python",
         "args": ["mcp-servers/k8s-monitor/server.py"],
         "env": {
           "LOG_LEVEL": "info"
         }
       }
     }
   }
   ```

3. **Restart Orchestrator**
   ```bash
   kubectl rollout restart deployment/k8s-assistant-orchestrator -n k8s-assistant
   ```

---

#### Issue: Circuit Breaker Open

**Symptoms**:
- Actions not executing
- "Circuit breaker open" errors
- Auto-fix disabled

**Diagnosis**:
```bash
# Check circuit breaker status
kubectl logs -n k8s-assistant deployment/k8s-assistant-orchestrator | grep "circuit breaker"

# Check action server logs
kubectl logs -n k8s-assistant deployment/mcp-actions | grep "circuit"
```

**Solutions**:

1. **Reset Circuit Breaker**
   ```bash
   # Restart action server
   kubectl rollout restart deployment/mcp-actions -n k8s-assistant
   ```

2. **Adjust Thresholds**
   ```yaml
   # mcp-servers/actions/config.yaml
   safety:
     circuit_breaker:
       enabled: true
       failure_threshold: 10  # Increase from 5
       timeout: 120s  # Increase from 60s
   ```

3. **Fix Underlying Issues**
   ```bash
   # Check why actions are failing
   kubectl logs -n k8s-assistant deployment/mcp-actions | grep "error"
   
   # Fix RBAC if needed
   kubectl apply -f config/rbac.yaml
   ```

---

### MCP Debugging Tools

#### Enable MCP Debug Mode

```bash
# Enable debug logging for all MCP servers
kubectl set env deployment -n k8s-assistant --all \
  MCP_DEBUG=true \
  LOG_LEVEL=debug

# Enable debug for specific server
kubectl set env deployment/mcp-k8s-monitor -n k8s-assistant \
  MCP_DEBUG=true \
  LOG_LEVEL=debug
```

#### MCP Request Tracing

```python
# Enable request tracing
import logging
logging.basicConfig(level=logging.DEBUG)

# Trace MCP calls
async with session:
    logger.debug(f"Calling tool: {tool_name}")
    result = await session.call_tool(tool_name, arguments)
    logger.debug(f"Result: {result}")
```

#### MCP Performance Profiling

```bash
# Enable profiling
kubectl set env deployment/mcp-k8s-monitor -n k8s-assistant \
  ENABLE_PROFILING=true \
  PROFILING_PORT=6060

# Port forward
kubectl port-forward -n k8s-assistant deployment/mcp-k8s-monitor 6060:6060

# View profiles
go tool pprof http://localhost:6060/debug/pprof/profile
go tool pprof http://localhost:6060/debug/pprof/heap
```

---

### MCP Diagnostic Commands

```bash
# List all MCP servers
k8s-assistant mcp list-servers

# Check server health
k8s-assistant mcp health-check --all

# List tools for a server
k8s-assistant mcp list-tools --server k8s-monitor

# Test tool execution
k8s-assistant mcp test-tool \
  --server k8s-monitor \
  --tool watch_pods \
  --args '{"namespace": "default"}'

# List resources
k8s-assistant mcp list-resources --server k8s-monitor

# Read resource
k8s-assistant mcp read-resource \
  --uri "pods://production/api-server-abc123"

# Get workflow status
k8s-assistant mcp get-workflow --id wf-123456

# View tool metrics
k8s-assistant mcp metrics --server k8s-monitor

# Validate configuration
k8s-assistant mcp validate-config --config config/mcp-servers.json
```

---

### Common MCP Error Codes

| Error Code | Description | Solution |
|------------|-------------|----------|
| -32700 | Parse error | Check JSON format in arguments |
| -32600 | Invalid request | Verify request structure |
| -32601 | Method not found | Check tool name spelling |
| -32602 | Invalid params | Verify argument types and required fields |
| -32603 | Internal error | Check server logs for details |
| -32001 | Server error | Restart MCP server |
| -32002 | Resource not found | Verify resource URI |
| -32003 | Permission denied | Check RBAC permissions |

---

### MCP Troubleshooting Checklist

- [ ] All MCP servers are running and healthy
- [ ] MCP server configuration is valid
- [ ] Environment variables are set correctly
- [ ] RBAC permissions are configured
- [ ] Network connectivity between components
- [ ] API keys are valid and not rate-limited
- [ ] Timeouts are appropriately configured
- [ ] Circuit breakers are not open
- [ ] Logs show no critical errors
- [ ] Resource limits are sufficient
- [ ] Cache is working properly
- [ ] Monitoring shows normal metrics

---

### Getting MCP-Specific Help

If you're still experiencing MCP-related issues:

1. **Collect MCP Diagnostics**:
   ```bash
   # Run MCP diagnostic script
   ./scripts/collect-mcp-diagnostics.sh
   
   # This creates a bundle with:
   # - MCP server logs
   # - Tool execution history
   # - Workflow traces
   # - Configuration files
   # - Performance metrics
   ```

2. **Check MCP Documentation**:
   - [MCP Specification](https://spec.modelcontextprotocol.io/)
   - [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
   - [Architecture Guide](../../ARCHITECTURE.md)

3. **Community Support**:
   - GitHub Issues: Tag with `mcp` label
   - Slack: #k8s-assistant-mcp channel
   - MCP Discord: https://discord.gg/mcp

4. **Debug Mode**:
   ```bash
   # Enable comprehensive debugging
   kubectl set env deployment -n k8s-assistant --all \
     MCP_DEBUG=true \
     LOG_LEVEL=debug \
     ENABLE_PROFILING=true \
     ENABLE_TRACING=true
   ```

   - Enterprise Support Portal: https://support.example.com