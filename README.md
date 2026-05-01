# AI-Powered Kubernetes SRE Assistant

An intelligent, autonomous Site Reliability Engineering (SRE) assistant that monitors your Kubernetes cluster, detects issues, explains root causes, and automatically applies fixes.

## 🎯 Overview

This AI-powered SRE assistant combines real-time Kubernetes monitoring with advanced AI/ML capabilities to provide:

- **Continuous Monitoring**: Watches cluster health, metrics, logs, and events
- **Intelligent Detection**: Identifies crashes, high latency, memory leaks, and resource issues
- **Root Cause Analysis**: Uses LLM to explain why issues occurred
- **Automated Remediation**: Suggests and auto-applies fixes with confidence scoring
- **Learning System**: Improves over time by learning from past incidents

## 🚀 Key Features

### 1. Multi-Layer Detection
- **Crash Detection**: Pod restarts, CrashLoopBackOff, exit code analysis
- **Latency Monitoring**: Request duration tracking (p50, p95, p99)
- **Memory Leak Detection**: Gradual memory growth pattern analysis
- **Resource Exhaustion**: CPU, memory, and disk usage monitoring

### 2. AI-Powered Analysis
- **Context Enrichment**: Collects logs, metrics, events, and historical data
- **LLM Integration**: Uses large language models for root cause analysis
- **Solution Generation**: Creates ranked fix options with confidence scores
- **Knowledge Base**: Learns from historical incidents and solutions

### 3. Safe Automation
- **Confidence Scoring**: Multi-level confidence thresholds
  - High (>90%): Auto-fix safe operations
  - Medium (70-90%): Suggest with approval
  - Low (<70%): Escalate to human
- **Safety Mechanisms**: Dry-run validation, automatic rollback, rate limiting
- **Human-in-the-Loop**: Review queue for complex or risky operations

### 4. Continuous Learning
- Tracks fix success rates
- Updates confidence models
- Improves detection accuracy
- Builds solution library

## 📊 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Data Collection Layer                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Metrics    │  │    Events    │  │     Logs     │      │
│  │  Collector   │  │   Watcher    │  │  Aggregator  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                 Detection & Analysis Layer                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Anomaly    │  │    Issue     │  │  Specialized │      │
│  │  Detection   │  │  Classifier  │  │   Analyzers  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   AI Intelligence Layer                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │     LLM      │  │   Context    │  │   Solution   │      │
│  │   Analyzer   │  │  Enrichment  │  │  Generator   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                       Action Layer                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Automated   │  │    Human     │  │  Validation  │      │
│  │ Remediation  │  │    Review    │  │ & Monitoring │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

## 🛠️ Technology Stack

### Infrastructure
- **Kubernetes**: 1.28+
- **Helm**: Chart-based deployment
- **Docker**: Container runtime

### Monitoring
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards
- **Loki**: Log aggregation and querying
- **Alertmanager**: Alert routing and management

### AI/ML
- **LangChain**: LLM orchestration framework
- **OpenAI/Claude**: Root cause analysis and solution generation
- **ChromaDB/Pinecone**: Vector database for semantic search
- **scikit-learn**: Anomaly detection algorithms

### Backend Services
- **Go**: Kubernetes controllers and operators
- **Python**: AI/ML services and data processing
- **FastAPI**: REST API services
- **gRPC**: Inter-service communication

### Data Storage
- **PostgreSQL**: Structured data and incident history
- **Redis**: Caching and message queues
- **S3/MinIO**: Log archives and backups

## 📁 Project Structure

```
k8s-assistant/
├── README.md                 # This file
├── ARCHITECTURE.md          # Detailed architecture documentation
├── WORKFLOW.md              # Workflow diagrams and explanations
├── IMPLEMENTATION.md        # Implementation guide
├── DEPLOYMENT.md            # Deployment instructions
├── docs/                    # Additional documentation
│   ├── api/                # API documentation
│   ├── configuration/      # Configuration guides
│   ├── examples/           # Example scenarios
│   └── troubleshooting/    # Troubleshooting guides
├── src/                     # Source code
│   ├── watcher/            # Kubernetes watcher service
│   ├── detector/           # Issue detection engine
│   ├── ai-service/         # AI analysis service
│   ├── action-controller/  # Remediation controller
│   └── api/                # REST API service
├── charts/                  # Helm charts
├── config/                  # Configuration files
├── tests/                   # Test suites
└── scripts/                 # Utility scripts
```

