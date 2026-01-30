# Proxmox AI Monitoring System - Phase 1 Setup

## Overview
This is the Phase 1 implementation of the Proxmox AI monitoring system, providing core infrastructure for monitoring Proxmox nodes, VMs, and LXC containers.

## Prerequisites
- Docker and Docker Compose installed
- Access to Proxmox server(s) with API credentials
- At least 2GB of available RAM
- Port 5678 (N8N), 8000 (Monitor API), and 5432 (PostgreSQL) available

## Quick Start

### 1. Configure Environment
```bash
cp .env.example .env
```

Edit `.env` and set secure passwords for:
- `POSTGRES_PASSWORD`
- `N8N_BASIC_AUTH_PASSWORD`

### 2. Configure Proxmox Nodes
Edit `config/proxmox-nodes.yml` and add your Proxmox nodes:

```yaml
nodes:
  - name: pve-node-1
    host: 192.168.1.100
    user: root@pam
    token_name: monitoring
    token_value: your-api-token-here
    verify_ssl: false
```

**To create a Proxmox API token:**
1. Log into Proxmox web UI
2. Navigate to Datacenter → Permissions → API Tokens
3. Click "Add"
4. Select user (e.g., root@pam), enter token ID (e.g., "monitoring")
5. Uncheck "Privilege Separation" for full access
6. Copy the token value (you won't see it again!)

Alternatively, use password authentication:
```yaml
nodes:
  - name: pve-node-1
    host: 192.168.1.100
    user: root@pam
    password: your-password-here
    verify_ssl: false
```

### 3. Start Services
```bash
docker-compose up -d
```

### 4. Verify Services
```bash
# Check all services are running
docker-compose ps

# Check proxmox-monitor health
curl http://localhost:8000/health

# Access N8N web interface
# Open browser to: http://localhost:5678
# Login with credentials from .env
```

### 5. Import N8N Workflow
1. Open N8N at http://localhost:5678
2. Login with credentials from `.env` file
3. Click on "Workflows" → "Import from File"
4. Select `n8n/workflows/monitoring-health-check.json`
5. Configure PostgreSQL credentials:
   - Click "Credentials" in the left menu
   - Add new "Postgres" credential named "Proxmox AI DB"
   - Host: `postgres`
   - Database: Value from `POSTGRES_DB` in `.env`
   - User: Value from `POSTGRES_USER` in `.env`
   - Password: Value from `POSTGRES_PASSWORD` in `.env`
   - Port: `5432`
6. Activate the workflow by clicking the toggle switch

### 6. Test Data Collection
```bash
# Trigger manual collection
curl -X POST http://localhost:8000/collect

# View current status
curl http://localhost:8000/status

# View configured nodes
curl http://localhost:8000/nodes
```

## Services

### PostgreSQL (Port 5432)
- Database for storing system state, alerts, and logs
- Schema automatically initialized on first start

### N8N (Port 5678)
- Workflow automation platform
- Web UI for managing workflows
- Executes health monitoring every 5 minutes

### Proxmox Monitor (Port 8000)
- FastAPI service for collecting Proxmox metrics
- RESTful API for system status
- Automatic data collection and storage

## API Endpoints

### Proxmox Monitor Service

**GET /health**
- Returns health status of the monitoring service
- Checks database and Proxmox connectivity

**POST /collect**
- Manually trigger system state collection
- Returns count of collected resources

**GET /status**
- Retrieve current system state
- Optional filters: `resource_type`, `status`, `node_name`

**GET /nodes**
- List configured Proxmox nodes

## Database Schema

The system uses the following main tables:
- `system_state` - Current state of all monitored resources
- `alerts` - Generated alerts for issues
- `conversations` - AI conversation history (Phase 3+)
- `action_logs` - Audit log of automated actions (Phase 6+)
- `system_health` - Health checks of monitoring components
- `configuration` - Dynamic configuration values

## Troubleshooting

### Service won't start
```bash
# View logs
docker-compose logs -f [service-name]

# Common issues:
# - Port already in use: Change ports in docker-compose.yml
# - Permission errors: Check file permissions
```

### Can't connect to Proxmox
```bash
# Test from within container
docker exec -it proxmox-ai-monitor curl -k https://YOUR_PROXMOX_HOST:8006/api2/json/nodes

# Verify API token/password in config/proxmox-nodes.yml
# Check firewall rules on Proxmox server
```

### Database connection errors
```bash
# Check PostgreSQL is running
docker-compose ps postgres

# Check database logs
docker-compose logs postgres

# Verify credentials match in .env and N8N
```

### N8N workflow not executing
- Ensure workflow is activated (toggle switch)
- Check PostgreSQL credentials are configured
- View execution logs in N8N interface

## Next Steps

Phase 1 provides the foundation. Future phases will add:
- **Phase 2**: Slack integration and alert routing
- **Phase 3**: LLM integration for conversational triage
- **Phase 4**: Voice alerts via Twilio and Sesame AI
- **Phase 5**: Advanced log analysis
- **Phase 6**: Automated remediation actions
- **Phase 7**: Comprehensive testing and hardening

## Security Notes

- Change all default passwords in `.env`
- Consider enabling SSL/TLS for production
- Restrict network access to monitoring ports
- Use API tokens instead of passwords when possible
- Regular backups of PostgreSQL database
- Keep Docker images updated

## Support

For issues or questions, refer to:
- N8N documentation: https://docs.n8n.io
- Proxmox API docs: https://pve.proxmox.com/pve-docs/api-viewer/
- FastAPI docs: https://fastapi.tiangolo.com
