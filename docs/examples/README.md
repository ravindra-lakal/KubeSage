# Examples & Use Cases

## K8s Assistant Real-World Examples

This document provides practical examples and use cases for the K8s Assistant.

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
- [Performance Tuning](./performance-tuning.md)