# API Documentation

## K8s Assistant REST API (MCP Architecture)

The K8s Assistant provides a REST API for interacting with the system, querying incidents, managing configurations, and monitoring MCP servers.

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


---

## MCP Server Endpoints

### MCP Server Health

#### GET /api/v1/mcp/servers

List all MCP servers and their status.

**Response:**
```json
{
  "servers": [
    {
      "name": "k8s-monitor",
      "status": "healthy",
      "uptime": "24h15m",
      "tools_count": 4,
      "resources_count": 3,
      "last_heartbeat": "2024-01-15T10:30:00Z",
      "version": "1.0.0"
    },
    {
      "name": "metrics",
      "status": "healthy",
      "uptime": "24h15m",
      "tools_count": 4,
      "resources_count": 3,
      "last_heartbeat": "2024-01-15T10:30:00Z",
      "version": "1.0.0"
    },
    {
      "name": "detection",
      "status": "healthy",
      "uptime": "24h15m",
      "tools_count": 4,
      "prompts_count": 3,
      "last_heartbeat": "2024-01-15T10:30:00Z",
      "version": "1.0.0"
    },
    {
      "name": "actions",
      "status": "healthy",
      "uptime": "24h15m",
      "tools_count": 5,
      "last_heartbeat": "2024-01-15T10:30:00Z",
      "version": "1.0.0"
    },
    {
      "name": "knowledge-base",
      "status": "healthy",
      "uptime": "24h15m",
      "tools_count": 4,
      "resources_count": 3,
      "last_heartbeat": "2024-01-15T10:30:00Z",
      "version": "1.0.0"
    },
    {
      "name": "notifications",
      "status": "healthy",
      "uptime": "24h15m",
      "tools_count": 4,
      "last_heartbeat": "2024-01-15T10:30:00Z",
      "version": "1.0.0"
    }
  ],
  "total": 6,
  "healthy": 6,
  "unhealthy": 0
}
```

#### GET /api/v1/mcp/servers/{server_name}

Get detailed information about a specific MCP server.

**Response:**
```json
{
  "name": "k8s-monitor",
  "status": "healthy",
  "uptime": "24h15m",
  "version": "1.0.0",
  "last_heartbeat": "2024-01-15T10:30:00Z",
  "capabilities": {
    "tools": [
      {
        "name": "watch_pods",
        "description": "Stream pod status changes",
        "parameters": {
          "namespace": "string",
          "label_selector": "string (optional)"
        }
      },
      {
        "name": "get_events",
        "description": "Retrieve cluster events",
        "parameters": {
          "namespace": "string",
          "limit": "integer (optional)"
        }
      },
      {
        "name": "fetch_logs",
        "description": "Get container logs",
        "parameters": {
          "namespace": "string",
          "pod_name": "string",
          "container": "string (optional)",
          "tail_lines": "integer (optional)"
        }
      },
      {
        "name": "describe_resource",
        "description": "Get detailed resource information",
        "parameters": {
          "kind": "string",
          "namespace": "string",
          "name": "string"
        }
      }
    ],
    "resources": [
      {
        "uri": "pods://{namespace}/{pod-name}",
        "description": "Pod details",
        "mime_type": "application/json"
      },
      {
        "uri": "events://{namespace}",
        "description": "Cluster events",
        "mime_type": "application/json"
      },
      {
        "uri": "logs://{namespace}/{pod-name}/{container}",
        "description": "Container logs",
        "mime_type": "text/plain"
      }
    ]
  },
  "metrics": {
    "tool_calls_total": 1523,
    "tool_calls_success": 1498,
    "tool_calls_failed": 25,
    "average_response_time_ms": 145,
    "resource_reads_total": 892
  }
}
```

#### GET /api/v1/mcp/servers/{server_name}/tools

List all tools provided by a specific MCP server.

