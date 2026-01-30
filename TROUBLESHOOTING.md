# Troubleshooting Guide

Common issues and solutions for the Proxmox AI Monitoring & Management System.

## Table of Contents

- [Service Health Issues](#service-health-issues)
- [Database Problems](#database-problems)
- [N8N Workflow Issues](#n8n-workflow-issues)
- [Proxmox Connectivity](#proxmox-connectivity)
- [Alert Routing Problems](#alert-routing-problems)
- [LLM and AI Issues](#llm-and-ai-issues)
- [Voice/Twilio Problems](#voicetwilio-problems)
- [Learning System Issues](#learning-system-issues)
- [Performance Problems](#performance-problems)
- [Recovery Procedures](#recovery-procedures)

## Service Health Issues

### Services Won't Start

**Symptom:** `docker-compose up -d` fails or services immediately exit.

**Diagnosis:**
```bash
docker-compose ps
docker-compose logs <service-name>
```

**Common Causes:**

**1. Port already in use**
```
Error: bind: address already in use
```
**Solution:**
```bash
# Find process using port
lsof -i :5678  # or any other port
# Kill process or change port in docker-compose.yml
```

**2. Missing .env file**
```
ERROR: The Compose file is invalid because:
Service "postgres" uses an undefined variable
```
**Solution:**
```bash
cp .env.example .env
nano .env  # Configure variables
```

**3. Database not ready**
```
psycopg2.OperationalError: could not connect to server
```
**Solution:**
```bash
# Wait for PostgreSQL to fully start
docker-compose up -d postgres
sleep 10
docker-compose up -d
```

### Service Unhealthy

**Symptom:** `docker ps` shows service as "unhealthy"

**Diagnosis:**
```bash
docker inspect --format='{{json .State.Health}}' <container-name> | jq
```

**Solutions:**

**Proxmox Monitor unhealthy:**
```bash
# Check logs
docker-compose logs proxmox-monitor | tail -50

# Verify Proxmox connectivity
docker-compose exec proxmox-monitor curl -k https://10.0.1.10:8006

# Restart service
docker-compose restart proxmox-monitor
```

**LLM Engine unhealthy:**
```bash
# Check if Ollama is ready
curl http://localhost:11434/api/tags

# Check if model is pulled
docker-compose exec ollama ollama list

# Pull model manually
docker-compose exec ollama ollama pull llama3.2:3b
```

## Database Problems

### Cannot Connect to Database

**Symptom:** Services fail with database connection errors.

**Diagnosis:**
```bash
docker-compose exec postgres psql -U proxmox_user -d proxmox_ai -c "SELECT 1"
```

**Solutions:**

**1. Wrong credentials**
```bash
# Verify .env matches docker-compose.yml
grep POSTGRES .env
# Update and restart
docker-compose restart postgres
```

**2. Database not initialized**
```bash
# Check if tables exist
docker-compose exec postgres psql -U proxmox_user -d proxmox_ai -c "\dt"

# If empty, reinitialize
docker-compose down -v  # WARNING: Deletes data!
docker-compose up -d postgres
```

**3. Connection pool exhausted**
```
FATAL:  sorry, too many clients already
```
**Solution:**
```sql
-- Increase max connections
docker-compose exec postgres psql -U postgres -c "ALTER SYSTEM SET max_connections = 200;"
docker-compose restart postgres
```

### Database Performance Issues

**Symptom:** Slow queries, high CPU usage on postgres container.

**Diagnosis:**
```sql
-- Check slow queries
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '5 seconds';

-- Check table sizes
SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

**Solutions:**

**1. Missing indexes**
```sql
-- Already created in init.sql, but verify:
\d+ alerts
\d+ incidents

-- Create missing indexes
CREATE INDEX CONCURRENTLY idx_alerts_created_at ON alerts(created_at);
```

**2. Vacuuming needed**
```bash
docker-compose exec postgres vacuumdb -U proxmox_user -d proxmox_ai -z -v
```

**3. Too much historical data**
```sql
-- Clean old data
DELETE FROM alerts WHERE created_at < NOW() - INTERVAL '90 days';
DELETE FROM action_logs WHERE created_at < NOW() - INTERVAL '180 days';
VACUUM FULL;
```

## N8N Workflow Issues

### Workflows Not Executing

**Symptom:** Workflows don't trigger or execute.

**Diagnosis:**
1. Check N8N UI → Executions for errors
2. Check if workflow is activated (toggle switch)
3. Review logs: `docker-compose logs n8n`

**Solutions:**

**1. Workflow not activated**
```
# In N8N UI, ensure workflow is "Active" (toggle on)
```

**2. Credential errors**
```
NodeApiError: Error: connect ECONNREFUSED
```
**Solution:** Configure credentials in N8N:
- Settings → Credentials
- Add PostgreSQL, Slack, etc.
- Test connection before saving

**3. Trigger not configured**
```bash
# For poll-based workflows, ensure interval is set
# For webhook workflows, ensure webhook URL is accessible
curl http://localhost:5678/webhook/<webhook-path>
```

### Workflow Execution Failures

**Symptom:** Workflow executes but nodes fail.

**Common Node Errors:**

**PostgreSQL node fails:**
```
ERROR: permission denied for table alerts
```
**Solution:**
```sql
GRANT ALL ON ALL TABLES IN SCHEMA public TO proxmox_user;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO proxmox_user;
```

**HTTP Request node fails:**
```
ERROR: connect ECONNREFUSED 127.0.0.1:8000
```
**Solution:**
- Change `localhost` to service name (e.g., `proxmox-monitor`)
- Ensure target service is running
- Check network: `docker-compose exec n8n ping proxmox-monitor`

**Code node fails:**
```
ReferenceError: <variable> is not defined
```
**Solution:** Review JavaScript code, check input/output bindings

## Proxmox Connectivity

### Cannot Reach Proxmox API

**Symptom:** Monitoring workflows fail with connection errors.

**Diagnosis:**
```bash
# From monitor container
docker-compose exec proxmox-monitor curl -k https://10.0.1.10:8006/api2/json/version

# Check config
docker-compose exec proxmox-monitor cat /app/config/proxmox-nodes.yml
```

**Solutions:**

**1. Wrong credentials**
```yaml
# config/proxmox-nodes.yml
nodes:
  - name: pve-node-01
    host: 10.0.1.10
    user: root@pam
    password: correct_password  # Update this
```

**2. SSL certificate issues**
```yaml
verify_ssl: false  # For self-signed certs
```

**3. Firewall blocking**
```bash
# On Proxmox node, allow traffic from Docker network
iptables -A INPUT -s 172.0.0.0/8 -p tcp --dport 8006 -j ACCEPT
```

**4. Network isolation**
```bash
# Verify Docker network connectivity
docker-compose exec proxmox-monitor ping 10.0.1.10
# If fails, check Docker network settings
```

### API Authentication Failures

**Symptom:** `401 Unauthorized` or `403 Forbidden` errors.

**Solutions:**

**Using API tokens:**
```yaml
# Create token in Proxmox UI first
nodes:
  - name: pve-node-01
    user: proxmox-ai@pam
    token_name: monitoring
    token_value: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

**Permission issues:**
```bash
# In Proxmox CLI, grant permissions:
pveum role add ProxmoxAI -privs "VM.Audit,Datastore.Audit,Sys.Audit"
pveum aclmod / -role ProxmoxAI -user proxmox-ai@pam
```

## Alert Routing Problems

### Not Receiving Slack Notifications

**Symptom:** Alerts generated but no Slack messages.

**Diagnosis:**
```bash
# Check alerts in database
docker-compose exec postgres psql -U proxmox_user -d proxmox_ai \
  -c "SELECT * FROM alerts WHERE status='active' ORDER BY created_at DESC LIMIT 5;"

# Check N8N workflow execution
# N8N UI → Executions → alert-router
```

**Solutions:**

**1. Invalid Slack token**
```bash
# Test token
curl -X POST https://slack.com/api/auth.test \
  -H "Authorization: Bearer xoxb-YOUR-TOKEN"

# Update .env with correct token
SLACK_BOT_TOKEN=xoxb-correct-token
docker-compose restart n8n
```

**2. Wrong channel ID**
```bash
# Get channel ID: Right-click channel → View details → Copy ID
# Update .env
SLACK_CHANNEL_ID=C0123456789
```

**3. Bot not in channel**
```
error: not_in_channel
```
**Solution:** In Slack, invite bot to channel: `/invite @YourBotName`

**4. Workflow not active**
```
# Activate in N8N UI: alert-router workflow → Toggle "Active"
```

### Voice Alerts Not Working

**Symptom:** No phone calls received during off-hours.

**Diagnosis:**
```bash
# Check Twilio Bridge logs
docker-compose logs twilio-bridge | grep -i error

# Verify Twilio credentials
docker-compose exec twilio-bridge env | grep TWILIO
```

**Solutions:**

**1. Invalid Twilio credentials**
```bash
# Test credentials
curl -X GET "https://api.twilio.com/2010-04-01/Accounts/<SID>" \
  -u "<SID>:<AUTH_TOKEN>"

# Update .env
TWILIO_ACCOUNT_SID=correct_sid
TWILIO_AUTH_TOKEN=correct_token
docker-compose restart twilio-bridge
```

**2. Webhook not accessible**
```
Twilio cannot reach http://your-server:8003/webhook
```
**Solution:**
- Use ngrok for testing: `ngrok http 8003`
- Configure Twilio webhook with ngrok URL
- For production, use public IP or domain

**3. Phone number format**
```bash
# Must be E.164 format
ALERT_PHONE_NUMBER=+1234567890  # Correct
ALERT_PHONE_NUMBER=123-456-7890 # Wrong
```

## LLM and AI Issues

### LLM Not Responding

**Symptom:** Conversations timeout or return empty responses.

**Diagnosis:**
```bash
# Check Ollama
curl http://localhost:11434/api/tags

# Test inference
curl -X POST http://localhost:8001/inference \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Hello", "max_tokens": 50}'
```

**Solutions:**

**1. Model not loaded**
```bash
# List models
docker-compose exec ollama ollama list

# Pull model
docker-compose exec ollama ollama pull llama3.2:3b

# Verify
curl http://localhost:11434/api/tags | jq '.models[].name'
```

**2. Out of memory**
```
Error: Out of memory
```
**Solution:**
```yaml
# docker-compose.yml - Increase memory limit or use smaller model
ollama:
  deploy:
    resources:
      limits:
        memory: 8G
```
Or switch to smaller model:
```bash
OLLAMA_MODEL=llama3.2:3b  # Instead of llama3:8b
```

**3. LLM Engine not connecting to Ollama**
```bash
# Check connectivity
docker-compose exec llm-engine curl http://ollama:11434/api/tags

# Verify environment
docker-compose exec llm-engine env | grep OLLAMA
```

### Slow AI Responses

**Symptom:** LLM takes >30 seconds to respond.

**Solutions:**

**1. Use smaller model**
```bash
OLLAMA_MODEL=llama3.2:3b  # Faster, requires 4GB RAM
# Instead of llama3:8b (requires 8GB RAM)
```

**2. Reduce max_tokens**
```python
# services/llm-engine/app.py
DEFAULT_MAX_TOKENS = 256  # Reduce from 512
```

**3. Use GPU acceleration** (if available)
```yaml
# docker-compose.yml
ollama:
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
```

## Voice/Twilio Problems

### TTS Not Generating Audio

**Symptom:** Voice calls fail or audio is silent.

**Diagnosis:**
```bash
# Test TTS directly
curl -X POST http://localhost:8002/synthesize \
  -H "Content-Type: application/json" \
  -d '{"text": "Test message", "voice": "default"}'
```

**Solutions:**

**1. Sesame model not downloaded**
```bash
# Download manually
docker-compose exec sesame-tts python download_model.py

# Check model files
docker-compose exec sesame-tts ls -lh /app/models/
```

**2. Out of disk space**
```
Error: No space left on device
```
**Solution:**
```bash
# Clean up Docker
docker system prune -a --volumes
# Clean logs
truncate -s 0 /var/lib/docker/containers/*/*-json.log
```

## Learning System Issues

### Similarity Search Returns No Results

**Symptom:** RAG system doesn't find similar incidents.

**Diagnosis:**
```bash
# Check incident count
curl http://localhost:8004/incidents/count

# Test search
curl -X POST http://localhost:8004/incidents/search \
  -H "Content-Type: application/json" \
  -d '{"query": "container restart", "limit": 5}'
```

**Solutions:**

**1. No historical data**
```bash
# Seed learning data
./scripts/seed-learning-data.sh
```

**2. Embeddings not generated**
```bash
# Regenerate embeddings
curl -X POST http://localhost:8004/embeddings/regenerate
```

**3. ChromaDB issues**
```bash
# Check learning engine logs
docker-compose logs learning-engine | grep -i error

# Restart and rebuild embeddings
docker-compose restart learning-engine
sleep 5
curl -X POST http://localhost:8004/embeddings/regenerate
```

### Patterns Not Detected

**Symptom:** Pattern detection returns empty results.

**Solution:**
```bash
# Need sufficient data (20+ incidents minimum)
./scripts/seed-learning-data.sh

# Manually trigger pattern detection
curl -X POST http://localhost:8004/patterns/detect
```

## Performance Problems

### High CPU Usage

**Diagnosis:**
```bash
docker stats
```

**Solutions:**

**1. LLM consuming CPU**
- Use smaller model
- Reduce concurrent inference requests
- Add rate limiting

**2. Database CPU high**
```bash
# Analyze slow queries
docker-compose exec postgres psql -U proxmox_user -d proxmox_ai \
  -c "SELECT query, calls, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"
```

**3. Log analysis overwhelming**
- Increase monitoring intervals in N8N workflows
- Filter logs before sending to LLM
- Reduce log retention period

### High Memory Usage

**Diagnosis:**
```bash
free -h
docker stats --no-stream --format "table {{.Container}}\t{{.MemUsage}}"
```

**Solutions:**

**1. Ollama/LLM**
```yaml
# Use smaller model or limit memory
ollama:
  deploy:
    resources:
      limits:
        memory: 6G
```

**2. PostgreSQL**
```sql
-- Reduce shared_buffers
ALTER SYSTEM SET shared_buffers = '256MB';
-- Restart required
```

**3. ChromaDB (Learning Engine)**
```bash
# Clear old embeddings
curl -X POST http://localhost:8004/embeddings/clear-old?days=90
```

## Recovery Procedures

### Complete System Reset

**WARNING:** This deletes all data!

```bash
# Stop all services
docker-compose down -v

# Clean up
docker system prune -a --volumes -f

# Rebuild
./scripts/setup.sh
```

### Restore from Backup

**Database restore:**
```bash
# Stop services
docker-compose stop

# Restore database
cat backup-YYYY-MM-DD.sql | docker-compose exec -T postgres \
  psql -U proxmox_user -d proxmox_ai

# Restart
docker-compose start
```

**Configuration restore:**
```bash
tar -xzf config-backup-YYYY-MM-DD.tar.gz
docker-compose restart
```

### Emergency Snapshot Rollback

```bash
# List snapshots
./scripts/snapshot-lvm.sh list

# Restore from snapshot
./scripts/snapshot-lvm.sh restore <snapshot-name>

# Verify
docker-compose ps
./scripts/test-integration.sh
```

## Getting Help

### Collect Diagnostic Information

```bash
# System info
docker-compose version
docker version
uname -a

# Service status
docker-compose ps

# Recent logs (last 100 lines per service)
docker-compose logs --tail=100 > diagnostic-logs.txt

# Database status
docker-compose exec postgres psql -U proxmox_user -d proxmox_ai \
  -c "SELECT COUNT(*) as alert_count FROM alerts;" \
  -c "SELECT COUNT(*) as incident_count FROM incidents;"

# Test connectivity
./scripts/test-integration.sh > test-results.txt 2>&1
```

### Enable Debug Logging

```bash
# Update .env
LOG_LEVEL=DEBUG

# Restart services
docker-compose restart

# Watch logs
docker-compose logs -f
```

### Report Issues

Include in issue report:
1. System information (OS, Docker version)
2. Service status (`docker-compose ps`)
3. Error messages from logs
4. Steps to reproduce
5. Expected vs actual behavior
6. Diagnostic logs (from above)
