# API Documentation

## K8s Assistant REST API

The K8s Assistant provides a REST API for interacting with the system, querying incidents, and managing configurations.

---

## Base URL

```
http://k8s-assistant-api.k8s-assistant.svc.cluster.local:8080
```

For external access (with Ingress):
```
https://k8s-assistant.example.com
```

---

## Authentication

All API requests require authentication using Bearer tokens.

```bash
# Get token
TOKEN=$(kubectl create token k8s-assistant -n k8s-assistant)

# Use in requests
curl -H "Authorization: Bearer $TOKEN" \
  https://k8s-assistant.example.com/api/v1/incidents
```

---

## Endpoints

### Health Check

#### GET /health

Check API health status.

**Response:**
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp": "2024-01-15T10:30:00Z",
  "components": {
    "database": "healthy",
    "redis": "healthy",
    "llm": "healthy"
  }
}
```

---

### Incidents

#### GET /api/v1/incidents

List all incidents.

**Query Parameters:**
- `namespace` (optional): Filter by namespace
- `status` (optional): Filter by status (detected, analyzing, resolved, failed)
- `from` (optional): Start date (ISO 8601)
- `to` (optional): End date (ISO 8601)
- `limit` (optional): Number of results (default: 50)
- `offset` (optional): Pagination offset

**Example:**
```bash
curl -H "Authorization: Bearer $TOKEN" \
  "https://k8s-assistant.example.com/api/v1/incidents?namespace=production&limit=10"
```

**Response:**
```json
{
  "incidents": [
    {
      "id": "inc-123456",
      "type": "OOMKill",
      "namespace": "production",
      "pod_name": "api-server-abc123",
      "severity": "high",
      "status": "resolved",
      "detected_at": "2024-01-15T10:00:00Z",
      "resolved_at": "2024-01-15T10:05:00Z",
      "root_cause": "Memory limit too low for workload",
      "solution_applied": "Increased memory limit from 2Gi to 4Gi",
      "auto_fixed": true,
      "confidence": 0.92
    }
  ],
  "total": 1,
  "limit": 10,
  "offset": 0
}
```

#### GET /api/v1/incidents/{id}

Get incident details.

**Response:**
```json
{
  "id": "inc-123456",
  "type": "OOMKill",
  "namespace": "production",
  "pod_name": "api-server-abc123",
  "severity": "high",
  "status": "resolved",
  "detected_at": "2024-01-15T10:00:00Z",
  "resolved_at": "2024-01-15T10:05:00Z",
  "root_cause": "Memory limit too low for workload spike",
  "contributing_factors": [
    "Increased traffic during peak hours",
    "Memory-intensive operations"
  ],
  "solution_applied": {
    "id": "sol-789",
    "description": "Increase memory limit",
    "actions": [
      {
        "type": "patch_deployment",
        "target": "api-server",
        "parameters": {
          "memory_limit": "4Gi"
        }
      }
    ],
    "confidence": 0.92,
    "risk_level": "low"
  },
  "metrics": {
    "memory_usage_before": "3.9Gi",
    "memory_usage_after": "2.1Gi",
    "restart_count": 5
  },
  "logs": ["Last 100 log lines..."],
  "events": [
    {
      "timestamp": "2024-01-15T10:00:00Z",
      "type": "Warning",
      "reason": "OOMKilling",
      "message": "Memory limit exceeded"
    }
  ],
  "auto_fixed": true,
  "validation_result": "success"
}
```

#### POST /api/v1/incidents/{id}/feedback

Provide feedback on incident resolution.

**Request Body:**
```json
{
  "success": true,
  "comments": "Fix worked perfectly",
  "rating": 5
}
```

**Response:**
```json
{
  "message": "Feedback recorded",
  "incident_id": "inc-123456"
}
```

---

### Solutions

#### GET /api/v1/solutions

List available solutions.

**Response:**
```json
{
  "solutions": [
    {
      "id": "sol-oom-increase-memory",
      "issue_type": "OOMKill",
      "description": "Increase memory limit",
      "success_rate": 0.95,
      "average_confidence": 0.92,
      "times_used": 150
    }
  ]
}
```

---

### Configuration

#### GET /api/v1/config

Get current configuration.

**Response:**
```json
{
  "auto_fix_enabled": true,
  "confidence_threshold": 0.90,
  "rate_limit": {
    "max_per_minute": 10,
    "max_per_hour": 50
  },
  "notifications": {
    "slack": true,
    "email": true
  }
}
```

#### PATCH /api/v1/config

Update configuration.

**Request Body:**
```json
{
  "auto_fix_enabled": false,
  "confidence_threshold": 0.95
}
```

**Response:**
```json
{
  "message": "Configuration updated",
  "config": {
    "auto_fix_enabled": false,
    "confidence_threshold": 0.95
  }
}
```

---

### Metrics

#### GET /api/v1/metrics

Get system metrics.

**Response:**
```json
{
  "period": "24h",
  "incidents": {
    "total": 45,
    "by_type": {
      "OOMKill": 20,
      "Crash": 15,
      "HighLatency": 10
    },
    "by_status": {
      "resolved": 40,
      "failed": 3,
      "in_progress": 2
    }
  },
  "auto_fix": {
    "attempts": 35,
    "successes": 32,
    "success_rate": 0.91
  },
  "performance": {
    "mean_time_to_detect": "25s",
    "mean_time_to_resolution": "4m30s"
  }
}
```

---

## WebSocket API

For real-time updates, connect to the WebSocket endpoint.

### WS /api/v1/stream

**Connection:**
```javascript
const ws = new WebSocket('wss://k8s-assistant.example.com/api/v1/stream');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Event:', data);
};
```

**Event Types:**
- `incident.detected`
- `incident.analyzing`
- `incident.resolved`
- `incident.failed`
- `action.applied`
- `validation.completed`

**Example Event:**
```json
{
  "type": "incident.detected",
  "timestamp": "2024-01-15T10:00:00Z",
  "data": {
    "id": "inc-123456",
    "type": "OOMKill",
    "namespace": "production",
    "pod_name": "api-server-abc123"
  }
}
```

---

## Error Responses

All errors follow this format:

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Invalid namespace parameter",
    "details": {
      "field": "namespace",
      "reason": "Namespace does not exist"
    }
  }
}
```