**Response:**
```json
{
  "server": "detection",
  "tools": [
    {
      "name": "detect_anomaly",
      "description": "Detect anomalous behavior in metrics",
      "input_schema": {
        "type": "object",
        "properties": {
          "metrics": {
            "type": "array",
            "description": "Time series metrics data"
          },
          "threshold": {
            "type": "number",
            "description": "Anomaly detection threshold"
          }
        },
        "required": ["metrics"]
      }
    },
    {
      "name": "classify_issue",
      "description": "Classify issue type from context",
      "input_schema": {
        "type": "object",
        "properties": {
          "context": {
            "type": "object",
            "description": "Issue context including events, logs, metrics"
          }
        },
        "required": ["context"]
      }
    }
  ]
}
```

#### POST /api/v1/mcp/servers/{server_name}/tools/{tool_name}

Call a tool on a specific MCP server.

**Request Body:**
```json
{
  "arguments": {
    "namespace": "production",
    "pod_name": "api-server-abc123"
  }
}
```

**Response:**
```json
{
  "tool": "fetch_logs",
  "server": "k8s-monitor",
  "result": {
    "logs": [
      "2024-01-15 10:00:00 INFO Starting application",
      "2024-01-15 10:00:01 INFO Connected to database",
      "2024-01-15 10:00:02 ERROR Out of memory"
    ],
    "lines": 3,
    "truncated": false
  },
  "execution_time_ms": 234,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

### MCP Workflows

#### GET /api/v1/mcp/workflows

List active MCP workflows.

**Response:**
```json
{
  "workflows": [
    {
      "id": "wf-123456",
      "type": "incident_resolution",
      "status": "in_progress",
      "incident_id": "inc-123456",
      "started_at": "2024-01-15T10:00:00Z",
      "steps": [
        {
          "step": 1,
          "server": "k8s-monitor",
          "tool": "watch_pods",
          "status": "completed",
          "duration_ms": 150
        },
        {
          "step": 2,
          "server": "metrics",
          "tool": "query_metrics",
          "status": "completed",
          "duration_ms": 320
        },
        {
          "step": 3,
          "server": "detection",
          "tool": "classify_issue",
          "status": "in_progress",
          "duration_ms": null
        }
      ]
    }
  ],
  "total": 1,
  "in_progress": 1,
  "completed": 0,
  "failed": 0
}
```

#### GET /api/v1/mcp/workflows/{workflow_id}

Get detailed workflow execution information.

**Response:**
```json
{
  "id": "wf-123456",
  "type": "incident_resolution",
  "status": "completed",
  "incident_id": "inc-123456",
  "started_at": "2024-01-15T10:00:00Z",
  "completed_at": "2024-01-15T10:05:00Z",
  "duration_ms": 300000,
  "steps": [
    {
      "step": 1,
      "description": "Detect pod crash",
      "server": "k8s-monitor",
      "tool": "watch_pods",
      "arguments": {
        "namespace": "production"
      },
      "result": {
        "pod_name": "api-server-abc123",
        "status": "OOMKilled",
        "exit_code": 137
      },
      "status": "completed",
      "duration_ms": 150,
      "timestamp": "2024-01-15T10:00:00Z"
    },
    {
      "step": 2,
      "description": "Query memory metrics",
      "server": "metrics",
      "tool": "query_metrics",
      "arguments": {
        "query": "container_memory_usage_bytes{pod='api-server-abc123'}"
      },
      "result": {
        "value": 3.9e9,
        "limit": 4e9,
        "utilization": 0.975
      },
      "status": "completed",
      "duration_ms": 320,
      "timestamp": "2024-01-15T10:00:01Z"
    },
    {
      "step": 3,
      "description": "Classify issue",
      "server": "detection",
      "tool": "classify_issue",
      "arguments": {
        "context": {
          "events": ["OOMKilling"],
          "metrics": {"memory_usage": 0.975}
        }
      },
      "result": {
        "issue_type": "OOMKill",
        "confidence": 0.95,
        "severity": "high"
      },
      "status": "completed",
      "duration_ms": 890,
      "timestamp": "2024-01-15T10:00:02Z"
    },
    {
      "step": 4,
      "description": "Search similar incidents",
      "server": "knowledge-base",
      "tool": "search_similar",
      "arguments": {
        "issue": {
          "type": "OOMKill",
          "pod": "api-server-abc123"
        },
        "limit": 3
      },
      "result": {
        "similar_incidents": [
          {
            "id": "inc-111111",
            "similarity": 0.92,
            "solution": "Increase memory limit"
          }
        ]
      },
      "status": "completed",
      "duration_ms": 450,
      "timestamp": "2024-01-15T10:00:03Z"
    },
    {
      "step": 5,
      "description": "Apply fix",
      "server": "actions",
      "tool": "patch_resource",
      "arguments": {
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
                    "limits": {"memory": "6Gi"}
                  }
                }]
              }
            }
          }
        }
      },
      "result": {
        "success": true,
        "message": "Deployment patched successfully"
      },
      "status": "completed",
      "duration_ms": 1200,
      "timestamp": "2024-01-15T10:00:04Z"
    },
    {
      "step": 6,
      "description": "Send notification",
      "server": "notifications",
      "tool": "send_slack",
      "arguments": {
        "message": "Auto-fixed OOMKill in api-server",
        "channel": "#sre-alerts"
      },
      "result": {
        "success": true,
        "message_id": "1234567890.123456"
      },
      "status": "completed",
      "duration_ms": 180,
      "timestamp": "2024-01-15T10:00:05Z"
    }
  ],
  "ai_analysis": {
    "model": "claude-3-5-sonnet-20241022",
    "tokens_used": 2450,
    "analysis_time_ms": 3200,
    "confidence": 0.92
  }
}
```

---

### MCP Tool Usage Metrics

#### GET /api/v1/mcp/metrics/tools

Get tool usage statistics across all MCP servers.

**Query Parameters:**
- `from` (optional): Start date (ISO 8601)
- `to` (optional): End date (ISO 8601)
- `server` (optional): Filter by server name

**Response:**
```json
{
  "period": {
    "from": "2024-01-14T10:00:00Z",
    "to": "2024-01-15T10:00:00Z"
  },
  "total_calls": 5432,
  "successful_calls": 5321,
  "failed_calls": 111,
  "success_rate": 0.98,
  "average_response_time_ms": 245,
  "by_server": {
    "k8s-monitor": {
      "calls": 1523,
      "success_rate": 0.98,
      "avg_response_time_ms": 145
    },
    "metrics": {
      "calls": 1234,
      "success_rate": 0.99,
      "avg_response_time_ms": 320
    },
    "detection": {
      "calls": 892,
      "success_rate": 0.97,
      "avg_response_time_ms": 890
    },
    "actions": {
      "calls": 456,
      "success_rate": 0.95,
      "avg_response_time_ms": 1200
    },
    "knowledge-base": {
      "calls": 789,
      "success_rate": 0.99,
      "avg_response_time_ms": 450
    },
    "notifications": {
      "calls": 538,
      "success_rate": 0.99,
      "avg_response_time_ms": 180
    }
  },
  "top_tools": [
    {
      "server": "k8s-monitor",
      "tool": "watch_pods",
      "calls": 892,
      "success_rate": 0.98
    },
    {
      "server": "metrics",
      "tool": "query_metrics",
      "calls": 756,
      "success_rate": 0.99
    },
    {
      "server": "detection",
      "tool": "classify_issue",
      "calls": 445,
      "success_rate": 0.97
    }
  ]
}
```

---

### MCP Resources

#### GET /api/v1/mcp/resources

List available MCP resources across all servers.

**Response:**
```json
{
  "resources": [
    {
      "server": "k8s-monitor",
      "uri": "pods://production/api-server-abc123",
      "description": "Pod details for api-server-abc123",
      "mime_type": "application/json",
      "last_accessed": "2024-01-15T10:30:00Z"
    },
    {
      "server": "k8s-monitor",
      "uri": "events://production",
      "description": "Events in production namespace",
      "mime_type": "application/json",
      "last_accessed": "2024-01-15T10:29:00Z"
    },
    {
      "server": "knowledge-base",
      "uri": "incidents://",
      "description": "Historical incidents database",
      "mime_type": "application/json",
      "last_accessed": "2024-01-15T10:28:00Z"
    }
  ],
  "total": 3
}
```

#### GET /api/v1/mcp/resources/{uri}

Read a specific MCP resource.

**Example:**
```bash
curl -H "Authorization: Bearer $TOKEN" \
  "https://k8s-assistant.example.com/api/v1/mcp/resources/pods%3A%2F%2Fproduction%2Fapi-server-abc123"
