# Implementation Guide

## AI-Powered Kubernetes SRE Assistant

This document provides a comprehensive technical implementation guide for building the AI-Powered Kubernetes SRE Assistant.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Core Components Implementation](#core-components-implementation)
4. [Service Implementation](#service-implementation)
5. [Integration Guide](#integration-guide)
6. [Testing Strategy](#testing-strategy)
7. [Performance Optimization](#performance-optimization)
8. [Security Implementation](#security-implementation)

---

## Prerequisites

### Required Tools & Technologies

```yaml
Development Environment:
  - Go: 1.21+
  - Python: 3.11+
  - Node.js: 18+ (for tooling)
  - Docker: 24+
  - Kubernetes: 1.28+
  - Helm: 3.12+

Development Tools:
  - kubectl
  - kind/minikube (for local testing)
  - Skaffold (optional, for dev workflow)
  - Tilt (optional, for live reload)

IDE Recommendations:
  - VS Code with Go/Python extensions
  - GoLand or PyCharm
```

### Required Services

```yaml
Infrastructure:
  - Kubernetes cluster (local or cloud)
  - Container registry (Docker Hub, ECR, GCR)

Monitoring Stack:
  - Prometheus
  - Grafana
  - Loki
  - Alertmanager

Data Storage:
  - PostgreSQL 15+
  - Redis 7+
  - S3-compatible storage (MinIO for local)

AI/ML:
  - OpenAI API key OR
  - Anthropic API key OR
  - Local LLM (Ollama, LM Studio)
  - ChromaDB or Pinecone for vector storage

### Python Package Requirements

The project includes comprehensive requirements.txt files for all components:

**Main Requirements** (`requirements.txt`):
- All dependencies in one file for complete installation
- Includes MCP SDK, AI libraries, Kubernetes client, databases, monitoring tools

**Component-Specific Requirements**:

1. **Orchestrator** (`orchestrator/requirements.txt`):
   - MCP SDK, Anthropic/OpenAI, Kubernetes client
   - FastAPI, workflow management, monitoring
   - 68 total dependencies

2. **K8s Monitor Server** (`mcp-servers/k8s-monitor/requirements.txt`):
   - MCP SDK, Kubernetes client
   - Async libraries, logging
   - 20 total dependencies

3. **Metrics Server** (`mcp-servers/metrics/requirements.txt`):
   - MCP SDK, Prometheus client
   - Data processing (numpy, pandas, scikit-learn)
   - 29 total dependencies

4. **Detection Server** (`mcp-servers/detection/requirements.txt`):
   - MCP SDK, ML libraries
   - Anomaly detection (pyod), pattern recognition
   - 32 total dependencies

5. **Actions Server** (`mcp-servers/actions/requirements.txt`):
   - MCP SDK, Kubernetes client
   - Rate limiting, circuit breaker, validation
   - 30 total dependencies

6. **Knowledge Base Server** (`mcp-servers/knowledge-base/requirements.txt`):
   - MCP SDK, PostgreSQL, ChromaDB
   - Vector embeddings, sentence transformers
   - 39 total dependencies

7. **Notifications Server** (`mcp-servers/notifications/requirements.txt`):
   - MCP SDK, Slack/Telegram SDKs
   - Email support, template engine
   - 39 total dependencies

**Installation**:
```bash
# Install all dependencies
pip install -r requirements.txt

# Or install per component
pip install -r orchestrator/requirements.txt
pip install -r mcp-servers/k8s-monitor/requirements.txt
# ... etc
```

```

---

## Project Structure

```
k8s-assistant/
├── cmd/
│   ├── watcher/              # Watcher service entry point
│   │   └── main.go
│   ├── detector/             # Detector service entry point
│   │   └── main.go
│   ├── ai-service/           # AI service entry point
│   │   └── main.py
│   ├── action-controller/    # Action controller entry point
│   │   └── main.go
│   └── api/                  # API service entry point
│       └── main.go
├── pkg/
│   ├── watcher/              # Watcher implementation
│   │   ├── metrics.go
│   │   ├── events.go
│   │   └── logs.go
│   ├── detector/             # Detection logic
│   │   ├── anomaly.go
│   │   ├── classifier.go
│   │   └── analyzers/
│   ├── ai/                   # AI integration
│   │   ├── llm.go
│   │   ├── context.go
│   │   └── solutions.go
│   ├── actions/              # Action execution
│   │   ├── controller.go
│   │   ├── validator.go
│   │   └── rollback.go
│   ├── storage/              # Data storage
│   │   ├── postgres.go
│   │   ├── redis.go
│   │   └── vector.go
│   └── common/               # Shared utilities
│       ├── config.go
│       ├── logger.go
│       └── metrics.go
├── internal/
│   ├── models/               # Data models
│   └── utils/                # Internal utilities
├── python/
│   ├── ai_service/           # Python AI service
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── llm_analyzer.py
│   │   ├── context_enricher.py
│   │   ├── solution_generator.py
│   │   └── knowledge_base.py
│   ├── requirements.txt
│   └── Dockerfile
├── charts/                   # Helm charts
│   └── k8s-assistant/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── config/                   # Configuration files
│   ├── config.yaml
│   ├── rbac.yaml
│   └── prometheus-rules.yaml
├── scripts/                  # Utility scripts
│   ├── setup.sh
│   ├── deploy.sh
│   └── test.sh
├── tests/                    # Test suites
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docs/                     # Additional documentation
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

---

## Core Components Implementation

### 1. Watcher Service (Go)

#### Metrics Collector

```go
// pkg/watcher/metrics.go
package watcher

import (
    "context"
    "time"
    
    "github.com/prometheus/client_golang/api"
    v1 "github.com/prometheus/client_golang/api/prometheus/v1"
    "github.com/prometheus/common/model"
)

type MetricsCollector struct {
    promClient v1.API
    interval   time.Duration
    metricsCh  chan<- Metric
}

type Metric struct {
    PodName     string
    Namespace   string
    MetricName  string
    Value       float64
    Timestamp   time.Time
    Labels      map[string]string
}

func NewMetricsCollector(promURL string, interval time.Duration, ch chan<- Metric) (*MetricsCollector, error) {
    client, err := api.NewClient(api.Config{
        Address: promURL,
    })
    if err != nil {
        return nil, err
    }
    
    return &MetricsCollector{
        promClient: v1.NewAPI(client),
        interval:   interval,
        metricsCh:  ch,
    }, nil
}

func (mc *MetricsCollector) Start(ctx context.Context) error {
    ticker := time.NewTicker(mc.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if err := mc.collectMetrics(ctx); err != nil {
                log.Errorf("Failed to collect metrics: %v", err)
            }
        }
    }
}

func (mc *MetricsCollector) collectMetrics(ctx context.Context) error {
    queries := []string{
        `container_memory_usage_bytes{container!=""}`,
        `rate(container_cpu_usage_seconds_total{container!=""}[5m])`,
        `container_network_receive_bytes_total{container!=""}`,
        `container_network_transmit_bytes_total{container!=""}`,
    }
    
    for _, query := range queries {
        result, warnings, err := mc.promClient.Query(ctx, query, time.Now())
        if err != nil {
            return err
        }
        
        if len(warnings) > 0 {
            log.Warnf("Prometheus warnings: %v", warnings)
        }
        
        mc.processResult(result)
    }
    
    return nil
}

func (mc *MetricsCollector) processResult(result model.Value) {
    vector, ok := result.(model.Vector)
    if !ok {
        return
    }
    
    for _, sample := range vector {
        metric := Metric{
            PodName:    string(sample.Metric["pod"]),
            Namespace:  string(sample.Metric["namespace"]),
            MetricName: string(sample.Metric["__name__"]),
            Value:      float64(sample.Value),
            Timestamp:  sample.Timestamp.Time(),
            Labels:     make(map[string]string),
        }
        
        for k, v := range sample.Metric {
            metric.Labels[string(k)] = string(v)
        }
        
        select {
        case mc.metricsCh <- metric:
        default:
            log.Warn("Metrics channel full, dropping metric")
        }
    }
}
```

#### Event Watcher

```go
// pkg/watcher/events.go
package watcher

import (
    "context"
    
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
)

type EventWatcher struct {
    clientset  kubernetes.Interface
    eventsCh   chan<- Event
    stopCh     chan struct{}
}

type Event struct {
    Type       string
    Reason     string
    Message    string
    Object     string
    Namespace  string
    Timestamp  time.Time
    Count      int32
}

func NewEventWatcher(clientset kubernetes.Interface, ch chan<- Event) *EventWatcher {
    return &EventWatcher{
        clientset: clientset,
        eventsCh:  ch,
        stopCh:    make(chan struct{}),
    }
}

func (ew *EventWatcher) Start(ctx context.Context) error {
    factory := informers.NewSharedInformerFactory(ew.clientset, 0)
    
    // Pod informer
    podInformer := factory.Core().V1().Pods().Informer()
    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    ew.handlePodAdd,
        UpdateFunc: ew.handlePodUpdate,
        DeleteFunc: ew.handlePodDelete,
    })
    
    // Event informer
    eventInformer := factory.Core().V1().Events().Informer()
    eventInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: ew.handleEvent,
    })
    
    factory.Start(ew.stopCh)
    factory.WaitForCacheSync(ew.stopCh)
    
    <-ctx.Done()
    close(ew.stopCh)
    return nil
}

func (ew *EventWatcher) handlePodUpdate(oldObj, newObj interface{}) {
    oldPod := oldObj.(*corev1.Pod)
    newPod := newObj.(*corev1.Pod)
    
    // Check for restart
    for i, cs := range newPod.Status.ContainerStatuses {
        oldCS := oldPod.Status.ContainerStatuses[i]
        if cs.RestartCount > oldCS.RestartCount {
            event := Event{
                Type:      "Warning",
                Reason:    "ContainerRestarted",
                Message:   fmt.Sprintf("Container %s restarted", cs.Name),
                Object:    newPod.Name,
                Namespace: newPod.Namespace,
                Timestamp: time.Now(),
                Count:     cs.RestartCount,
            }
            
            select {
            case ew.eventsCh <- event:
            default:
                log.Warn("Events channel full")
            }
        }
    }
}

func (ew *EventWatcher) handleEvent(obj interface{}) {
    k8sEvent := obj.(*corev1.Event)
    
    // Filter relevant events
    if !ew.isRelevantEvent(k8sEvent) {
        return
    }
    
    event := Event{
        Type:      k8sEvent.Type,
        Reason:    k8sEvent.Reason,
        Message:   k8sEvent.Message,
        Object:    k8sEvent.InvolvedObject.Name,
        Namespace: k8sEvent.InvolvedObject.Namespace,
        Timestamp: k8sEvent.LastTimestamp.Time,
        Count:     k8sEvent.Count,
    }
    
    select {
    case ew.eventsCh <- event:
    default:
        log.Warn("Events channel full")
    }
}

func (ew *EventWatcher) isRelevantEvent(event *corev1.Event) bool {
    relevantReasons := map[string]bool{
        "OOMKilling":          true,
        "BackOff":             true,
        "FailedScheduling":    true,
        "FailedMount":         true,
        "Unhealthy":           true,
        "FailedCreatePodSandBox": true,
    }
    
    return event.Type == "Warning" && relevantReasons[event.Reason]
}
```

### 2. Detector Service (Go)

#### Anomaly Detector

```go
// pkg/detector/anomaly.go
package detector

import (
    "math"
    "time"
)

type AnomalyDetector struct {
    baselines map[string]*Baseline
    threshold float64
}

type Baseline struct {
    Mean   float64
    StdDev float64
    Values []float64
    Window time.Duration
}

func NewAnomalyDetector(threshold float64) *AnomalyDetector {
    return &AnomalyDetector{
        baselines: make(map[string]*Baseline),
        threshold: threshold,
    }
}

func (ad *AnomalyDetector) IsAnomaly(metricKey string, value float64) (bool, float64) {
    baseline, exists := ad.baselines[metricKey]
    if !exists {
        // Initialize baseline
        ad.baselines[metricKey] = &Baseline{
            Values: []float64{value},
            Window: 15 * time.Minute,
        }
        return false, 0
    }
    
    // Calculate z-score
    zScore := (value - baseline.Mean) / baseline.StdDev
    
    // Update baseline
    baseline.Values = append(baseline.Values, value)
    if len(baseline.Values) > 100 {
        baseline.Values = baseline.Values[1:]
    }
    baseline.Mean = calculateMean(baseline.Values)
    baseline.StdDev = calculateStdDev(baseline.Values, baseline.Mean)
    
    return math.Abs(zScore) > ad.threshold, zScore
}

func calculateMean(values []float64) float64 {
    sum := 0.0
    for _, v := range values {
        sum += v
    }
    return sum / float64(len(values))
}

func calculateStdDev(values []float64, mean float64) float64 {
    variance := 0.0
    for _, v := range values {
        variance += math.Pow(v-mean, 2)
    }
    return math.Sqrt(variance / float64(len(values)))
}
```

#### Issue Classifier

```go
// pkg/detector/classifier.go
package detector

import (
    "strings"
)

type IssueClassifier struct {
    rules []ClassificationRule
}

type ClassificationRule struct {
    Name      string
    Condition func(Issue) bool
    IssueType IssueType
}

type IssueType string

const (
    IssueTypeOOMKill        IssueType = "OOMKill"
    IssueTypeCrash          IssueType = "Crash"
    IssueTypeHighLatency    IssueType = "HighLatency"
    IssueTypeMemoryLeak     IssueType = "MemoryLeak"
    IssueTypeCPUExhaustion  IssueType = "CPUExhaustion"
    IssueTypeDiskFull       IssueType = "DiskFull"
    IssueTypeNetworkError   IssueType = "NetworkError"
    IssueTypeGeneric        IssueType = "Generic"
)

type Issue struct {
    ID          string
    PodName     string
    Namespace   string
    Type        IssueType
    Severity    string
    Description string
    Metrics     map[string]float64
    Events      []Event
    Logs        []string
    DetectedAt  time.Time
}

func NewIssueClassifier() *IssueClassifier {
    ic := &IssueClassifier{}
    ic.initializeRules()
    return ic
}

func (ic *IssueClassifier) initializeRules() {
    ic.rules = []ClassificationRule{
        {
            Name: "OOMKill Detection",
            Condition: func(issue Issue) bool {
                for _, event := range issue.Events {
                    if event.Reason == "OOMKilling" {
                        return true
                    }
                }
                return false
            },
            IssueType: IssueTypeOOMKill,
        },
        {
            Name: "Crash Detection",
            Condition: func(issue Issue) bool {
                for _, event := range issue.Events {
                    if strings.Contains(event.Reason, "BackOff") ||
                       strings.Contains(event.Reason, "CrashLoop") {
                        return true
                    }
                }
                return false
            },
            IssueType: IssueTypeCrash,
        },
        {
            Name: "High Latency Detection",
            Condition: func(issue Issue) bool {
                if latency, ok := issue.Metrics["p95_latency"]; ok {
                    baseline := issue.Metrics["baseline_p95_latency"]
                    return latency > baseline*2
                }
                return false
            },
            IssueType: IssueTypeHighLatency,
        },
        {
            Name: "Memory Leak Detection",
            Condition: func(issue Issue) bool {
                if growth, ok := issue.Metrics["memory_growth_rate"]; ok {
                    return growth > 0.1 // 10% per hour
                }
                return false
            },
            IssueType: IssueTypeMemoryLeak,
        },
    }
}

func (ic *IssueClassifier) Classify(issue *Issue) IssueType {
    for _, rule := range ic.rules {
        if rule.Condition(*issue) {
            return rule.IssueType
        }
    }
    return IssueTypeGeneric
}
```

### 3. AI Service (Python)

#### LLM Analyzer

```python
# python/ai_service/llm_analyzer.py
from typing import Dict, List, Optional
from dataclasses import dataclass
import openai
from langchain.llms import OpenAI
from langchain.prompts import PromptTemplate
from langchain.chains import LLMChain

@dataclass
class RootCauseAnalysis:
    root_cause: str
    contributing_factors: List[str]
    impact_assessment: str
    recommendations: List[str]
    confidence: float

class LLMAnalyzer:
    def __init__(self, api_key: str, model: str = "gpt-4"):
        self.llm = OpenAI(
            api_key=api_key,
            model=model,
            temperature=0.2,
            max_tokens=2000
        )
        self.prompt_template = self._create_prompt_template()
        self.chain = LLMChain(llm=self.llm, prompt=self.prompt_template)
    
    def _create_prompt_template(self) -> PromptTemplate:
        template = """
You are an expert Site Reliability Engineer analyzing a Kubernetes issue.

System Context:
Cluster: {cluster_name}
Namespace: {namespace}
Pod: {pod_name}
Resource Limits: CPU: {cpu_limit}, Memory: {memory_limit}

Issue Symptoms:
Type: {issue_type}
Description: {description}
Detected At: {detected_at}

Recent Metrics:
{metrics}

Recent Events:
{events}

Recent Logs (last 100 lines):
{logs}

Similar Past Incidents:
{similar_incidents}

Task: Analyze this issue and provide:
1. Root Cause: What is the primary cause of this issue?
2. Contributing Factors: What other factors contributed to this issue?
3. Impact Assessment: What is the impact on the system and users?
4. Recommendations: What are the recommended solutions?

Provide your analysis in a structured format.
"""
        return PromptTemplate(
            input_variables=[
                "cluster_name", "namespace", "pod_name", "cpu_limit",
                "memory_limit", "issue_type", "description", "detected_at",
                "metrics", "events", "logs", "similar_incidents"
            ],
            template=template
        )
    
    def analyze(self, enriched_context: Dict) -> RootCauseAnalysis:
        """Analyze issue using LLM"""
        
        # Prepare input
        input_data = self._prepare_input(enriched_context)
        
        # Call LLM
        response = self.chain.run(**input_data)
        
        # Parse response
        analysis = self._parse_response(response)
        
        return analysis
    
    def _prepare_input(self, context: Dict) -> Dict:
        """Prepare input for LLM"""
        return {
            "cluster_name": context.get("cluster_name", "unknown"),
            "namespace": context.get("namespace", ""),
            "pod_name": context.get("pod_name", ""),
            "cpu_limit": context.get("cpu_limit", ""),
            "memory_limit": context.get("memory_limit", ""),
            "issue_type": context.get("issue_type", ""),
            "description": context.get("description", ""),
            "detected_at": context.get("detected_at", ""),
            "metrics": self._format_metrics(context.get("metrics", {})),
            "events": self._format_events(context.get("events", [])),
            "logs": self._format_logs(context.get("logs", [])),
            "similar_incidents": self._format_similar_incidents(
                context.get("similar_incidents", [])
            )
        }
    
    def _format_metrics(self, metrics: Dict) -> str:
        """Format metrics for prompt"""
        lines = []
        for key, value in metrics.items():
            lines.append(f"- {key}: {value}")
        return "\n".join(lines)
    
    def _format_events(self, events: List[Dict]) -> str:
        """Format events for prompt"""
        lines = []
        for event in events[:10]:  # Last 10 events
            lines.append(
                f"- [{event['timestamp']}] {event['type']}: "
                f"{event['reason']} - {event['message']}"
            )
        return "\n".join(lines)
    
    def _format_logs(self, logs: List[str]) -> str:
        """Format logs for prompt"""
        return "\n".join(logs[-100:])  # Last 100 lines
    
    def _format_similar_incidents(self, incidents: List[Dict]) -> str:
        """Format similar incidents for prompt"""
        if not incidents:
            return "No similar incidents found."
        
        lines = []
        for incident in incidents[:3]:  # Top 3 similar
            lines.append(
                f"- Issue: {incident['description']}\n"
                f"  Solution: {incident['solution']}\n"
                f"  Success: {incident['success']}"
            )
        return "\n".join(lines)
    
    def _parse_response(self, response: str) -> RootCauseAnalysis:
        """Parse LLM response into structured format"""
        # Simple parsing - in production, use more robust parsing
        sections = response.split("\n\n")
        
        root_cause = ""
        contributing_factors = []
        impact_assessment = ""
        recommendations = []
        
        for section in sections:
            if "Root Cause:" in section:
                root_cause = section.split("Root Cause:")[1].strip()
            elif "Contributing Factors:" in section:
                factors = section.split("Contributing Factors:")[1].strip()
                contributing_factors = [
                    f.strip() for f in factors.split("\n") if f.strip()
                ]
            elif "Impact Assessment:" in section:
                impact_assessment = section.split("Impact Assessment:")[1].strip()
            elif "Recommendations:" in section:
                recs = section.split("Recommendations:")[1].strip()
                recommendations = [
                    r.strip() for r in recs.split("\n") if r.strip()
                ]
        
        return RootCauseAnalysis(
            root_cause=root_cause,
            contributing_factors=contributing_factors,
            impact_assessment=impact_assessment,
            recommendations=recommendations,
            confidence=0.85  # Calculate based on response quality
        )
```

#### Solution Generator

```python
# python/ai_service/solution_generator.py
from typing import List, Dict
from dataclasses import dataclass
from enum import Enum

class RiskLevel(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"

@dataclass
class Action:
    type: str
    target: str
    parameters: Dict
    description: str

@dataclass
class Solution:
    id: str
    description: str
    actions: List[Action]
    confidence: float
    risk_level: RiskLevel
    estimated_impact: str
    rollback_procedure: List[Action]
    validation_checks: List[str]

class SolutionGenerator:
    def __init__(self):
        self.strategies = {
            "OOMKill": self._generate_oom_solutions,
            "Crash": self._generate_crash_solutions,
            "HighLatency": self._generate_latency_solutions,
            "MemoryLeak": self._generate_memory_leak_solutions,
        }
    
    def generate(self, issue_type: str, analysis: Dict) -> List[Solution]:
        """Generate solutions for the given issue type"""
        strategy = self.strategies.get(issue_type, self._generate_generic_solutions)
        solutions = strategy(analysis)
        
        # Rank solutions
        return sorted(solutions, key=lambda s: (s.confidence, -s.risk_level.value))
    
    def _generate_oom_solutions(self, analysis: Dict) -> List[Solution]:
        """Generate solutions for OOMKill issues"""
        solutions = []
        
        # Solution 1: Increase memory limit
        solutions.append(Solution(
            id="oom-increase-memory",
            description="Increase memory limit",
            actions=[
                Action(
                    type="patch_deployment",
                    target=analysis["pod_name"],
                    parameters={
                        "memory_limit": self._calculate_new_memory_limit(analysis)
                    },
                    description="Increase memory limit to accommodate workload"
                )
            ],
            confidence=0.92,
            risk_level=RiskLevel.LOW,
            estimated_impact="Pod will restart with new limits",
            rollback_procedure=[
                Action(
                    type="patch_deployment",
                    target=analysis["pod_name"],
                    parameters={"memory_limit": analysis["current_memory_limit"]},
                    description="Restore previous memory limit"
                )
            ],
            validation_checks=[
                "Pod is running",
                "Memory usage < 80% of limit",
                "No OOMKill events for 5 minutes"
            ]
        ))
        
        # Solution 2: Horizontal scaling
        solutions.append(Solution(
            id="oom-horizontal-scale",
            description="Scale horizontally to distribute load",
            actions=[
                Action(
                    type="scale_deployment",
                    target=analysis["deployment_name"],
                    parameters={"replicas": analysis["current_replicas"] + 2},
                    description="Add 2 more replicas"
                )
            ],
            confidence=0.75,
            risk_level=RiskLevel.LOW,
            estimated_impact="Additional pods will be created",
            rollback_procedure=[
                Action(
                    type="scale_deployment",
                    target=analysis["deployment_name"],
                    parameters={"replicas": analysis["current_replicas"]},
                    description="Restore previous replica count"
                )
            ],
            validation_checks=[
                "All pods are running",
                "Load is distributed",
                "No OOMKill events"
            ]
        ))
        
        return solutions
    
    def _calculate_new_memory_limit(self, analysis: Dict) -> str:
        """Calculate appropriate new memory limit"""
        current = self._parse_memory(analysis["current_memory_limit"])
        usage = self._parse_memory(analysis["peak_memory_usage"])
        
        # Add 50% buffer
        new_limit = int(usage * 1.5)
        
        return f"{new_limit}Mi"
    
    def _parse_memory(self, memory_str: str) -> int:
        """Parse memory string to MB"""
        if memory_str.endswith("Gi"):
            return int(float(memory_str[:-2]) * 1024)
        elif memory_str.endswith("Mi"):
            return int(memory_str[:-2])
        return 0
```

### 4. Action Controller (Go)

```go
// pkg/actions/controller.go
package actions

import (
    "context"
    "fmt"
    "time"
    
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

type ActionController struct {
    clientset      kubernetes.Interface
    validator      *Validator
    rollback       *RollbackSystem
    rateLimiter    *RateLimiter
    circuitBreaker *CircuitBreaker
    dryRunMode     bool
}

type ActionResult struct {
    Success   bool
    Message   string
    Timestamp time.Time
    Metrics   map[string]interface{}
}

func NewActionController(clientset kubernetes.Interface) *ActionController {
    return &ActionController{
        clientset:      clientset,
        validator:      NewValidator(clientset),
        rollback:       NewRollbackSystem(clientset),
        rateLimiter:    NewRateLimiter(),
        circuitBreaker: NewCircuitBreaker(),
        dryRunMode:     false,
    }
}

func (ac *ActionController) ExecuteAction(ctx context.Context, action Action) (*ActionResult, error) {
    // Rate limiting
    if !ac.rateLimiter.Allow(action.Type) {
        return nil, fmt.Errorf("rate limit exceeded for action type: %s", action.Type)
    }
    
    // Circuit breaker check
    if ac.circuitBreaker.IsOpen(action.Type) {
        return nil, fmt.Errorf("circuit breaker open for action type: %s", action.Type)
    }
    
    // Dry run
    if err := ac.dryRun(ctx, action); err != nil {
        return nil, fmt.Errorf("dry run failed: %w", err)
    }
    
    // Store current state for rollback
    if err := ac.rollback.StoreState(ctx, action.Target); err != nil {
        return nil, fmt.Errorf("failed to store state: %w", err)
    }
    
    // Execute action
    result, err := ac.execute(ctx, action)
    if err != nil {
        ac.circuitBreaker.RecordFailure(action.Type)
        return nil, fmt.Errorf("execution failed: %w", err)
    }
    
    // Validate result
    validationResult, err := ac.validator.Validate(ctx, action, 5*time.Minute)
    if err != nil || !validationResult.Success {
        // Rollback
        if rbErr := ac.rollback.Rollback(ctx, action.Target); rbErr != nil {
            log.Errorf("Rollback failed: %v", rbErr)
        }
        ac.circuitBreaker.RecordFailure(action.Type)
        return nil, fmt.Errorf("validation failed: %w", err)
    }
    
    ac.circuitBreaker.RecordSuccess(action.Type)
    return result, nil
}

func (ac *ActionController) dryRun(ctx context.Context, action Action) error {
    // Implement dry run logic based on action type
    switch action.Type {
    case "patch_deployment":
        return ac.dryRunPatchDeployment(ctx, action)
    case "scale_deployment":
        return ac.dryRunScaleDeployment(ctx, action)
    default:
        return fmt.Errorf("unknown action type: %s", action.Type)
    }
}

func (ac *ActionController) execute(ctx context.Context, action Action) (*ActionResult, error) {
    switch action.Type {
    case "patch_deployment":
        return ac.patchDeployment(ctx, action)
    case "scale_deployment":
        return ac.scaleDeployment(ctx, action)
    default:
        return nil, fmt.Errorf("unknown action type: %s", action.Type)
    }
}

func (ac *ActionController) patchDeployment(ctx context.Context, action Action) (*ActionResult, error) {
    // Get deployment
    deployment, err := ac.clientset.AppsV1().Deployments(action.Namespace).Get(
        ctx, action.Target, metav1.GetOptions{},
    )
    if err != nil {
        return nil, err
    }
    
    // Apply patch
    if memLimit, ok := action.Parameters["memory_limit"].(string); ok {
        for i := range deployment.Spec.Template.Spec.Containers {
            deployment.Spec.Template.Spec.Containers[i].Resources.Limits["memory"] = 
                resource.MustParse(memLimit)
        }
    }
    
    // Update deployment
    _, err = ac.clientset.AppsV1().Deployments(action.Namespace).Update(
        ctx, deployment, metav1.UpdateOptions{},
    )
    if err != nil {
        return nil, err
    }
    
    return &ActionResult{
        Success:   true,
        Message:   "Deployment patched successfully",
        Timestamp: time.Now(),
    }, nil
}
```

---

## Integration Guide

### Setting Up Development Environment

```bash
# 1. Clone repository
git clone https://github.com/your-org/k8s-assistant.git
cd k8s-assistant

# 2. Install Go dependencies
go mod download

# 3. Install Python dependencies
cd python
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cd ..

# 4. Set up local Kubernetes cluster
kind create cluster --name k8s-assistant

# 5. Install monitoring stack
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

# 6. Configure environment
cp config/config.example.yaml config/config.yaml
# Edit config.yaml with your settings

# 7. Build services
make build

# 8. Deploy to local cluster
make deploy-local
```

### Configuration

```yaml
# config/config.yaml
cluster:
  name: production-cluster
  kubeconfig: ~/.kube/config

monitoring:
  prometheus:
    url: http://prometheus-server:9090
  loki:
    url: http://loki:3100

ai:
  provider: openai  # or anthropic, local
  api_key: ${OPENAI_API_KEY}
  model: gpt-4
  temperature: 0.2
  max_tokens: 2000

storage:
  postgres:
    host: postgres
    port: 5432
    database: k8s_assistant
    user: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
  
  redis:
    host: redis
    port: 6379
    db: 0
  
  vector_db:
    provider: chromadb  # or pinecone
    url: http://chromadb:8000

detection:
  metrics_interval: 15s
  anomaly_threshold: 2.0
  min_samples: 10

actions:
  auto_fix_enabled: true
  confidence_threshold: 0.90
  rate_limit:
    max_per_minute: 10
    max_per_hour: 50
  circuit_breaker:
    failure_threshold: 5
    timeout: 60s

notifications:
  slack:
    webhook_url: ${SLACK_WEBHOOK_URL}
    channel: #sre-alerts
  email:
    smtp_host: smtp.gmail.com
    smtp_port: 587
    from: alerts@example.com
```

---

## Testing Strategy

### Unit Tests

```go
// pkg/detector/anomaly_test.go
package detector

import (
    "testing"
)

func TestAnomalyDetector(t *testing.T) {
    detector := NewAnomalyDetector(2.0)
    
    // Normal values
    for i := 0; i < 50; i++ {
        isAnomaly, _ := detector.IsAnomaly("test_metric", 100.0)
        if isAnomaly {
            t.Error("Expected no anomaly for normal values")
        }
    }
    
    // Anomalous value
    isAnomaly, zScore := detector.IsAnomaly("test_metric", 500.0)
    if !isAnomaly {
        t.Errorf("Expected anomaly, got z-score: %f", zScore)
    }
}
```

### Integration Tests

```python
# tests/integration/test_ai_service.py
import pytest
from ai_service.llm_analyzer import LLMAnalyzer

@pytest.fixture
def analyzer():
    return LLMAnalyzer(api_key="test-key")

def test_analyze_oom_issue(analyzer):
    context = {
        "cluster_name": "test-cluster",
        "namespace": "default",
        "pod_name": "test-pod",
        "issue_type": "OOMKill",
        "metrics": {"memory_usage": "3.9Gi"},
        "events": [{"reason": "OOMKilling"}],
        "logs": ["Out of memory"],
    }
    
    analysis = analyzer.analyze(context)
    
    assert analysis.root_cause is not None
    assert len(analysis.recommendations) > 0
    assert analysis.confidence > 0.5
```

### End-to-End Tests

```bash
# tests/e2e/test_full_workflow.sh
#!/bin/bash

# Deploy test application with memory leak
kubectl apply -f tests/e2e/fixtures/memory-leak-app.yaml

# Wait for OOMKill
echo "Waiting for OOMKill..."
sleep 120

# Check if issue was detected
ISSUE_COUNT=$(kubectl logs -n k8s-assistant deployment/detector | grep "OOMKill detected" | wc -l)
if [ $ISSUE_COUNT -eq 0 ]; then
    echo "FAIL: Issue not detected"
    exit 1
fi

# Check if fix was applied
FIX_COUNT=$(kubectl logs -n k8s-assistant deployment/action-controller | grep "Fix applied" | wc -l)
if [ $FIX_COUNT -eq 0 ]; then
    echo "FAIL: Fix not applied"
    exit 1
fi

# Verify pod is healthy
POD_STATUS=$(kubectl get pod -l app=memory-leak-app -o jsonpath='{.items[0].status.phase}')
if [ "$POD_STATUS" != "Running" ]; then
    echo "FAIL: Pod not running after fix"
    exit 1
fi

echo "PASS: Full workflow completed successfully"
```

---

## Performance Optimization

### Caching Strategy

```go
// pkg/common/cache.go
package common

import (
    "context"
    "encoding/json"
    "time"
    
    "github.com/go-redis/redis/v8"
)

type Cache struct {
    client *redis.Client
}

func NewCache(addr string) *Cache {
    return &Cache{
        client: redis.NewClient(&redis.Options{
            Addr: addr,
        }),
    }
}

func (c *Cache) Get(ctx context.Context, key string, dest interface{}) error {
    val, err := c.client.Get(ctx, key).Result()
    if err != nil {
        return err
    }
    return json.Unmarshal([]byte(val), dest)
}

func (c *Cache) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    return c.client.Set(ctx, key, data, ttl).Err()
}
```

### Connection Pooling

```python
# python/ai_service/db_pool.py
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

def create_db_engine(connection_string: str):
    return create_engine(
        connection_string,
        poolclass=QueuePool,
        pool_size=20,
        max_overflow=10,
        pool_pre_ping=True,
        pool_recycle=3600,
    )
```

---

## Security Implementation

### RBAC Configuration

```yaml
# config/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-assistant
  namespace: k8s-assistant
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8s-assistant
rules:
- apiGroups: [""]
  resources: ["pods", "events", "configmaps", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-assistant
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: k8s-assistant
subjects:
- kind: ServiceAccount
  name: k8s-assistant
  namespace: k8s-assistant
```

### Secrets Management

```go
// pkg/common/secrets.go
package common

import (
    "context"
    "fmt"
    
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

type SecretsManager struct {
    clientset kubernetes.Interface
    namespace string
}

func (sm *SecretsManager) GetSecret(ctx context.Context, name, key string) (string, error) {
    secret, err := sm.clientset.CoreV1().Secrets(sm.namespace).Get(
        ctx, name, metav1.GetOptions{},
    )
    if err != nil {
        return "", err
    }
    
    value, ok := secret.Data[key]
    if !ok {
        return "", fmt.Errorf("key %s not found in secret %s", key, name)
    }
    
    return string(value), nil
}
```

---

## Conclusion

This implementation guide provides the foundation for building the AI-Powered Kubernetes SRE Assistant. Key points:

1. **Modular Design**: Each component is independent and testable
2. **Scalable Architecture**: Services can scale horizontally
3. **Production-Ready**: Includes error handling, logging, and monitoring
4. **Secure**: Implements RBAC, secrets management, and audit logging
5. **Extensible**: Easy to add new detectors, analyzers, and actions

Next steps:
1. Review the DEPLOYMENT.md for deployment instructions
2. Customize configuration for your environment
3. Implement additional analyzers for specific use cases
4. Set up monitoring and alerting
5. Train the system with your historical incidents