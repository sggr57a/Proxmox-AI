# Security Hardening Guide

## Overview

This document outlines security best practices and hardening measures for the Proxmox AI Monitoring System.

## Secrets Management

### Environment Variables

**CRITICAL**: Never commit `.env` file to version control.

Required secrets in `.env`:
```bash
# Database
POSTGRES_PASSWORD=<strong-password>

# N8N
N8N_BASIC_AUTH_PASSWORD=<strong-password>

# Proxmox
PROXMOX_PASSWORD=<proxmox-password>
CONTAINER_DEFAULT_PASSWORD=<container-password>
LXC_DEFAULT_PASSWORD=<lxc-password>
VM_DEFAULT_PASSWORD=<vm-password>

# Slack
SLACK_BOT_TOKEN=xoxb-<your-token>

# Twilio
TWILIO_AUTH_TOKEN=<your-token>
```

### Password Requirements

- **Minimum 16 characters**
- **Mix of uppercase, lowercase, numbers, symbols**
- **Unique per service**
- **Rotate every 90 days**

### Using Docker Secrets (Recommended for Production)

Instead of environment variables, use Docker secrets:

```bash
# Create secrets
echo "my-strong-password" | docker secret create postgres_password -
echo "my-n8n-password" | docker secret create n8n_password -
echo "xoxb-token" | docker secret create slack_token -
echo "twilio-token" | docker secret create twilio_token -
```

Update `docker-compose.yml`:
```yaml
services:
  postgres:
    secrets:
      - postgres_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
      
secrets:
  postgres_password:
    external: true
  n8n_password:
    external: true
  slack_token:
    external: true
  twilio_token:
    external: true
```

## Network Security

### Firewall Configuration

**Only expose necessary ports:**

```bash
# Allow only from trusted networks
sudo ufw allow from 10.0.0.0/8 to any port 5678  # N8N UI
sudo ufw allow from 10.0.0.0/8 to any port 5432  # PostgreSQL
sudo ufw deny 8000:8004/tcp  # Block API services from external
```

### Reverse Proxy with TLS

Use Nginx/Traefik with Let's Encrypt:

```nginx
server {
    listen 443 ssl http2;
    server_name n8n.yourdomain.com;
    
    ssl_certificate /etc/letsencrypt/live/n8n.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Update N8N environment:
```bash
N8N_PROTOCOL=https
WEBHOOK_URL=https://n8n.yourdomain.com
```

### Service Isolation

Services communicate only within `proxmox-ai-network`:

```yaml
networks:
  proxmox-ai-network:
    driver: bridge
    internal: false  # Set to true to block external access
```

## Database Security

### PostgreSQL Hardening

**1. Restrict connections** (`database/init.sql`):
```sql
-- Already implemented: database user has limited privileges
-- Verify with:
SELECT * FROM pg_roles WHERE rolname = 'proxmox_user';
```

**2. Enable SSL** (production):
```yaml
postgres:
  environment:
    POSTGRES_HOST_SSL: "on"
  volumes:
    - ./certs/server.crt:/var/lib/postgresql/server.crt:ro
    - ./certs/server.key:/var/lib/postgresql/server.key:ro
```

**3. Regular backups** (encrypted):
```bash
# Daily encrypted backup
pg_dump -U proxmox_user proxmox_ai | \
  openssl enc -aes-256-cbc -salt -pbkdf2 -out backup-$(date +%F).sql.enc
```

## API Security

### Authentication

**All API services require authentication in production.**

Add API key authentication:

```python
# services/proxmox-monitor/monitor.py
from fastapi import Security, HTTPException
from fastapi.security import APIKeyHeader

API_KEY = os.getenv("API_KEY", "changeme")
api_key_header = APIKeyHeader(name="X-API-Key")

async def verify_api_key(api_key: str = Security(api_key_header)):
    if api_key != API_KEY:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return api_key

@app.get("/nodes", dependencies=[Security(verify_api_key)])
async def get_nodes():
    ...
```

Add to `.env`:
```bash
API_KEY=<generate-strong-random-key>
```

Generate strong API key:
```bash
openssl rand -hex 32
```

### Rate Limiting

**Prevent abuse** with rate limiting:

```python
# services/proxmox-monitor/monitor.py
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/nodes")
@limiter.limit("10/minute")
async def get_nodes(request: Request):
    ...
```

Add to `requirements.txt`:
```
slowapi==0.1.9
```

## Proxmox Access Control

### API Token (Preferred over Password)

Create API token in Proxmox:
```bash
# In Proxmox UI: Datacenter > Permissions > API Tokens
# Create token: proxmox-ai@pam!monitoring
```

Update `config/proxmox-nodes.yml`:
```yaml
nodes:
  - name: pve-node-01
    host: 10.0.1.10
    user: proxmox-ai@pam
    token_name: monitoring
    token_value: <token-secret>
    verify_ssl: false  # Set true if using valid cert