## 🚦 Quick Start

### Prerequisites
- Kubernetes cluster (1.28+)
- kubectl configured
- Helm 3.x
- Access to LLM API (OpenAI/Claude) or local LLM

### Installation

1. **Clone the repository**
```bash
git clone https://github.com/your-org/k8s-assistant.git
cd k8s-assistant
```

2. **Configure settings**
```bash
cp config/config.example.yaml config/config.yaml
# Edit config.yaml with your settings
```

3. **Deploy with Helm**
```bash
helm install k8s-assistant ./charts/k8s-assistant \
  --namespace k8s-assistant \
  --create-namespace \
  --values config/values.yaml
```

4. **Verify installation**
```bash
kubectl get pods -n k8s-assistant
```

## 📖 Documentation

- **[Architecture](ARCHITECTURE.md)**: Detailed system design and components
- **[Workflow](WORKFLOW.md)**: Complete workflow diagrams and explanations
- **[Implementation](IMPLEMENTATION.md)**: Technical implementation guide
- **[Deployment](DEPLOYMENT.md)**: Deployment and configuration instructions
- **[API Documentation](docs/api/)**: REST API reference
- **[Configuration Guide](docs/configuration/)**: Configuration options
- **[Examples](docs/examples/)**: Real-world usage examples

## 🔍 How It Works

### 1. Continuous Monitoring
The system continuously monitors your Kubernetes cluster:
- Collects metrics every 15 seconds
- Watches events in real-time
- Aggregates logs from all pods

### 2. Issue Detection
When anomalies are detected:
- Statistical analysis identifies deviations
- Pattern matching recognizes known issues
- Machine learning detects unusual behavior

### 3. AI Analysis
For each detected issue:
- Collects relevant context (logs, metrics, events)
- Queries knowledge base for similar incidents
- Uses LLM to analyze root cause
- Generates multiple solution options

### 4. Automated Action
Based on confidence and risk:
- **High confidence + Low risk**: Auto-apply fix
- **Medium confidence**: Suggest to SRE team
- **Low confidence + High risk**: Escalate for review

### 5. Validation & Learning
After applying fixes:
- Monitors metrics to validate success
- Rolls back if issues persist
- Updates knowledge base with outcomes
- Improves future detection and remediation

## 🎯 Use Cases

### Scenario 1: Pod Crash (OOMKilled)
```
Detection → Pod restarted with exit code 137
Analysis → Memory limit too low for workload spike
Solution → Increase memory limit from 2Gi to 4Gi
Action → Auto-applied (confidence: 92%)
Result → Pod stable, no further crashes
```

### Scenario 2: High Latency
```
Detection → P95 latency increased from 100ms to 2000ms
Analysis → Database connection pool exhausted
Solution → Increase max connections from 50 to 100
Action → Suggested for review (confidence: 78%)
Result → SRE approved, latency normalized
```

### Scenario 3: Memory Leak
```
Detection → Gradual memory growth over 6 hours
Analysis → Memory not released after processing requests
Solution → Restart pod + add memory profiling
Action → Auto-applied restart (confidence: 85%)
Result → Memory usage normalized, profiling enabled
```

## 🔒 Security Considerations

- **RBAC**: Minimal permissions for cluster access
- **Secrets Management**: Encrypted storage for API keys
- **Audit Logging**: All actions logged for compliance
- **Rate Limiting**: Prevents excessive auto-fixes
- **Dry-Run Mode**: Test fixes before applying
- **Manual Override**: SRE can disable auto-fix anytime

## 📊 Metrics & Monitoring

The assistant provides comprehensive metrics:
- Issues detected per hour/day
- Auto-fix success rate
- Mean time to detection (MTTD)
- Mean time to resolution (MTTR)
- Confidence score distribution
- Human intervention rate

## 🤝 Contributing

We welcome contributions! Please see our [Contributing Guide](CONTRIBUTING.md) for details.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- Kubernetes community for excellent tooling
- OpenAI/Anthropic for LLM capabilities
- Prometheus and Grafana teams for monitoring stack

## 📞 Support

- **Documentation**: [docs/](docs/)
- **Issues**: [GitHub Issues](https://github.com/your-org/k8s-assistant/issues)
- **Discussions**: [GitHub Discussions](https://github.com/your-org/k8s-assistant/discussions)
- **Slack**: [Join our community](https://slack.k8s-assistant.io)

---

**Built with ❤️ for SRE teams everywhere**