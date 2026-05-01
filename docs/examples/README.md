# Examples & Use Cases

## K8s Assistant Real-World Examples (MCP Architecture)

This document provides practical examples and use cases for the K8s Assistant using the Model Context Protocol (MCP) architecture.

---

## Table of Contents

1. [Common Scenarios](#common-scenarios)
2. [Custom Analyzers](#custom-analyzers)
3. [Integration Examples](#integration-examples)
4. [Advanced Use Cases](#advanced-use-cases)

---

## Common Scenarios

### Scenario 1: OOMKill Auto-Remediation

**Problem**: Application pods are being OOMKilled during traffic spikes.

**Detection**:
```yaml
# The system detects:
- Event: OOMKilling
- Exit Code: 137
- Memory usage: 3.9Gi / 4Gi (97.5%)
- Restart count: 5 in 10 minutes
```

**AI Analysis**:
```
Root Cause: Memory limit (4Gi) is insufficient for peak workload
Contributing Factors:
  - Traffic increased 3x during peak hours
  - Memory-intensive JSON parsing operations
  - No memory profiling enabled
Impact: Service degradation, increased latency
```

**Solution Applied**:
```yaml
Action: Patch Deployment
Changes:
  - memory_limit: 4Gi → 6Gi
  - memory_request: 2Gi → 3Gi
Confidence: 92%
Risk: Low
```

**Result**:
```
✓ Pod restarted with new limits
✓ Memory usage stabilized at 3.2Gi (53%)
✓ No further OOMKills
✓ Latency returned to normal
Duration: 4 minutes 30 seconds
```

---

### Scenario 2: Database Connection Pool Exhaustion

**Problem**: API service experiencing high latency and timeouts.

**Detection**:
```yaml
# The system detects:
- P95 latency: 2000ms (baseline: 100ms)
- Error rate: 15% (baseline: 0.1%)
- Logs: "connection pool exhausted"
```

**AI Analysis**:
```
Root Cause: Database connection pool size (50) insufficient for current load
Contributing Factors:
  - Increased concurrent requests
  - Long-running queries holding connections
  - No connection timeout configured
Impact: 15% of requests failing, user-facing errors
```

**Solution Applied**:
```yaml
Action: Update ConfigMap
Changes:
  - max_connections: 50 → 100
  - connection_timeout: none → 30s
  - idle_timeout: none → 5m
Confidence: 85%
Risk: Low
```

**Result**:
```
✓ Connection pool expanded
✓ Latency reduced to 120ms
✓ Error rate dropped to 0.2%
✓ No connection exhaustion
Duration: 2 minutes
```

---

### Scenario 3: Memory Leak Detection

**Problem**: Gradual memory growth leading to eventual OOMKill.

**Detection**:
```yaml
# The system detects:
- Memory growth: 10% per hour
- Pattern: Linear increase over 6 hours
- Estimated time to OOM: 2 hours
```

**AI Analysis**:
```
Root Cause: Memory leak in request processing
Contributing Factors:
  - Objects not being garbage collected
  - Event listeners not removed
  - Cache growing unbounded
Impact: Eventual OOMKill, service restart required
```

**Solution Applied**:
```yaml
Action: Immediate restart + Enable profiling
Changes:
  - Restart pod to free memory
  - Enable memory profiling
  - Add heap dump on OOM
  - Create ticket for dev team
Confidence: 78%
Risk: Medium (requires restart)
```

**Result**:
```
✓ Pod restarted successfully
✓ Memory profiling enabled
✓ Heap dumps configured
✓ Dev ticket created: DEV-1234
✓ Monitoring for recurrence
Duration: 3 minutes
```

---

## Custom Analyzers

### Example 1: Redis Cache Hit Rate Analyzer

```python
# scripts/analyzers/redis_cache_analyzer.py
from k8s_assistant.analyzer import BaseAnalyzer
from typing import Dict, Optional

class RedisCacheAnalyzer(BaseAnalyzer):
    """Analyzes Redis cache hit rate and suggests optimizations"""
    
    def __init__(self, config: Dict):
        super().__init__(config)
        self.min_hit_rate = config.get('min_hit_rate', 0.7)
        self.check_interval = config.get('check_interval', 60)
    
    def analyze(self, context: Dict) -> Optional[Dict]:
        """Analyze cache performance"""
        metrics = context.get('metrics', {})
        
        # Calculate hit rate
        hits = metrics.get('redis_cache_hits', 0)
        misses = metrics.get('redis_cache_misses', 0)
        total = hits + misses
        
        if total == 0:
            return None
        
        hit_rate = hits / total
        
        if hit_rate < self.min_hit_rate:
            return {
                'issue_type': 'LowCacheHitRate',
                'severity': 'medium',
                'description': f'Cache hit rate is {hit_rate:.2%}, below threshold of {self.min_hit_rate:.2%}',
                'metrics': {
                    'hit_rate': hit_rate,
                    'hits': hits,
                    'misses': misses
                },
                'recommendations': [
                    'Review cache key patterns',
                    'Increase cache TTL',
                    'Add cache warming',
                    'Review cache size limits'
                ]
            }
        
        return None
    
    def generate_solution(self, analysis: Dict) -> Dict:
        """Generate solution for low cache hit rate"""
        hit_rate = analysis['metrics']['hit_rate']
        
        if hit_rate < 0.5:
            # Very low hit rate - increase cache size
            return {
                'description': 'Increase Redis cache size',
                'actions': [
                    {
                        'type': 'patch_configmap',
                        'target': 'redis-config',
                        'parameters': {
                            'maxmemory': '2gb'
                        }
                    }
                ],
                'confidence': 0.85,
                'risk_level': 'low'
            }
        else:
            # Moderate hit rate - adjust TTL
            return {
                'description': 'Optimize cache TTL settings',
                'actions': [
                    {
                        'type': 'patch_configmap',
                        'target': 'app-config',
                        'parameters': {
                            'cache_ttl': '3600'
                        }
                    }
                ],
                'confidence': 0.75,
                'risk_level': 'low'
            }
```

**Configuration**:
```yaml
# config/custom-analyzers.yaml
analyzers:
  custom:
    - name: redis_cache_analyzer
      enabled: true
      script: /scripts/analyzers/redis_cache_analyzer.py
      config:
        min_hit_rate: 0.7
        check_interval: 60
      schedule: "*/5 * * * *"  # Every 5 minutes
```

---

### Example 2: API Rate Limit Analyzer

```python
# scripts/analyzers/rate_limit_analyzer.py
from k8s_assistant.analyzer import BaseAnalyzer
from typing import Dict, Optional

class RateLimitAnalyzer(BaseAnalyzer):
    """Analyzes API rate limit usage and suggests adjustments"""
    
    def analyze(self, context: Dict) -> Optional[Dict]:
        """Analyze rate limit metrics"""
        metrics = context.get('metrics', {})
        
        # Get rate limit metrics
        requests = metrics.get('api_requests_total', 0)
        rate_limited = metrics.get('api_rate_limited_total', 0)
        
        if requests == 0:
            return None
        
        rate_limit_percentage = (rate_limited / requests) * 100
        
        if rate_limit_percentage > 5:  # More than 5% rate limited
            return {
                'issue_type': 'HighRateLimitRejection',
                'severity': 'high' if rate_limit_percentage > 20 else 'medium',
                'description': f'{rate_limit_percentage:.1f}% of requests are being rate limited',
                'metrics': {
                    'total_requests': requests,
                    'rate_limited': rate_limited,
                    'percentage': rate_limit_percentage
                },
                'recommendations': [
                    'Increase rate limits',
                    'Implement request queuing',
                    'Add caching layer',
                    'Scale horizontally'
                ]
            }
        
        return None
    
    def generate_solution(self, analysis: Dict) -> Dict:
        """Generate solution for high rate limiting"""
        percentage = analysis['metrics']['percentage']
        
        if percentage > 20:
            # Critical - increase limits significantly
            return {
                'description': 'Increase rate limits and scale horizontally',
                'actions': [
                    {
                        'type': 'patch_configmap',
                        'target': 'api-config',
                        'parameters': {
                            'rate_limit_per_minute': '1000',
                            'rate_limit_per_hour': '50000'
                        }
                    },
                    {
                        'type': 'scale_deployment',
                        'target': 'api-server',
                        'parameters': {
                            'replicas': '+2'
                        }
                    }
                ],
                'confidence': 0.88,
                'risk_level': 'low'
            }
        else:
            # Moderate - adjust limits
            return {
                'description': 'Increase rate limits',
                'actions': [
                    {
                        'type': 'patch_configmap',
                        'target': 'api-config',
                        'parameters': {
                            'rate_limit_per_minute': '500'
                        }
                    }
                ],
                'confidence': 0.82,
                'risk_level': 'low'
            }
```

---

## Integration Examples

### Example 1: Slack Integration with Custom Formatting

```python
# scripts/integrations/slack_custom.py
from slack_sdk import WebClient
from slack_sdk.errors import SlackApiError

class CustomSlackNotifier:
    def __init__(self, webhook_url: str):
        self.client = WebClient(token=webhook_url)
    
    def send_incident_alert(self, incident: Dict):
        """Send formatted incident alert to Slack"""
        
        # Determine color based on severity
        color_map = {
            'critical': '#FF0000',
            'high': '#FF6B00',
            'medium': '#FFB800',
            'low': '#00FF00'
        }
        color = color_map.get(incident['severity'], '#808080')
        
        # Build message blocks
        blocks = [
            {
                "type": "header",
                "text": {
                    "type": "plain_text",
                    "text": f"🚨 {incident['type']} Detected"
                }
            },
            {
                "type": "section",
                "fields": [
                    {
                        "type": "mrkdwn",
                        "text": f"*Namespace:*\n{incident['namespace']}"
                    },
                    {
                        "type": "mrkdwn",
                        "text": f"*Pod:*\n{incident['pod_name']}"
                    },
                    {
                        "type": "mrkdwn",
                        "text": f"*Severity:*\n{incident['severity'].upper()}"
                    },
                    {
                        "type": "mrkdwn",
                        "text": f"*Detected:*\n<!date^{int(incident['detected_at'].timestamp())}^{date_short_pretty} {time}|{incident['detected_at']}>"
                    }
                ]
            },
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*Root Cause:*\n{incident['root_cause']}"
                }
            }
        ]
        
        # Add solution if auto-fixed
        if incident.get('auto_fixed'):
            blocks.append({
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*✅ Auto-Fixed:*\n{incident['solution_applied']}"
                }
            })
        else:
            blocks.append({
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": "*⚠️ Manual intervention required*"
                }
            })
        
        # Add action buttons
        blocks.append({
            "type": "actions",
            "elements": [
                {
                    "type": "button",
                    "text": {
                        "type": "plain_text",
                        "text": "View Details"
                    },
                    "url": f"https://k8s-assistant.example.com/incidents/{incident['id']}"
                },
                {
                    "type": "button",
                    "text": {
                        "type": "plain_text",
                        "text": "View Logs"
                    },
                    "url": f"https://grafana.example.com/logs?pod={incident['pod_name']}"
                }
            ]
        })
        
        try:
            response = self.client.chat_postMessage(
                channel="#sre-alerts",
                blocks=blocks,
                attachments=[{
                    "color": color,
                    "fallback": f"{incident['type']} in {incident['namespace']}/{incident['pod_name']}"
                }]
            )
            return response
        except SlackApiError as e:
            print(f"Error sending Slack message: {e}")
```

---

### Example 2: PagerDuty Integration

```python
# scripts/integrations/pagerduty_custom.py
import requests
from typing import Dict

class PagerDutyIntegration:
    def __init__(self, integration_key: str):
        self.integration_key = integration_key
        self.api_url = "https://events.pagerduty.com/v2/enqueue"
    
    def create_incident(self, incident: Dict):
        """Create PagerDuty incident"""
        
        # Map severity to PagerDuty severity
        severity_map = {
            'critical': 'critical',
            'high': 'error',
            'medium': 'warning',
            'low': 'info'
        }
        
        payload = {
            "routing_key": self.integration_key,
            "event_action": "trigger",
            "dedup_key": f"{incident['namespace']}-{incident['pod_name']}-{incident['type']}",
            "payload": {
                "summary": f"{incident['type']} in {incident['namespace']}/{incident['pod_name']}",
                "severity": severity_map.get(incident['severity'], 'error'),
                "source": "k8s-assistant",
                "timestamp": incident['detected_at'].isoformat(),
                "component": incident['pod_name'],
                "group": incident['namespace'],
                "class": incident['type'],
                "custom_details": {
                    "root_cause": incident['root_cause'],
                    "metrics": incident.get('metrics', {}),
                    "auto_fixed": incident.get('auto_fixed', False),
                    "solution": incident.get('solution_applied', 'N/A')
                }
            },
            "links": [
                {
                    "href": f"https://k8s-assistant.example.com/incidents/{incident['id']}",
                    "text": "View in K8s Assistant"
                },
                {
                    "href": f"https://grafana.example.com/d/k8s-pod?var-pod={incident['pod_name']}",
                    "text": "View Metrics"
                }
            ]
        }
        
        response = requests.post(self.api_url, json=payload)
        return response.json()
    
    def resolve_incident(self, incident: Dict):
        """Resolve PagerDuty incident"""
        
        payload = {
            "routing_key": self.integration_key,
            "event_action": "resolve",
            "dedup_key": f"{incident['namespace']}-{incident['pod_name']}-{incident['type']}"
        }
        
        response = requests.post(self.api_url, json=payload)
        return response.json()
```

---

## Advanced Use Cases

### Use Case 1: Predictive Scaling

```python
# scripts/advanced/predictive_scaling.py
from k8s_assistant.predictor import BasePredictor
import numpy as np
from sklearn.linear_model import LinearRegression

class PredictiveScaler(BasePredictor):
    """Predicts future resource needs and scales proactively"""
    
    def __init__(self):
        self.model = LinearRegression()
        self.history_window = 24  # hours
    
    def predict_resource_needs(self, deployment: str, namespace: str) -> Dict:
        """Predict resource needs for next hour"""
        
        # Get historical metrics
        metrics = self.get_historical_metrics(
            deployment=deployment,
            namespace=namespace,
            duration=f"{self.history_window}h"
        )
        
        # Prepare data
        X = np.array(range(len(metrics))).reshape(-1, 1)
        y_cpu = np.array([m['cpu'] for m in metrics])
        y_memory = np.array([m['memory'] for m in metrics])
        
        # Train models
        self.model.fit(X, y_cpu)
        predicted_cpu = self.model.predict([[len(metrics) + 12]])[0]  # +1 hour
        
        self.model.fit(X, y_memory)
        predicted_memory = self.model.predict([[len(metrics) + 12]])[0]
        
        # Get current resources
        current = self.get_current_resources(deployment, namespace)
        
        # Calculate if scaling needed
        cpu_utilization = predicted_cpu / current['cpu_limit']
        memory_utilization = predicted_memory / current['memory_limit']
        
        if cpu_utilization > 0.8 or memory_utilization > 0.8:
            return {
                'scale_needed': True,
                'predicted_cpu': predicted_cpu,
                'predicted_memory': predicted_memory,
                'recommended_replicas': self.calculate_replicas(
                    cpu_utilization, memory_utilization
                ),
                'confidence': 0.75
            }
        
        return {'scale_needed': False}
```

---

### Use Case 2: Cost Optimization

```python
# scripts/advanced/cost_optimizer.py
from k8s_assistant.optimizer import BaseOptimizer

class CostOptimizer(BaseOptimizer):
    """Optimizes resource allocation for cost efficiency"""
    
    def analyze_resource_waste(self, namespace: str) -> Dict:
        """Identify over-provisioned resources"""
        
        deployments = self.get_deployments(namespace)
        recommendations = []
        
        for deployment in deployments:
            # Get actual usage vs limits
            usage = self.get_average_usage(deployment, duration="7d")
            limits = self.get_resource_limits(deployment)
            
            cpu_waste = (limits['cpu'] - usage['cpu']) / limits['cpu']
            memory_waste = (limits['memory'] - usage['memory']) / limits['memory']
            
            if cpu_waste > 0.5 or memory_waste > 0.5:
                # Significant over-provisioning
                recommendations.append({
                    'deployment': deployment['name'],
                    'current_limits': limits,
                    'actual_usage': usage,
                    'waste_percentage': {
                        'cpu': cpu_waste * 100,
                        'memory': memory_waste * 100
                    },
                    'recommended_limits': {
                        'cpu': usage['cpu'] * 1.3,  # 30% buffer
                        'memory': usage['memory'] * 1.3
                    },
                    'estimated_savings': self.calculate_savings(
                        limits, usage
                    )
                })
        
        return {
            'total_deployments': len(deployments),
            'over_provisioned': len(recommendations),
            'recommendations': recommendations,
            'total_estimated_savings': sum(
                r['estimated_savings'] for r in recommendations
            )
        }
```

---

## Testing Examples

### Example: Testing Custom Analyzer

```python
# tests/test_custom_analyzer.py
import pytest
from scripts.analyzers.redis_cache_analyzer import RedisCacheAnalyzer

def test_low_cache_hit_rate():
    """Test detection of low cache hit rate"""
    
    analyzer = RedisCacheAnalyzer({'min_hit_rate': 0.7})
    
    context = {
        'metrics': {
            'redis_cache_hits': 300,
            'redis_cache_misses': 700
        }
    }
    
    result = analyzer.analyze(context)
    
    assert result is not None
    assert result['issue_type'] == 'LowCacheHitRate'
    assert result['severity'] == 'medium'
    assert result['metrics']['hit_rate'] == 0.3

def test_acceptable_cache_hit_rate():
    """Test no alert for acceptable hit rate"""
    
    analyzer = RedisCacheAnalyzer({'min_hit_rate': 0.7})
    
    context = {
        'metrics': {
            'redis_cache_hits': 800,
            'redis_cache_misses': 200
        }
    }
    
    result = analyzer.analyze(context)
    
    assert result is None
```

---

## More Examples

For more examples, see:
- [Custom Detectors](./custom-detectors.md)
- [Integration Patterns](./integration-patterns.md)
- [Advanced Workflows](./advanced-workflows.md)

---

## MCP Workflow Examples

### Example 1: Complete OOMKill Resolution with MCP

This example shows the complete MCP workflow for detecting and resolving an OOMKill incident.

```python
# orchestrator/workflows/oomkill_resolution.py
from mcp import ClientSession, StdioServerParameters
from anthropic import Anthropic
import asyncio

async def resolve_oomkill_incident():
    """Complete MCP workflow for OOMKill resolution"""
    
    # Initialize MCP servers
    k8s_monitor = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/k8s-monitor/server.py"]
        )
    )
    
    metrics_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/metrics/server.py"]
        )
    )
    
    detection_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/detection/server.py"]
        )
    )
    
    kb_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/knowledge-base/server.py"]
        )
    )
    
    action_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/actions/server.py"]
        )
    )
    
    notify_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/notifications/server.py"]
        )
    )
    
    # Initialize Claude AI
    claude = Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
    
    async with k8s_monitor, metrics_server, detection_server, \
               kb_server, action_server, notify_server:
        
        # Step 1: Detect OOMKill event
        print("Step 1: Watching for pod events...")
        event_result = await k8s_monitor.call_tool(
            "watch_pods",
            arguments={
                "namespace": "production",
                "label_selector": "app=api-server"
            }
        )
        
        if event_result.content[0].text:
            event_data = json.loads(event_result.content[0].text)
            print(f"Detected: {event_data['reason']} in {event_data['pod_name']}")
        
        # Step 2: Query memory metrics
        print("Step 2: Querying memory metrics...")
        metrics_result = await metrics_server.call_tool(
            "query_metrics",
            arguments={
                "query": f"container_memory_usage_bytes{{pod='{event_data['pod_name']}'}}",
                "duration": "1h"
            }
        )
        
        metrics_data = json.loads(metrics_result.content[0].text)
        print(f"Memory usage: {metrics_data['current']} / {metrics_data['limit']}")
        
        # Step 3: Fetch logs
        print("Step 3: Fetching container logs...")
        logs_result = await k8s_monitor.call_tool(
            "fetch_logs",
            arguments={
                "namespace": "production",
                "pod_name": event_data['pod_name'],
                "tail_lines": 1000
            }
        )
        
        logs_data = json.loads(logs_result.content[0].text)
        
        # Step 4: Classify issue
        print("Step 4: Classifying issue...")
        classification_result = await detection_server.call_tool(
            "classify_issue",
            arguments={
                "context": {
                    "events": [event_data],
                    "metrics": metrics_data,
                    "logs": logs_data['logs'][:100]  # Last 100 lines
                }
            }
        )
        
        issue = json.loads(classification_result.content[0].text)
        print(f"Issue classified: {issue['type']} (confidence: {issue['confidence']})")
        
        # Step 5: Search for similar incidents
        print("Step 5: Searching knowledge base...")
        similar_result = await kb_server.call_tool(
            "search_similar",
            arguments={
                "issue": issue,
                "limit": 3
            }
        )
        
        similar_incidents = json.loads(similar_result.content[0].text)
        print(f"Found {len(similar_incidents['incidents'])} similar incidents")
        
        # Step 6: AI Analysis with Claude
        print("Step 6: Analyzing with Claude AI...")
        analysis_prompt = f"""
        Analyze this Kubernetes incident and provide a solution:
        
        Event: {json.dumps(event_data, indent=2)}
        Metrics: {json.dumps(metrics_data, indent=2)}
        Issue Classification: {json.dumps(issue, indent=2)}
        Similar Incidents: {json.dumps(similar_incidents, indent=2)}
        
        Provide:
        1. Root cause analysis
        2. Recommended solution
        3. Risk assessment
        4. Confidence level
        """
        
        ai_response = claude.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=2000,
            messages=[{"role": "user", "content": analysis_prompt}]
        )
        
        analysis = ai_response.content[0].text
        print(f"AI Analysis:\n{analysis}")
        
        # Step 7: Dry run the fix
        print("Step 7: Testing fix with dry-run...")
        dry_run_result = await action_server.call_tool(
            "dry_run",
            arguments={
                "action": "patch_resource",
                "kind": "deployment",
                "namespace": "production",
                "name": "api-server",
                "patch": {
                    "spec": {
                        "template": {
                            "spec": {
                                "containers": [{
                                    "name": "api",
                                    "resources": {
                                        "limits": {"memory": "6Gi"},
                                        "requests": {"memory": "3Gi"}
                                    }
                                }]
                            }
                        }
                    }
                }
            }
        )
        
        dry_run_data = json.loads(dry_run_result.content[0].text)
        if dry_run_data['success']:
            print("Dry run successful!")
        
        # Step 8: Apply the fix
        print("Step 8: Applying fix...")
        patch_result = await action_server.call_tool(
            "patch_resource",
            arguments={
                "kind": "deployment",
                "namespace": "production",
                "name": "api-server",
                "patch": {
                    "spec": {
                        "template": {
                            "spec": {
                                "containers": [{
                                    "name": "api",
                                    "resources": {
                                        "limits": {"memory": "6Gi"},
                                        "requests": {"memory": "3Gi"}
                                    }
                                }]
                            }
                        }
                    }
                }
            }
        )
        
        patch_data = json.loads(patch_result.content[0].text)
        print(f"Patch applied: {patch_data['message']}")
        
        # Step 9: Validate the fix
        print("Step 9: Validating fix...")
        await asyncio.sleep(60)  # Wait for pod to restart
        
        validation_result = await k8s_monitor.call_tool(
            "watch_pods",
            arguments={
                "namespace": "production",
                "label_selector": "app=api-server"
            }
        )
        
        validation_data = json.loads(validation_result.content[0].text)
        if validation_data['status'] == 'Running':
            print("Validation successful! Pod is healthy.")
        
        # Step 10: Store incident in knowledge base
        print("Step 10: Storing incident...")
        store_result = await kb_server.call_tool(
            "store_incident",
            arguments={
                "incident": {
                    "type": issue['type'],
                    "namespace": "production",
                    "pod_name": event_data['pod_name'],
                    "root_cause": "Memory limit too low",
                    "solution": "Increased memory limit from 4Gi to 6Gi",
                    "confidence": issue['confidence'],
                    "auto_fixed": True
                }
            }
        )
        
        print("Incident stored in knowledge base")
        
        # Step 11: Send notification
        print("Step 11: Sending notification...")
        notify_result = await notify_server.call_tool(
            "send_slack",
            arguments={
                "message": f"✅ Auto-fixed OOMKill in {event_data['pod_name']}\n"
                          f"Solution: Increased memory limit to 6Gi\n"
                          f"Confidence: {issue['confidence']*100:.0f}%",
                "channel": "#sre-alerts"
            }
        )
        
        print("Notification sent!")
        print("\n✅ Incident resolution complete!")

# Run the workflow
if __name__ == "__main__":
    asyncio.run(resolve_oomkill_incident())
```

---

### Example 2: MCP-Based Custom Monitoring Workflow

```python
# scripts/custom_workflows/database_monitoring.py
from mcp import ClientSession, StdioServerParameters
import asyncio
import json

async def monitor_database_connections():
    """Monitor database connection pool using MCP servers"""
    
    metrics_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/metrics/server.py"]
        )
    )
    
    detection_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/detection/server.py"]
        )
    )
    
    action_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/actions/server.py"]
        )
    )
    
    async with metrics_server, detection_server, action_server:
        while True:
            # Query database connection metrics
            conn_metrics = await metrics_server.call_tool(
                "query_metrics",
                arguments={
                    "query": "pg_stat_activity_count",
                    "duration": "5m"
                }
            )
            
            metrics_data = json.loads(conn_metrics.content[0].text)
            current_connections = metrics_data['value']
            max_connections = 100
            
            utilization = current_connections / max_connections
            
            if utilization > 0.8:
                print(f"⚠️  High connection pool utilization: {utilization*100:.1f}%")
                
                # Detect if this is an anomaly
                anomaly_result = await detection_server.call_tool(
                    "detect_anomaly",
                    arguments={
                        "metrics": metrics_data['timeseries'],
                        "threshold": 2.0
                    }
                )
                
                anomaly_data = json.loads(anomaly_result.content[0].text)
                
                if anomaly_data['is_anomaly']:
                    print("🚨 Anomaly detected! Taking action...")
                    
                    # Scale up connection pool
                    await action_server.call_tool(
                        "patch_resource",
                        arguments={
                            "kind": "configmap",
                            "namespace": "production",
                            "name": "database-config",
                            "patch": {
                                "data": {
                                    "max_connections": "150"
                                }
                            }
                        }
                    )
                    
                    print("✅ Connection pool scaled to 150")
            
            await asyncio.sleep(60)  # Check every minute

if __name__ == "__main__":
    asyncio.run(monitor_database_connections())
```

---

### Example 3: Multi-Server MCP Tool Composition

```python
# scripts/workflows/comprehensive_health_check.py
from mcp import ClientSession, StdioServerParameters
import asyncio
import json

async def comprehensive_health_check(namespace: str):
    """Perform comprehensive health check using multiple MCP servers"""
    
    # Initialize all MCP servers
    servers = {
        'k8s_monitor': ClientSession(
            StdioServerParameters(
                command="python",
                args=["mcp-servers/k8s-monitor/server.py"]
            )
        ),
        'metrics': ClientSession(
            StdioServerParameters(
                command="python",
                args=["mcp-servers/metrics/server.py"]
            )
        ),
        'detection': ClientSession(
            StdioServerParameters(
                command="python",
                args=["mcp-servers/detection/server.py"]
            )
        )
    }
    
    async with servers['k8s_monitor'], servers['metrics'], servers['detection']:
        health_report = {
            'namespace': namespace,
            'timestamp': datetime.now().isoformat(),
            'checks': []
        }
        
        # Check 1: Pod Health
        print("Checking pod health...")
        pods_result = await servers['k8s_monitor'].call_tool(
            "watch_pods",
            arguments={"namespace": namespace}
        )
        
        pods_data = json.loads(pods_result.content[0].text)
        unhealthy_pods = [p for p in pods_data['pods'] if p['status'] != 'Running']
        
        health_report['checks'].append({
            'name': 'Pod Health',
            'status': 'healthy' if len(unhealthy_pods) == 0 else 'unhealthy',
            'details': {
                'total_pods': len(pods_data['pods']),
                'unhealthy_pods': len(unhealthy_pods),
                'unhealthy_list': unhealthy_pods
            }
        })
        
        # Check 2: Resource Utilization
        print("Checking resource utilization...")
        cpu_result = await servers['metrics'].call_tool(
            "query_metrics",
            arguments={
                "query": f"sum(rate(container_cpu_usage_seconds_total{{namespace='{namespace}'}}[5m]))",
                "duration": "5m"
            }
        )
        
        memory_result = await servers['metrics'].call_tool(
            "query_metrics",
            arguments={
                "query": f"sum(container_memory_usage_bytes{{namespace='{namespace}'}})",
                "duration": "5m"
            }
        )
        
        cpu_data = json.loads(cpu_result.content[0].text)
        memory_data = json.loads(memory_result.content[0].text)
        
        health_report['checks'].append({
            'name': 'Resource Utilization',
            'status': 'healthy',
            'details': {
                'cpu_usage': cpu_data['value'],
                'memory_usage': memory_data['value']
            }
        })
        
        # Check 3: Anomaly Detection
        print("Running anomaly detection...")
        anomaly_result = await servers['detection'].call_tool(
            "detect_anomaly",
            arguments={
                "metrics": memory_data['timeseries'],
                "threshold": 2.0
            }
        )
        
        anomaly_data = json.loads(anomaly_result.content[0].text)
        
        health_report['checks'].append({
            'name': 'Anomaly Detection',
            'status': 'healthy' if not anomaly_data['is_anomaly'] else 'warning',
            'details': anomaly_data
        })
        
        # Check 4: Event Analysis
        print("Analyzing cluster events...")
        events_result = await servers['k8s_monitor'].call_tool(
            "get_events",
            arguments={
                "namespace": namespace,
                "limit": 100
            }
        )
        
        events_data = json.loads(events_result.content[0].text)
        warning_events = [e for e in events_data['events'] if e['type'] == 'Warning']
        
        health_report['checks'].append({
            'name': 'Cluster Events',
            'status': 'healthy' if len(warning_events) == 0 else 'warning',
            'details': {
                'total_events': len(events_data['events']),
                'warning_events': len(warning_events),
                'recent_warnings': warning_events[:5]
            }
        })
        
        # Generate overall status
        statuses = [check['status'] for check in health_report['checks']]
        if 'unhealthy' in statuses:
            health_report['overall_status'] = 'unhealthy'
        elif 'warning' in statuses:
            health_report['overall_status'] = 'warning'
        else:
            health_report['overall_status'] = 'healthy'
        
        print(f"\n{'='*60}")
        print(f"Health Check Report for namespace: {namespace}")
        print(f"Overall Status: {health_report['overall_status'].upper()}")
        print(f"{'='*60}\n")
        
        for check in health_report['checks']:
            print(f"✓ {check['name']}: {check['status']}")
            if check['details']:
                for key, value in check['details'].items():
                    if not isinstance(value, list):
                        print(f"  - {key}: {value}")
        
        return health_report

if __name__ == "__main__":
    report = asyncio.run(comprehensive_health_check("production"))
    
    # Save report
    with open(f"health_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json", 'w') as f:
        json.dump(report, f, indent=2)
```

---

### Example 4: MCP Resource Reading Pattern

```python
# scripts/examples/resource_reading.py
from mcp import ClientSession, StdioServerParameters
import asyncio

async def read_mcp_resources():
    """Example of reading MCP resources"""
    
    k8s_monitor = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/k8s-monitor/server.py"]
        )
    )
    
    kb_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/knowledge-base/server.py"]
        )
    )
    
    async with k8s_monitor, kb_server:
        # Read pod resource
        pod_resource = await k8s_monitor.read_resource(
            "pods://production/api-server-abc123"
        )
        print("Pod Details:")
        print(json.dumps(json.loads(pod_resource.contents[0].text), indent=2))
        
        # Read events resource
        events_resource = await k8s_monitor.read_resource(
            "events://production"
        )
        print("\nRecent Events:")
        print(json.dumps(json.loads(events_resource.contents[0].text), indent=2))
        
        # Read logs resource
        logs_resource = await k8s_monitor.read_resource(
            "logs://production/api-server-abc123/api"
        )
        print("\nContainer Logs:")
        print(logs_resource.contents[0].text)
        
        # Read incidents from knowledge base
        incidents_resource = await kb_server.read_resource(
            "incidents://"
        )
        print("\nHistorical Incidents:")
        incidents = json.loads(incidents_resource.contents[0].text)
        for incident in incidents['incidents'][:5]:
            print(f"  - {incident['type']}: {incident['root_cause']}")

if __name__ == "__main__":
    asyncio.run(read_mcp_resources())
```

---

### Example 5: MCP Prompt Usage

```python
# scripts/examples/prompt_usage.py
from mcp import ClientSession, StdioServerParameters
import asyncio

async def use_mcp_prompts():
    """Example of using MCP prompts for analysis"""
    
    detection_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/detection/server.py"]
        )
    )
    
    async with detection_server:
        # List available prompts
        prompts = await detection_server.list_prompts()
        print("Available Prompts:")
        for prompt in prompts.prompts:
            print(f"  - {prompt.name}: {prompt.description}")
        
        # Use OOM analysis prompt
        oom_prompt = await detection_server.get_prompt(
            "analyze_oom",
            arguments={
                "pod_name": "api-server-abc123",
                "namespace": "production",
                "memory_limit": "4Gi",
                "memory_usage": "3.9Gi"
            }
        )
        
        print("\nOOM Analysis Prompt:")
        print(oom_prompt.messages[0].content.text)
        
        # Use crash analysis prompt
        crash_prompt = await detection_server.get_prompt(
            "analyze_crash",
            arguments={
                "pod_name": "worker-xyz789",
                "namespace": "production",
                "exit_code": "1",
                "restart_count": "5"
            }
        )
        
        print("\nCrash Analysis Prompt:")
        print(crash_prompt.messages[0].content.text)

if __name__ == "__main__":
    asyncio.run(use_mcp_prompts())
```

---

### Example 6: Error Handling in MCP Workflows

```python
# scripts/examples/error_handling.py
from mcp import ClientSession, StdioServerParameters
from mcp.types import McpError
import asyncio
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

async def robust_mcp_workflow():
    """Example of robust error handling in MCP workflows"""
    
    action_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/actions/server.py"]
        )
    )
    
    notify_server = ClientSession(
        StdioServerParameters(
            command="python",
            args=["mcp-servers/notifications/server.py"]
        )
    )
    
    async with action_server, notify_server:
        try:
            # Attempt to patch deployment
            logger.info("Attempting to patch deployment...")
            result = await action_server.call_tool(
                "patch_resource",
                arguments={
                    "kind": "deployment",
                    "namespace": "production",
                    "name": "api-server",
                    "patch": {
                        "spec": {
                            "replicas": 5
                        }
                    }
                }
            )
            
            logger.info("Patch successful!")
            
        except McpError as e:
            logger.error(f"MCP Error: {e.error.message}")
            
            # Attempt rollback
            try:
                logger.info("Attempting rollback...")
                rollback_result = await action_server.call_tool(
                    "rollback_deployment",
                    arguments={
                        "namespace": "production",
                        "name": "api-server"
                    }
                )
                logger.info("Rollback successful!")
                
            except McpError as rollback_error:
                logger.error(f"Rollback failed: {rollback_error.error.message}")
                
                # Send critical alert
                await notify_server.call_tool(
                    "send_slack",
                    arguments={
                        "message": f"🚨 CRITICAL: Failed to patch and rollback deployment\n"
                                  f"Error: {e.error.message}\n"
                                  f"Rollback Error: {rollback_error.error.message}",
                        "channel": "#sre-critical"
                    }
                )
        
        except Exception as e:
            logger.error(f"Unexpected error: {str(e)}")
            
            # Send error notification
            await notify_server.call_tool(
                "send_slack",
                arguments={
                    "message": f"⚠️  Unexpected error in workflow: {str(e)}",
                    "channel": "#sre-alerts"
                }
            )

if __name__ == "__main__":
    asyncio.run(robust_mcp_workflow())
```

---

### Example 7: MCP Server Health Monitoring

```python
# scripts/monitoring/mcp_server_health.py
from mcp import ClientSession, StdioServerParameters
import asyncio
import time

async def monitor_mcp_servers():
    """Monitor health of all MCP servers"""
    
    servers_config = {
        'k8s-monitor': ["mcp-servers/k8s-monitor/server.py"],
        'metrics': ["mcp-servers/metrics/server.py"],
        'detection': ["mcp-servers/detection/server.py"],
        'actions': ["mcp-servers/actions/server.py"],
        'knowledge-base': ["mcp-servers/knowledge-base/server.py"],
        'notifications': ["mcp-servers/notifications/server.py"]
    }
    
    while True:
        print(f"\n{'='*60}")
        print(f"MCP Server Health Check - {time.strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"{'='*60}\n")
        
        for server_name, args in servers_config.items():
            try:
                session = ClientSession(
                    StdioServerParameters(
                        command="python",
                        args=args
                    )
                )
                
                async with session:
                    # Initialize and check server info
                    start_time = time.time()
                    server_info = await session.initialize()
                    response_time = (time.time() - start_time) * 1000
                    
                    # List tools to verify server is responsive
                    tools = await session.list_tools()
                    
                    print(f"✅ {server_name:20} | "
                          f"Status: HEALTHY | "
                          f"Tools: {len(tools.tools):2} | "
                          f"Response: {response_time:6.2f}ms")
                    
            except Exception as e:
                print(f"❌ {server_name:20} | "
                      f"Status: UNHEALTHY | "
                      f"Error: {str(e)[:40]}")
        
        await asyncio.sleep(30)  # Check every 30 seconds

if __name__ == "__main__":
    asyncio.run(monitor_mcp_servers())
```

---

## MCP Integration Patterns

### Pattern 1: Sequential Tool Execution

```python
async def sequential_pattern(session):
    """Execute tools sequentially"""
    result1 = await session.call_tool("tool1", {"arg": "value"})
    result2 = await session.call_tool("tool2", {"input": result1.content[0].text})
    result3 = await session.call_tool("tool3", {"data": result2.content[0].text})
    return result3
```

### Pattern 2: Parallel Tool Execution

```python
async def parallel_pattern(session1, session2, session3):
    """Execute tools in parallel"""
    results = await asyncio.gather(
        session1.call_tool("tool1", {"arg": "value"}),
        session2.call_tool("tool2", {"arg": "value"}),
        session3.call_tool("tool3", {"arg": "value"})
    )
    return results
```

### Pattern 3: Conditional Execution

```python
async def conditional_pattern(session):
    """Execute tools conditionally"""
    check_result = await session.call_tool("check_condition", {})
    condition = json.loads(check_result.content[0].text)
    
    if condition['should_proceed']:
        return await session.call_tool("action_tool", {"data": condition['data']})
    else:
        return await session.call_tool("alternative_tool", {})
```

### Pattern 4: Retry with Backoff

```python
async def retry_pattern(session, max_retries=3):
    """Execute tool with retry logic"""
    for attempt in range(max_retries):
        try:
            result = await session.call_tool("unreliable_tool", {})
            return result
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(2 ** attempt)  # Exponential backoff
```

---

## Testing MCP Workflows

### Unit Test Example

```python
# tests/test_mcp_workflows.py
import pytest
from unittest.mock import Mock, AsyncMock
from workflows.oomkill_resolution import resolve_oomkill_incident

@pytest.mark.asyncio
async def test_oomkill_workflow():
    """Test OOMKill resolution workflow"""
    
    # Mock MCP servers
    mock_k8s_monitor = Mock()
    mock_k8s_monitor.call_tool = AsyncMock(return_value=Mock(
        content=[Mock(text='{"pod_name": "test-pod", "reason": "OOMKilled"}')]
    ))
    
    mock_metrics = Mock()
    mock_metrics.call_tool = AsyncMock(return_value=Mock(
        content=[Mock(text='{"current": 3900000000, "limit": 4000000000}')]
    ))
    
    # Run workflow with mocks
    result = await resolve_oomkill_incident(
        k8s_monitor=mock_k8s_monitor,
        metrics_server=mock_metrics
    )
    
    assert result['success'] == True
    assert result['action'] == 'patch_applied'
```

- [Performance Tuning](./performance-tuning.md)