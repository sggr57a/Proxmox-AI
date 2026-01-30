# Configuration Guide

Complete configuration reference for the Proxmox AI Monitoring & Management System.

## Table of Contents

- [Environment Variables](#environment-variables)
- [Proxmox Nodes Configuration](#proxmox-nodes-configuration)
- [Alert Rules](#alert-rules)
- [Action Policies](#action-policies)
- [Naming Conventions](#naming-conventions)
- [N8N Workflow Configuration](#n8n-workflow-configuration)
- [Advanced Settings](#advanced-settings)

## Environment Variables

### Required Configuration

Edit `.env` file (created from `.env.example`):

#### Database Settings
```bash
POSTGRES_DB=proxmox_ai
POSTGRES_USER=proxmox_user
POSTGRES_PASSWORD=your_secure_password_here
```

#### N8N Settings
```bash
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_secure_password_here
N8N_HOST=0.0.0.0
N8N_PORT=5678
N8N_PROTOCOL=http  # Change to https in production
WEBHOOK_URL=http://localhost:5678  # Or your public URL
GENERIC_TIMEZONE=America/New_York
```

#### Proxmox Credentials
```bash
PROXMOX_USER=root@pam
PROXMOX_PASSWORD=your_proxmox_password
PROXMOX_VERIFY_SSL=false  # Set true if using valid SSL cert

# Default credentials for containers/VMs
CONTAINER_DEFAULT_USER=root
CONTAINER_DEFAULT_PASSWORD=your_container_password
LXC_DEFAULT_PASSWORD=your_lxc_password
VM_DEFAULT_PASSWORD=your_vm_password
```

#### LLM Engine
```bash
OLLAMA_HOST=http://ollama:11434
OLLAMA_MODEL=llama3.2:3b  # Or llama3:8b, mistral, etc.
```

#### Slack Integration
```bash
SLACK_BOT_TOKEN=xoxb-your-slack-bot-token
SLACK_CHANNEL_ID=C1234567890
```

**How to get Slack credentials:**
1. Go to https://api.slack.com/apps
2. Create new app → "From scratch"
3. Add Bot Token Scopes: `chat:write`, `chat:write.public`, `im:write`, `channels:history`
4. Install app to workspace
5. Copy "Bot User OAuth Token" (starts with `xoxb-`)
6. Get channel ID: Right-click channel → View channel details → Copy ID

#### Twilio Integration
```bash
TWILIO_ACCOUNT_SID=your_account_sid
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_PHONE_NUMBER=+1234567890
ALERT_PHONE_NUMBER=+1987654321  # Your phone for receiving alerts
```

**How to get Twilio credentials:**
1. Go to https://www.twilio.com/console
2. Copy Account SID and Auth Token
3. Buy a phone number (Phone Numbers → Buy a Number)
4. Configure webhook URL: `http://your-server:8003/webhook`

#### Voice Services
```bash
SESAME_TTS_URL=http://sesame-tts:8002
LLM_ENGINE_URL=http://llm-engine:8001
N8N_WEBHOOK_URL=http://n8n:5678
```

#### Business Hours
```bash
BUSINESS_HOURS_START=7
BUSINESS_HOURS_END=16
BUSINESS_DAYS=1,2,3,4,5  # Monday-Friday
```

#### Service Settings
```bash
MONITOR_API_HOST=0.0.0.0
MONITOR_API_PORT=8000
LOG_LEVEL=INFO  # DEBUG, INFO, WARNING, ERROR
```

## Proxmox Nodes Configuration

Edit `config/proxmox-nodes.yml`:

### Basic Configuration

```yaml
nodes:
  - name: pve-node-01
    host: 10.0.1.10
    user: root@pam
    password: your_password
    verify_ssl: false
```

### Using API Tokens (Recommended)

```yaml
nodes:
  - name: pve-node-01
    host: 10.0.1.10
    user: proxmox-ai@pam
    token_name: monitoring
    token_value: your-token-value
    verify_ssl: false
```

**Create API token in Proxmox:**
1. Datacenter → Permissions → API Tokens
2. Add token: User `proxmox-ai@pam`, Token ID `monitoring`
3. Uncheck "Privilege Separation" (or create appropriate role)
4. Copy token secret

### Multiple Nodes

```yaml
nodes:
  - name: pve-node-01
    host: 10.0.1.10
    user: root@pam
    password: password1
    verify_ssl: false
    
  - name: pve-node-02
    host: 10.0.1.11
    user: root@pam
    password: password2
    verify_ssl: false
    
  - name: pve-node-03
    host: 10.0.1.12
    user: root@pam
    password: password3
    verify_ssl: false
```

## Alert Rules

Edit `config/alert-rules.yml`:

### Deduplication Settings

```yaml
deduplication:
  window_seconds: 300  # 5 minutes
  group_by:
    - resource_type
    - resource_id
    - alert_type
    - node_name
```

### Alert Storm Protection

```yaml
alert_storm:
  threshold: 20  # alerts per minute
  action: batch  # or 'throttle', 'suppress'
  batch_window_seconds: 60
```

### Business Hours

```yaml
business_hours:
  start: 7  # 7am
  end: 16   # 4pm
  days: [1, 2, 3, 4, 5]  # Mon-Fri
  timezone: America/New_York
```

### Alert Types

**Container Stopped:**
```yaml
container_stopped:
  severity: critical
  auto_escalate: true
  escalate_after_minutes: 5
  suggested_actions:
    - restart_container
```

**High CPU:**
```yaml
high_cpu:
  severity: warning
  auto_escalate: false
  threshold:
    percentage: 90
    duration_minutes: 5
  suggested_actions:
    - check_processes
    - scale_up
```

**Restart Loop:**
```yaml
restart_loop:
  severity: critical
  auto_escalate: true
  escalate_after_minutes: 1
  threshold:
    restart_count: 3
    time_window_minutes: 5
  suggested_actions:
    - check_logs
    - manual_intervention_required
```

### Severity Levels

```yaml
severity_levels:
  critical:
    always_notify: true
    voice_threshold_minutes: 2
    retry_count: 3
    retry_interval_minutes: 5
  
  warning:
    always_notify: false
    voice_threshold_minutes: 10
    retry_count: 2
    retry_interval_minutes: 10
  
  info:
    always_notify: false
    voice_threshold_minutes: null
    retry_count: 1
```

### Routing Rules

```yaml
routing:
  business_hours:
    primary_channel: slack
    fallback_channel: voice
    fallback_after_minutes: 5
  
  after_hours:
    primary_channel: voice
    fallback_channel: slack
    fallback_after_minutes: 0
  
  critical_always:
    channels:
      - slack
      - voice
    simultaneous: true
```

## Action Policies

Edit `config/action-policies.yml`:

### Autonomous Actions (No Approval)

```yaml
autonomous_actions:
  - restart_container
  - start_container
  - start_vm
  - start_lxc
  - clear_temp_files
  - rotate_logs
  - create_snapshot_pre_action
```

### Approval Required Actions

```yaml
approval_required:
  - rebuild_container
  - modify_config
  - restart_node
  - network_config_change
  - restore_backup
  - clone_vm
  - delete_resource
```

### Prohibited Actions

```yaml
prohibited_actions:
  - delete_node
  - delete_storage
  - format_disk
  - delete_n8n_workflow
  - modify_monitoring_system
```

### Action Limits

```yaml
action_limits:
  max_restart_attempts: 3
  max_rebuild_attempts: 1
  restart_cooldown_minutes: 5
  alert_storm_threshold: 20
```

### Snapshot Retention

```yaml
snapshot_retention:
  lvm:
    max_age_days: 30
    max_snapshots_per_volume: 10
    auto_cleanup: true
  vm:
    max_age_days: 7
    max_snapshots_per_vm: 5
    auto_cleanup: true
```

### Backup Retention

```yaml
backup_retention:
  default_retention_days: 30
  max_backups_per_resource: 10
  auto_verify_backups: true
  verification_interval_days: 7
  auto_cleanup_expired: true
```

### Replication Settings

```yaml
replication:
  primary_node: "pve-node-01"
  auto_replicate_from_primary: true
  cross_node_replication: true
  auto_network_assignment: true
  naming_convention_enforcement: true
```

### Self-Protection

```yaml
self_protection:
  protected_containers:
    - "proxmox-ai-n8n"
    - "proxmox-ai-postgres"
    - "proxmox-ai-monitor"
  
  protected_workflows:
    - "monitoring-health-check"
    - "alert-router"
    - "self-protection"
  
  prevent_self_modification: true
  canary_monitoring_enabled: true
```

## Naming Conventions

Edit `config/naming-conventions.yml`:

```yaml
naming_patterns:
  lxc:
    pattern: "pve-{node}-{type}-{seq}"
    examples:
      - "pve-node-01-web-01"
      - "pve-node-01-db-02"
  
  vm:
    pattern: "pve-{node}-vm-{seq}"
    examples:
      - "pve-node-01-vm-100"
      - "pve-node-02-vm-200"

node_suffixes:
  - "01"
  - "02"
  - "03"

resource_types:
  - web
  - db
  - cache
  - app
  - test
  - av
  - backup

network_config:
  subnets:
    pve-node-01: "10.0.1.0/24"
    pve-node-02: "10.0.2.0/24"
    pve-node-03: "10.0.3.0/24"
  
  ip_allocation:
    mode: sequential
    start_ip: 10
    reserved_ips: [1, 2, 3, 254, 255]
```

## N8N Workflow Configuration

### Importing Workflows

1. Open N8N UI: http://localhost:5678
2. Click "Workflows" → "Import from File"
3. Import each workflow from `n8n/workflows/`
4. Configure credentials for each workflow

### Required Credentials

**PostgreSQL:**
- Name: `Proxmox AI DB`
- Type: PostgreSQL
- Host: `postgres` (container name)
- Database: `proxmox_ai`
- User: `proxmox_user`
- Password: (from `.env`)

**Slack:**
- Name: `Slack Bot`
- Type: Slack API
- OAuth Access Token: (from `.env` - `SLACK_BOT_TOKEN`)

**HTTP Auth (for internal APIs):**
- Name: `Proxmox Monitor API`
- Type: Header Auth
- Name: `X-API-Key`
- Value: (generate with `openssl rand -hex 32`)

### Workflow Activation

After importing, activate workflows:
1. `monitoring-health-check` - Start this first
2. `monitoring-log-analysis`
3. `monitoring-network`
4. `alert-router`
5. `action-executor`
6. `self-protection`

**Testing:**
- Manually trigger `monitoring-health-check` to verify
- Check execution history for errors
- Review logs: `docker-compose logs n8n`

## Advanced Settings

### Database Connection Pooling

For high load environments, adjust in `docker-compose.yml`:

```yaml
proxmox-monitor:
  environment:
    DATABASE_URL: postgresql://user:pass@postgres:5432/proxmox_ai?max_connections=20&pool_timeout=10
```

### LLM Model Selection

Change model in `.env`:
```bash
# Smaller, faster (recommended for <16GB RAM)
OLLAMA_MODEL=llama3.2:3b

# Better quality (requires 16GB+ RAM)
OLLAMA_MODEL=llama3:8b

# Alternative models
OLLAMA_MODEL=mistral:7b
OLLAMA_MODEL=codellama:13b
```

### Log Verbosity

Adjust in `.env`:
```bash
LOG_LEVEL=DEBUG  # Very detailed
LOG_LEVEL=INFO   # Normal (recommended)
LOG_LEVEL=WARNING  # Only warnings/errors
LOG_LEVEL=ERROR  # Only errors
```

### Performance Tuning

**Monitoring Intervals:**

Edit workflow schedules in N8N:
- Health checks: 1-5 minutes (default: 2 min)
- Log analysis: 30-60 seconds (default: 30 sec)
- Network monitoring: 2-10 minutes (default: 5 min)

**Resource Limits:**

Edit `docker-compose.yml`:
```yaml
services:
  llm-engine:
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 8G
        reservations:
          cpus: '2.0'
          memory: 4G
```

### Timezone Configuration

System timezone in `.env`:
```bash
GENERIC_TIMEZONE=America/New_York
# Other examples:
# Europe/London
# Asia/Tokyo
# UTC
```

### Custom Prompts

Edit LLM prompts in `services/llm-engine/prompts/`:

**Triage Prompt** (`triage.txt`):
```
You are an expert system administrator analyzing a Proxmox infrastructure alert.

Alert Details:
{alert_details}

System State:
{system_state}

Past Similar Incidents:
{similar_incidents}

Analyze the alert and:
1. Identify the likely root cause
2. Suggest 2-3 remediation steps
3. Assess severity and urgency

Be concise and actionable.
```

### Backup Storage Configuration

Configure backup locations in N8N workflow `backup-orchestration.json` or via API:

```bash
curl -X POST http://localhost:8000/backup-locations \
  -H "Content-Type: application/json" \
  -d '{
    "name": "local",
    "type": "directory",
    "path": "/mnt/backups",
    "max_size_gb": 500
  }'
```

## Validation

### Verify Configuration

```bash
# Test database connection
docker-compose exec postgres psql -U proxmox_user -d proxmox_ai -c "SELECT 1"

# Test Proxmox connectivity
curl http://localhost:8000/nodes

# Test LLM
curl -X POST http://localhost:8001/inference \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Test", "max_tokens": 10}'

# Test Learning Engine
curl http://localhost:8004/health

# Run full integration tests
./scripts/test-integration.sh
```

### Configuration Backup

Backup your configuration:
```bash
tar -czf proxmox-ai-config-$(date +%F).tar.gz \
  config/ \
  .env \
  n8n/workflows/
```

### Restore Configuration

```bash
tar -xzf proxmox-ai-config-YYYY-MM-DD.tar.gz
docker-compose restart
```

## Troubleshooting Configuration Issues

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common configuration problems and solutions.

## Next Steps

After configuration:
1. Run `./scripts/setup.sh` to apply changes
2. Run `./scripts/test-integration.sh` to verify
3. Check N8N workflows are executing
4. Monitor first alerts in Slack
5. Test voice alerting by triggering off-hours alert