```

**Response:**
```json
{
  "uri": "pods://production/api-server-abc123",
  "server": "k8s-monitor",
  "mime_type": "application/json",
  "content": {
    "name": "api-server-abc123",
    "namespace": "production",
    "status": "Running",
    "containers": [
      {
        "name": "api",
        "image": "api-server:v1.2.3",
        "resources": {
          "limits": {
            "cpu": "1",
            "memory": "4Gi"
          },
          "requests": {
            "cpu": "500m",
            "memory": "2Gi"
          }
        }
      }
    ]
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## MCP Client SDK

### Python SDK

```python
from k8s_assistant.mcp import MCPClient

# Initialize MCP client
client = MCPClient(
    base_url="https://k8s-assistant.example.com",
    token="your-token"
)

# List MCP servers
servers = client.servers.list()
for server in servers:
    print(f"{server.name}: {server.status}")

# Call a tool
result = client.tools.call(
    server="k8s-monitor",
    tool="fetch_logs",
    arguments={
        "namespace": "production",
        "pod_name": "api-server-abc123",
        "tail_lines": 100
    }
)
print(result.logs)

# Read a resource
resource = client.resources.read("pods://production/api-server-abc123")
print(resource.content)

# Get workflow status
workflow = client.workflows.get("wf-123456")
print(f"Status: {workflow.status}")
for step in workflow.steps:
    print(f"  Step {step.step}: {step.tool} - {step.status}")
```

### Go SDK

```go
import "github.com/your-org/k8s-assistant-sdk-go/mcp"

// Initialize MCP client
client := mcp.NewClient(
    "https://k8s-assistant.example.com",
    "your-token",
)

// List MCP servers
servers, err := client.Servers.List(ctx)
if err != nil {
    log.Fatal(err)
}

// Call a tool
result, err := client.Tools.Call(ctx, &mcp.ToolCallRequest{
    Server: "k8s-monitor",
    Tool:   "fetch_logs",
    Arguments: map[string]interface{}{
        "namespace": "production",
        "pod_name":  "api-server-abc123",
        "tail_lines": 100,
    },
})

// Read a resource
resource, err := client.Resources.Read(ctx, "pods://production/api-server-abc123")

// Get workflow status
workflow, err := client.Workflows.Get(ctx, "wf-123456")
```

---

## MCP WebSocket Streaming

For real-time MCP server events and tool execution updates:

### WS /api/v1/mcp/stream

**Connection:**
```javascript
const ws = new WebSocket('wss://k8s-assistant.example.com/api/v1/mcp/stream');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('MCP Event:', data);
};
```

**Event Types:**
- `mcp.server.status_changed`
- `mcp.tool.called`
- `mcp.tool.completed`
- `mcp.tool.failed`
- `mcp.workflow.started`
- `mcp.workflow.step_completed`
- `mcp.workflow.completed`
- `mcp.resource.accessed`

**Example Events:**

```json
{
  "type": "mcp.tool.called",
  "timestamp": "2024-01-15T10:00:00Z",
  "data": {
    "server": "k8s-monitor",
    "tool": "watch_pods",
    "arguments": {
      "namespace": "production"
    },
    "workflow_id": "wf-123456"
  }
}
```

```json
{
  "type": "mcp.workflow.step_completed",
  "timestamp": "2024-01-15T10:00:01Z",
  "data": {
    "workflow_id": "wf-123456",
    "step": 2,
    "server": "metrics",
    "tool": "query_metrics",
    "status": "completed",
    "duration_ms": 320
  }
}
```

View the full OpenAPI spec: [openapi.yaml](./openapi.yaml)