```

### Least Privilege

Grant minimum required permissions:
```bash
# In Proxmox CLI
pveum role add ProxmoxAI-Readonly -privs "VM.Audit,Datastore.Audit,Sys.Audit"
pveum aclmod / -role ProxmoxAI-Readonly -user proxmox-ai@pam
```

## Logging and Audit

### Sensitive Data Filtering

**Never log secrets:**

```python
# services/proxmox-monitor/monitor.py
import re

def sanitize_log(message: str) -> str:
    message = re.sub(r'password["\']?\s*[:=]\s*["\']?[^"\'&\s]+', 'password=***', message, flags=re.IGNORECASE)
    message = re.sub(r'token["\']?\s*[:=]\s*["\']?[^"\'&\s]+', 'token=***', message, flags=re.IGNORECASE)
    return message

logger.info(sanitize_log(f"Connecting to {host}"))
```

### Action Audit Trail

All actions are logged to `action_logs` table with:
- Timestamp
- User/system identifier
- Action type
- Entity affected
- Success/failure status
- IP address (if applicable)

Review logs:
```sql
SELECT * FROM action_logs 
WHERE action_type IN ('rebuild_container', 'modify_config', 'delete_resource')
ORDER BY created_at DESC 
LIMIT 100;
```

## Self-Protection Mechanisms

### Workflow Protection

**N8N workflows cannot modify themselves** - implemented in `self-protection.json`:

```json
{
  "prohibited_actions": [
    "delete_workflow",
    "modify_workflow",
    "stop_n8n",
    "delete_database"
  ]
}
```

### Rollback Safeguards

Before destructive actions:
1. **Create snapshot** (LVM or VM/LXC)
2. **Record action in database**
3. **Execute action**
4. **Verify success**
5. **Rollback if failed**

## Monitoring System Security

### Regular Security Audits

Run security scan:
```bash
# Check for exposed secrets
docker-compose config | grep -iE '(password|token|secret|key)' | grep -v '_FILE'

# Check container vulnerabilities
docker scan proxmox-ai-monitor
docker scan proxmox-ai-llm-engine

# Check outdated dependencies
docker-compose exec proxmox-monitor pip list --outdated
```

### Update Policy

- **Security patches**: Apply within 48 hours
- **Minor updates**: Monthly
- **Major updates**: Quarterly with testing

```bash
# Update all services
docker-compose pull
docker-compose up -d
```

## Incident Response

### Security Incident Procedure

1. **Isolate**: Stop affected containers
   ```bash
   docker-compose stop <service>
   ```

2. **Investigate**: Check logs
   ```bash
   docker-compose logs <service> --tail 1000
   ```

3. **Restore**: Rollback from snapshot
   ```bash
   ./scripts/snapshot-lvm.sh restore <snapshot-name>
   ```

4. **Report**: Document in incident log
   ```sql
   INSERT INTO incidents (alert_id, entity_type, symptoms, root_cause, actions_taken)
   VALUES ('security-001', 'system', 'Unauthorized access attempt', ...);
   ```

## Compliance

### Data Retention

- **Alerts**: 90 days
- **Incidents**: 12 months
- **Action logs**: 24 months (compliance)
- **Conversations**: 30 days

Automated cleanup:
```sql
-- Add to cron: daily at 2am
DELETE FROM alerts WHERE created_at < NOW() - INTERVAL '90 days';
DELETE FROM incidents WHERE created_at < NOW() - INTERVAL '12 months';
DELETE FROM conversations WHERE created_at < NOW() - INTERVAL '30 days';
```

### GDPR Compliance

User data anonymization:
```sql
UPDATE feedback 
SET user_email = 'anonymized@localhost', 
    user_name = 'anonymized'
WHERE created_at < NOW() - INTERVAL '12 months';
```

## Security Checklist

- [ ] Change all default passwords in `.env`
- [ ] Use Docker secrets for sensitive values
- [ ] Enable firewall (ufw/iptables)
- [ ] Configure TLS/SSL for N8N
- [ ] Use Proxmox API tokens (not passwords)
- [ ] Enable PostgreSQL SSL
- [ ] Add API key authentication to services
- [ ] Implement rate limiting
- [ ] Configure automated backups (encrypted)
- [ ] Set up log monitoring for security events
- [ ] Enable audit trail review
- [ ] Configure data retention policies
- [ ] Document incident response procedures
- [ ] Schedule regular security audits
- [ ] Keep all dependencies updated

## Resources

- [N8N Security Best Practices](https://docs.n8n.io/hosting/security/)
- [Docker Secrets](https://docs.docker.com/engine/swarm/secrets/)
- [Proxmox API Security](https://pve.proxmox.com/wiki/Proxmox_VE_API)
- [PostgreSQL Security](https://www.postgresql.org/docs/current/security.html)