**Error Codes:**
- `INVALID_REQUEST`: Invalid request parameters
- `UNAUTHORIZED`: Authentication failed
- `FORBIDDEN`: Insufficient permissions
- `NOT_FOUND`: Resource not found
- `RATE_LIMIT_EXCEEDED`: Too many requests
- `INTERNAL_ERROR`: Internal server error

---

## Rate Limiting

API requests are rate limited:
- 100 requests per minute per client
- 1000 requests per hour per client

Rate limit headers:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642248000
```

---

## SDK Examples

### Python

```python
from k8s_assistant import Client

client = Client(
    base_url="https://k8s-assistant.example.com",
    token="your-token"
)

# List incidents
incidents = client.incidents.list(namespace="production")

# Get incident details
incident = client.incidents.get("inc-123456")

# Provide feedback
client.incidents.feedback("inc-123456", success=True, rating=5)
```

### Go

```go
import "github.com/your-org/k8s-assistant-sdk-go"

client := k8sassistant.NewClient(
    "https://k8s-assistant.example.com",
    "your-token",
)

// List incidents
incidents, err := client.Incidents.List(&k8sassistant.IncidentListOptions{
    Namespace: "production",
})

// Get incident details
incident, err := client.Incidents.Get("inc-123456")
```

---

## Postman Collection

Download the Postman collection: [k8s-assistant.postman_collection.json](./k8s-assistant.postman_collection.json)

---

## OpenAPI Specification

View the full OpenAPI spec: [openapi.yaml](./openapi.yaml)