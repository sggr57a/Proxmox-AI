# Proxmox AI Monitoring & Management System

Intelligent 24/7 monitoring and autonomous management for Proxmox infrastructure with AI-powered triage, multi-channel alerting, and continuous learning capabilities.

## Features

### Core Monitoring
- **Multi-layer monitoring**: Proxmox nodes, LXC containers, VMs, Docker containers
- **Real-time health checks**: CPU, memory, disk, network status
- **Log analysis**: AI-powered error detection and pattern recognition
- **Network monitoring**: Connectivity, DNS, port availability

### Intelligent Alerting
- **Multi-channel notifications**: Slack (business hours), Voice/Twilio (off-hours)
- **Smart routing**: Time-based and severity-based alert distribution
- **Deduplication**: Automatic suppression of duplicate alerts
- **Storm prevention**: Rate limiting for alert floods (>20/minute)

### AI-Powered Triage
- **Conversational AI**: Chat via Slack or voice for incident investigation
- **Root cause analysis**: LLM-based log analysis and diagnosis
- **Suggested actions**: Recommendations based on similar past incidents
- **Voice integration**: Sesame AI TTS for verbal alerts and conversation

### Autonomous Actions
- **Self-healing**: Automatic container restarts, service recovery
- **Approval workflows**: Human-in-the-loop for critical actions
- **Snapshot management**: Automatic pre-action snapshots with rollback
- **Backup orchestration**: Scheduled backups with integrity verification

### Infrastructure Management
- **Multi-node replication**: Automatic VM/LXC cloning across nodes
- **Network automation**: Sequential IP assignment, MAC generation
- **Naming conventions**: Automated enforcement of resource naming
- **Metadata tracking**: Origin node and template source tracking

### Learning & Improvement
- **Incident knowledge base**: RAG-powered similarity search
- **Pattern detection**: Recurring issues, cascade failures, false positives
- **Feedback loop**: Continuous improvement from user feedback
- **Performance metrics**: Accuracy, resolution time, success rate tracking

## Quick Start

### Prerequisites

- **Docker** & **Docker Compose** (v2.0+)
- **Proxmox VE** (7.0+) with API access
- **Slack workspace** (optional, for Slack notifications)
- **Twilio account** (optional, for voice alerts)
- **8GB+ RAM** recommended (for LLM and services)

### Installation

1. **Clone repository**
   ```bash
   git clone <repository-url>
   cd proxmox-ai
   ```

2. **Configure environment**
   ```bash
   cp .env.example .env
   nano .env  # Edit with your credentials
   ```

3. **Update Proxmox nodes**
   ```bash
   nano config/proxmox-nodes.yml
   ```

4. **Run setup script**
   ```bash
   ./scripts/setup.sh
   ```

   This will:
   - Build Docker images
   - Initialize PostgreSQL database
   - Pull LLM model (llama3.2:3b)
   - Download Sesame TTS model
   - Start all services
   - Verify health checks

5. **Access N8N**
   ```
   URL: http://localhost:5678
   Username: admin (from .env)
   Password: (from .env)
   ```

6. **Import workflows**
   - Go to N8N UI
   - Import workflows from `n8n/workflows/*.json`
   - Configure credentials (Slack, Twilio, Proxmox)

7. **Test the system**
   ```bash
   ./scripts/test-integration.sh
   ```

8. **Seed learning data** (optional but recommended)
   ```bash
   ./scripts/seed-learning-data.sh
   ```

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    N8N Workflow Engine                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Monitoring  │  │ Alert Router │  │   Actions    │  │
│  │  Workflows   │  │  Workflows   │  │  Workflows   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐
│ Proxmox Monitor │  │ Notifications   │  │ LLM + TTS    │
│ - Health checks │  │ - Slack         │  │ - Ollama     │
│ - Log analysis  │  │ - Twilio/Voice  │  │ - Sesame AI  │
│ - Metadata sync │  └─────────────────┘  └──────────────┘
└─────────────────┘                                │
         │                                         ▼
         ▼                              ┌────────────────────┐
┌──────────────────────────────┐       │ Learning Engine    │
│  PostgreSQL Database         │◄──────┤ - Knowledge base   │
│  - System state              │       │ - Pattern detection│
│  - Alerts & incidents        │       │ - RAG search       │
│  - Action logs               │       │ - Feedback system  │
└──────────────────────────────┘       └────────────────────┘
```

## Services

| Service | Port | Description |
|---------|------|-------------|
| **N8N** | 5678 | Workflow automation UI |
| **PostgreSQL** | 5432 | Database for state & history |
| **Proxmox Monitor** | 8000 | Monitoring & management API |
| **LLM Engine** | 8001 | AI inference (Ollama) |
| **Sesame TTS** | 8002 | Voice synthesis |
| **Twilio Bridge** | 8003 | Voice call handling |
| **Learning Engine** | 8004 | Incident analysis & RAG |
| **Ollama** | 11434 | LLM model server |

## Configuration

### Proxmox Nodes

Edit `config/proxmox-nodes.yml`:
```yaml
nodes:
  - name: pve-node-01
    host: 10.0.1.10
    user: root@pam
    password: your-password
    verify_ssl: false
