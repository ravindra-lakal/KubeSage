# Architecture Documentation

## AI-Powered Kubernetes SRE Assistant

This document provides a comprehensive overview of the system architecture, components, and design decisions.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Layers](#architecture-layers)
3. [Component Details](#component-details)
4. [Data Flow](#data-flow)
5. [Technology Stack](#technology-stack)
6. [Design Patterns](#design-patterns)
7. [Scalability & Performance](#scalability--performance)
8. [Security Architecture](#security-architecture)

---

## System Overview

The AI-Powered Kubernetes SRE Assistant is designed as a multi-layered, event-driven system that continuously monitors Kubernetes clusters, detects anomalies, and automatically remediates issues using AI-powered analysis.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │   Pods   │  │ Services │  │   Nodes  │  │  Events  │           │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │
└─────────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      Data Collection Layer                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  Prometheus  │  │   K8s API    │  │    Loki      │             │
│  │   Metrics    │  │   Watcher    │  │    Logs      │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    Detection & Analysis Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │   Anomaly    │  │    Issue     │  │  Specialized │             │
│  │  Detection   │  │  Classifier  │  │   Analyzers  │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      AI Intelligence Layer                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │     LLM      │  │   Context    │  │   Solution   │             │
│  │   Analyzer   │  │  Enrichment  │  │  Generator   │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         Action Layer                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  Automated   │  │    Human     │  │  Validation  │             │
│  │ Remediation  │  │    Review    │  │ & Rollback   │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────────┐
│                       Feedback & Learning                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  Knowledge   │  │   Learning   │  │   Metrics    │             │
│  │     Base     │  │    System    │  │  Dashboard   │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Architecture Layers

### 1. Data Collection Layer

**Purpose**: Gather comprehensive data from the Kubernetes cluster

**Components**:

#### 1.1 Metrics Collector
- **Technology**: Prometheus + Metrics Server
- **Responsibility**: Collect resource utilization metrics
- **Collection Interval**: 15 seconds
- **Metrics Collected**:
  - CPU usage (per pod, node, container)
  - Memory usage and limits
  - Network I/O
  - Disk I/O
  - Request rates and latencies
  - Custom application metrics

#### 1.2 Event Watcher
- **Technology**: Kubernetes Informers (client-go)
- **Responsibility**: Real-time event streaming
- **Events Monitored**:
  - Pod lifecycle events (Created, Started, Failed, Killed)
  - Node events (NotReady, OutOfDisk, MemoryPressure)
  - Deployment events (ScalingReplicaSet, FailedCreate)
  - ConfigMap/Secret changes
  - Resource quota violations

#### 1.3 Log Aggregator
- **Technology**: Fluentd/Fluent Bit + Loki
- **Responsibility**: Centralized log collection and indexing
- **Log Sources**:
  - Container stdout/stderr
  - Kubernetes audit logs
  - System logs
  - Application logs

**Data Flow**:
```
K8s Cluster → Collectors → Time-Series DB (Prometheus)
                        → Event Stream (Kafka/Redis)
                        → Log Storage (Loki/Elasticsearch)
```

---

### 2. Detection & Analysis Layer

**Purpose**: Identify anomalies and classify issues

**Components**:

#### 2.1 Anomaly Detection Engine
- **Technology**: Python (scikit-learn, statsmodels)
- **Algorithms**:
  - **Statistical Methods**:
    - Moving averages and standard deviations
    - Seasonal decomposition
    - Z-score analysis
  - **Machine Learning**:
    - Isolation Forest for outlier detection
    - LSTM for time-series prediction
    - Autoencoders for pattern recognition

#### 2.2 Issue Classifier
- **Technology**: Rule-based + ML classification
- **Classification Categories**:
  - Crash/Restart issues
  - Performance degradation
  - Resource exhaustion
  - Configuration errors
  - Network issues
  - Security incidents

#### 2.3 Specialized Analyzers

**Crash Analyzer**:
```go
type CrashAnalyzer struct {
    restartThreshold  int
    exitCodePatterns  map[int]string
    backoffDetector   *BackoffDetector
}

func (ca *CrashAnalyzer) Analyze(pod *v1.Pod) *CrashReport {
    // Analyze restart count
    // Check exit codes
    // Detect CrashLoopBackOff
    // Extract error messages from logs
}
```

**Latency Analyzer**:
```python
class LatencyAnalyzer:
    def __init__(self):
        self.baseline_percentiles = {}
        self.alert_threshold = 2.0  # 2x baseline
    
    def analyze(self, service_name: str, metrics: dict) -> LatencyReport:
        # Compare current p50, p95, p99 with baseline
        # Identify slow endpoints
        # Correlate with resource usage
        # Check for downstream dependencies
```

**Memory Leak Detector**:
```python
class MemoryLeakDetector:
    def __init__(self):
        self.growth_threshold = 0.1  # 10% per hour
        self.observation_window = 6  # hours
    
    def detect(self, pod_name: str, memory_series: TimeSeries) -> LeakReport:
        # Calculate memory growth rate
        # Detect linear/exponential growth
        # Identify leak patterns
        # Estimate time to OOM
```

---

### 3. AI Intelligence Layer

**Purpose**: Provide intelligent root cause analysis and solution generation

**Components**:

#### 3.1 Context Enrichment Service
```python
class ContextEnricher:
    def enrich_issue(self, issue: Issue) -> EnrichedContext:
        context = {
            'logs': self.get_recent_logs(issue.pod, lines=1000),
            'metrics': self.get_metric_window(issue.pod, duration='15m'),
            'events': self.get_related_events(issue.pod),
            'config': self.get_pod_config(issue.pod),
            'dependencies': self.get_service_dependencies(issue.service),
            'history': self.query_similar_incidents(issue)
        }
        return EnrichedContext(context)
```

#### 3.2 LLM Root Cause Analyzer
```python
class LLMAnalyzer:
    def __init__(self, llm_client):
        self.llm = llm_client
        self.prompt_template = self.load_prompt_template()
    
    def analyze_root_cause(self, enriched_context: EnrichedContext) -> RootCauseAnalysis:
        prompt = self.build_prompt(enriched_context)
        
        response = self.llm.complete(
            prompt=prompt,
            temperature=0.2,  # Low temperature for consistency
            max_tokens=2000
        )
        
        return self.parse_analysis(response)
```

**Prompt Structure**:
```
System Context:
- Cluster: production-us-east-1
- Namespace: payment-service
- Resource Limits: CPU: 2 cores, Memory: 4Gi

Issue Symptoms:
- Pod restarted 5 times in last 10 minutes
- Exit code: 137 (OOMKilled)
- Memory usage before crash: 3.9Gi (97.5% of limit)

Recent Logs:
[Last 100 lines of logs]

Metrics History:
[15-minute metric window showing memory growth]

Similar Past Incidents:
[3 similar incidents with resolutions]

Task: Analyze the root cause of this issue and explain:
1. What happened?
2. Why did it happen?
3. What are the contributing factors?
4. What is the immediate impact?
```

#### 3.3 Solution Generator
```python
class SolutionGenerator:
    def generate_solutions(self, root_cause: RootCauseAnalysis) -> List[Solution]:
        solutions = []
        
        # Generate multiple solution options
        for strategy in self.solution_strategies:
            solution = strategy.generate(root_cause)
            solution.confidence = self.calculate_confidence(solution, root_cause)
            solution.risk_level = self.assess_risk(solution)
            solution.estimated_impact = self.estimate_impact(solution)
            solutions.append(solution)
        
        # Rank solutions by confidence and safety
        return sorted(solutions, key=lambda s: (s.confidence, -s.risk_level))
```

**Solution Structure**:
```python
@dataclass
class Solution:
    id: str
    description: str
    actions: List[Action]
    confidence: float  # 0.0 to 1.0
    risk_level: RiskLevel  # LOW, MEDIUM, HIGH
    estimated_impact: str
    rollback_procedure: List[Action]
    validation_checks: List[Check]
```

#### 3.4 Knowledge Base
```python
class KnowledgeBase:
    def __init__(self):
        self.vector_db = ChromaDB()  # For semantic search
        self.sql_db = PostgreSQL()   # For structured data
    
    def store_incident(self, incident: Incident, resolution: Resolution):
        # Store in SQL for structured queries
        self.sql_db.insert_incident(incident, resolution)
        
        # Create embedding for semantic search
        embedding = self.create_embedding(incident)
        self.vector_db.add(embedding, incident.id)
    
    def find_similar_incidents(self, current_issue: Issue, limit=5) -> List[Incident]:
        # Semantic search using embeddings
        query_embedding = self.create_embedding(current_issue)
        similar_ids = self.vector_db.query(query_embedding, limit=limit)
        
        return [self.sql_db.get_incident(id) for id in similar_ids]
```

---

### 4. Action Layer

**Purpose**: Execute remediation actions safely and efficiently

**Components**:

#### 4.1 Action Controller
```go
type ActionController struct {
    k8sClient     kubernetes.Interface
    dryRunMode    bool
    rateLimiter   *RateLimiter
    circuitBreaker *CircuitBreaker
}

func (ac *ActionController) ExecuteAction(action Action) (*ActionResult, error) {
    // Check if auto-fix is enabled
    if !ac.shouldAutoFix(action) {
        return ac.queueForReview(action)
    }
    
    // Rate limiting
    if !ac.rateLimiter.Allow(action.Type) {
        return nil, ErrRateLimitExceeded
    }
    
    // Circuit breaker check
    if ac.circuitBreaker.IsOpen(action.Type) {
        return nil, ErrCircuitBreakerOpen
    }
    
    // Dry run first
    if err := ac.dryRun(action); err != nil {
        return nil, fmt.Errorf("dry run failed: %w", err)
    }
    
    // Execute action
    result, err := ac.execute(action)
    if err != nil {
        ac.circuitBreaker.RecordFailure(action.Type)
        return nil, err
    }
    
    // Validate result
    if err := ac.validate(result); err != nil {
        ac.rollback(action)
        return nil, fmt.Errorf("validation failed: %w", err)
    }
    
    ac.circuitBreaker.RecordSuccess(action.Type)
    return result, nil
}
```

#### 4.2 Auto-Fix Decision Engine
```python
class AutoFixDecisionEngine:
    def should_auto_fix(self, solution: Solution, issue: Issue) -> Decision:
        # Confidence threshold
        if solution.confidence < 0.90:
            return Decision.HUMAN_REVIEW
        
        # Risk assessment
        if solution.risk_level == RiskLevel.HIGH:
            return Decision.HUMAN_REVIEW
        
        # Critical service check
        if issue.service in self.critical_services:
            if solution.confidence < 0.95:
                return Decision.HUMAN_REVIEW
        
        # Time-based restrictions
        if self.is_business_hours() and solution.requires_restart:
            return Decision.HUMAN_REVIEW
        
        # Historical success rate
        success_rate = self.get_historical_success_rate(solution.type)
        if success_rate < 0.85:
            return Decision.HUMAN_REVIEW
        
        return Decision.AUTO_FIX
```

**Decision Matrix**:

| Confidence | Risk Level | Critical Service | Time | Action |
|-----------|-----------|------------------|------|--------|
| >95% | LOW | Yes | Any | Auto-fix |
| >90% | LOW | No | Any | Auto-fix |
| >90% | MEDIUM | No | Off-hours | Auto-fix |
| >90% | MEDIUM | No | Business hours | Review |
| >90% | HIGH | Any | Any | Review |
| 70-90% | Any | Any | Any | Review |
| <70% | Any | Any | Any | Escalate |

#### 4.3 Validation & Rollback System
```python
class ValidationSystem:
    def validate_fix(self, action: Action, timeout: int = 300) -> ValidationResult:
        start_time = time.time()
        
        while time.time() - start_time < timeout:
            # Check pod health
            if not self.is_pod_healthy(action.target_pod):
                return ValidationResult.FAILED
            
            # Check metrics improvement
            metrics = self.get_current_metrics(action.target_pod)
            if self.metrics_improved(metrics, action.baseline_metrics):
                return ValidationResult.SUCCESS
            
            time.sleep(10)
        
        return ValidationResult.TIMEOUT

class RollbackSystem:
    def rollback(self, action: Action) -> RollbackResult:
        # Get previous state
        previous_state = self.state_store.get(action.target)
        
        # Apply rollback
        try:
            self.k8s_client.apply(previous_state)
            
            # Wait for rollback to complete
            self.wait_for_rollback(action.target)
            
            return RollbackResult.SUCCESS
        except Exception as e:
            return RollbackResult.FAILED
```

---

### 5. Feedback & Learning Layer

**Purpose**: Continuously improve system performance

**Components**:

#### 5.1 Learning System
```python
class LearningSystem:
    def update_models(self, incident: Incident, outcome: Outcome):
        # Update confidence model
        self.confidence_model.update(
            features=incident.features,
            actual_success=outcome.success
        )
        
        # Update anomaly detection thresholds
        if outcome.false_positive:
            self.anomaly_detector.adjust_threshold(
                metric=incident.metric,
                direction='increase'
            )
        
        # Update solution effectiveness
        self.solution_ranker.update_score(
            solution_type=incident.solution.type,
            success=outcome.success
        )
```

#### 5.2 Metrics & Observability
```yaml
Metrics Tracked:
  Detection:
    - issues_detected_total
    - false_positive_rate
    - mean_time_to_detect (MTTD)
  
  Analysis:
    - llm_analysis_duration
    - confidence_score_distribution
    - root_cause_accuracy
  
  Remediation:
    - auto_fix_attempts_total
    - auto_fix_success_rate
    - mean_time_to_resolution (MTTR)
    - rollback_rate
  
  Learning:
    - knowledge_base_size
    - model_accuracy_improvement
    - similar_incident_match_rate
```

---

## Data Flow

### Complete Issue Resolution Flow

```
1. Detection Phase
   ├─ Metrics Collector detects anomaly
   ├─ Event Watcher captures pod restart
   └─ Triggers Issue Detection

2. Analysis Phase
   ├─ Issue Classifier categorizes as "OOMKill"
   ├─ Context Enricher gathers:
   │  ├─ Last 1000 log lines
   │  ├─ 15-minute metric window
   │  ├─ Related events
   │  └─ Similar historical incidents
   └─ LLM Analyzer determines root cause

3. Solution Phase
   ├─ Solution Generator creates options:
   │  ├─ Option 1: Increase memory limit (confidence: 92%)
   │  ├─ Option 2: Add memory profiling (confidence: 75%)
   │  └─ Option 3: Horizontal scaling (confidence: 68%)
   └─ Ranks by confidence and risk

4. Decision Phase
   ├─ Auto-Fix Engine evaluates:
   │  ├─ Confidence: 92% (>90% threshold) ✓
   │  ├─ Risk: LOW ✓
   │  ├─ Critical service: No ✓
   │  └─ Decision: AUTO-FIX
   └─ Proceeds to execution

5. Execution Phase
   ├─ Rate limiter check: PASS
   ├─ Circuit breaker check: CLOSED
   ├─ Dry run: SUCCESS
   ├─ Apply patch: memory 2Gi → 4Gi
   └─ Trigger rolling update

6. Validation Phase
   ├─ Monitor pod health: HEALTHY
   ├─ Check memory usage: 2.1Gi (52% of limit)
   ├─ Verify no restarts: 0 restarts in 5 minutes
   └─ Validation: SUCCESS

7. Learning Phase
   ├─ Store incident in knowledge base
   ├─ Update confidence model
   ├─ Record solution effectiveness
   └─ Generate metrics
```

---

## Technology Stack Details

### Backend Services

#### Watcher Service (Go)
```go
// Main watcher implementation
type WatcherService struct {
    k8sClient     kubernetes.Interface
    metricsClient promclient.API
    eventChan     chan Event
    metricsChan   chan Metric
}

func (ws *WatcherService) Start(ctx context.Context) error {
    // Start Kubernetes informers
    go ws.watchPods(ctx)
    go ws.watchNodes(ctx)
    go ws.watchEvents(ctx)
    
    // Start metrics collection
    go ws.collectMetrics(ctx)
    
    return nil
}
```

#### AI Service (Python)
```python
# FastAPI service for AI operations
from fastapi import FastAPI
from langchain import LLMChain

app = FastAPI()

@app.post("/analyze")
async def analyze_issue(issue: Issue) -> Analysis:
    # Enrich context
    context = context_enricher.enrich(issue)
    
    # LLM analysis
    analysis = llm_analyzer.analyze(context)
    
    # Generate solutions
    solutions = solution_generator.generate(analysis)
    
    return Analysis(
        root_cause=analysis,
        solutions=solutions
    )
```

### Data Storage

#### PostgreSQL Schema
```sql
-- Incidents table
CREATE TABLE incidents (
    id UUID PRIMARY KEY,
    cluster_id VARCHAR(255),
    namespace VARCHAR(255),
    pod_name VARCHAR(255),
    issue_type VARCHAR(100),
    detected_at TIMESTAMP,
    resolved_at TIMESTAMP,
    root_cause TEXT,
    solution_applied TEXT,
    confidence_score FLOAT,
    auto_fixed BOOLEAN,
    success BOOLEAN,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Solutions table
CREATE TABLE solutions (
    id UUID PRIMARY KEY,
    incident_id UUID REFERENCES incidents(id),
    solution_type VARCHAR(100),
    description TEXT,
    actions JSONB,
    confidence FLOAT,
    risk_level VARCHAR(20),
    applied BOOLEAN,
    success BOOLEAN,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Metrics table
CREATE TABLE metrics_history (
    id BIGSERIAL PRIMARY KEY,
    pod_name VARCHAR(255),
    metric_name VARCHAR(100),
    metric_value FLOAT,
    timestamp TIMESTAMP,
    INDEX idx_pod_metric_time (pod_name, metric_name, timestamp)
);
```

---

## Design Patterns

### 1. Observer Pattern
Used for event watching and notification system.

### 2. Strategy Pattern
Used for different solution generation strategies.

### 3. Circuit Breaker Pattern
Prevents cascading failures in auto-fix system.

### 4. Command Pattern
Used for action execution and rollback.

### 5. Repository Pattern
Used for data access abstraction.

---

## Scalability & Performance

### Horizontal Scaling
- **Watcher Service**: Multiple replicas with leader election
- **AI Service**: Stateless, can scale to N replicas
- **Action Controller**: Single active instance with standby

### Performance Optimizations
- **Caching**: Redis for frequently accessed data
- **Batching**: Batch metric queries to reduce API calls
- **Async Processing**: Non-blocking I/O for all operations
- **Connection Pooling**: Reuse database connections

### Resource Requirements
```yaml
Watcher Service:
  CPU: 500m - 2 cores
  Memory: 1Gi - 4Gi
  Replicas: 2-5

AI Service:
  CPU: 1 - 4 cores
  Memory: 2Gi - 8Gi
  Replicas: 2-10

Action Controller:
  CPU: 500m - 1 core
  Memory: 512Mi - 2Gi
  Replicas: 1 (active) + 1 (standby)
```

---

## Security Architecture

### Authentication & Authorization
- **Service Accounts**: Kubernetes RBAC for cluster access
- **API Keys**: Encrypted storage for LLM API keys
- **mTLS**: Service-to-service communication

### RBAC Permissions
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8s-assistant
rules:
- apiGroups: [""]
  resources: ["pods", "events", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "patch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### Audit Logging
All actions are logged with:
- Timestamp
- User/Service account
- Action type
- Target resource
- Result (success/failure)
- Reason

---

## Conclusion

This architecture provides a robust, scalable, and intelligent system for autonomous Kubernetes cluster management. The multi-layered approach ensures separation of concerns, while the AI-powered analysis enables sophisticated root cause detection and remediation.