```

### Alert Rules

Edit `config/alert-rules.yml` to customize:
- Alert thresholds (CPU, memory, disk)
- Severity levels
- Suppression windows
- Escalation policies

### Action Policies

Edit `config/action-policies.yml`:
- Autonomous actions (no approval needed)
- Approval-required actions
- Prohibited actions (always manual)

See [CONFIGURATION.md](CONFIGURATION.md) for detailed configuration options.

## Usage

### Monitoring Dashboard

View system state:
```bash
curl http://localhost:8000/nodes
curl http://localhost:8000/containers
curl http://localhost:8000/alerts?status=active
```

### Conversational AI

**Via Slack:**
1. Alert appears in configured Slack channel
2. Click "Investigate" button
3. Ask questions: "What caused this?", "Show me recent logs", "Suggest a fix"
4. Approve actions: "Restart the container"

**Via Voice:**
1. Receive phone call from Twilio (off-hours or critical alerts)
2. System reads alert details via TTS
3. Speak to investigate: "What's the root cause?"
4. Approve actions: "Yes, restart it"

### Manual Actions

Trigger actions via API:
```bash
# Restart container
curl -X POST http://localhost:8000/actions/restart \
  -H "Content-Type: application/json" \
  -d '{"entity_type": "container", "entity_id": "nginx-proxy", "node": "pve-node-01"}'

# Create snapshot
curl -X POST http://localhost:8000/snapshots/create \
  -H "Content-Type: application/json" \
  -d '{"entity_type": "lxc", "entity_id": "100", "node": "pve-node-01", "snapshot_name": "pre-upgrade"}'

# Replicate VM to all nodes
curl -X POST http://localhost:8000/replication/replicate \
  -H "Content-Type: application/json" \
  -d '{"entity_type": "vm", "entity_id": "200", "source_node": "pve-node-01"}'
```

### Learning System

Query similar past incidents:
```bash
curl -X POST http://localhost:8004/incidents/search \
  -H "Content-Type: application/json" \
  -d '{"query": "container restart loop port conflict", "limit": 5}'
```

See [LEARNING_SYSTEM.md](LEARNING_SYSTEM.md) for details on how the system learns and improves.

## Workflows

### Monitoring Workflows (N8N)
- `monitoring-health-check.json` - Poll system health every 1-5 minutes
- `monitoring-log-analysis.json` - Analyze logs for errors every 30 seconds
- `monitoring-network.json` - Check network connectivity every 2-5 minutes
- `monitoring-metadata-sync.json` - Sync resource metadata hourly

### Alert Workflows
- `alert-router.json` - Route alerts to Slack/voice based on time/severity
- `conversation-slack.json` - Handle Slack conversations and commands
- `conversation-voice.json` - Handle Twilio voice interactions

### Action Workflows
- `action-executor.json` - Execute approved actions with rollback support
- `snapshot-management.json` - Create/restore snapshots
- `backup-orchestration.json` - Manage backups and verification
- `replication-manager.json` - Clone VMs/LXCs across nodes
- `self-protection.json` - Prevent self-destructive actions

### Learning Workflows
- `learning-incident-recording.json` - Record resolved incidents
- `learning-feedback-collection.json` - Collect user feedback
- `learning-pattern-detection.json` - Detect recurring patterns

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common issues and solutions.

**Quick health check:**
```bash
docker-compose ps
curl http://localhost:8000/health
curl http://localhost:8001/health
curl http://localhost:8004/health
```

**View logs:**
```bash
docker-compose logs -f proxmox-monitor
docker-compose logs -f llm-engine
docker-compose logs -f learning-engine
```

## Security

**Important security practices:**
- Change all default passwords in `.env`
- Use Proxmox API tokens instead of passwords
- Enable firewall and restrict port access
- Use TLS/SSL for N8N in production
- Review SECURITY.md for complete hardening guide

See [SECURITY.md](SECURITY.md) for detailed security configuration.

## Maintenance

### Backups

Backup database:
```bash
./scripts/backup.sh
```

Backup configuration:
```bash
tar -czf config-backup-$(date +%F).tar.gz config/ .env
```

### Updates

Update services:
```bash
docker-compose pull
docker-compose up -d
```

Update LLM model:
```bash
docker-compose exec ollama ollama pull llama3.2:3b
```

### Monitoring

Check system health:
```bash
./scripts/test-integration.sh
```

View metrics:
```bash
curl http://localhost:8004/metrics
```

## Documentation

- **[CONFIGURATION.md](CONFIGURATION.md)** - Detailed configuration guide
- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** - Common issues and solutions
- **[LEARNING_SYSTEM.md](LEARNING_SYSTEM.md)** - How the learning system works
- **[SECURITY.md](SECURITY.md)** - Security hardening guide
- **[INFRASTRUCTURE_MANAGEMENT.md](INFRASTRUCTURE_MANAGEMENT.md)** - Replication, backups, networking

## Contributing

This is a custom monitoring solution. Modify workflows and configurations to fit your needs.

## License

Proprietary - Internal use only

## Support

For issues or questions:
1. Check TROUBLESHOOTING.md
2. Review service logs: `docker-compose logs <service>`
3. Check database for errors: `docker-compose exec postgres psql -U proxmox_user -d proxmox_ai`

## Roadmap

Future enhancements:
- Web dashboard for metrics visualization
- Mobile app for alerts
- Advanced ML models for anomaly detection
- Integration with additional monitoring tools (Prometheus, Grafana)
- Multi-tenant support
- Advanced RBAC for actions
