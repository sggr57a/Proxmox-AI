# Phase 3: AI Integration (Text-based) - Setup Guide

This guide covers the setup and configuration of the AI conversational system for Slack-based interaction with your Proxmox monitoring infrastructure.

## Overview

Phase 3 adds:
- **LLM Engine**: Local AI model (via Ollama) for intelligent triage and conversation
- **Slack Integration**: Chat with the AI about alerts and system state
- **Context-Aware Responses**: AI has access to alerts, system state, and conversation history
- **Action Suggestions**: AI can suggest and help execute remediation actions
- **Three Conversation Modes**:
  - **Triage**: Analyze alerts and suggest fixes
  - **Conversation**: General Q&A about infrastructure
  - **Approval**: Request approval for suggested actions

## Architecture

```
Slack Message → N8N Workflow → LLM Engine → Ollama → Response
                      ↓
                  Database
              (Context & History)
```

## Prerequisites

- Phases 1 & 2 completed (PostgreSQL, N8N, Proxmox Monitor, Alert Router)
- Slack workspace with bot configured
- At least 4GB RAM available for Ollama (8GB recommended)
- Docker Compose 2.x

## Setup Steps

### 1. Configure Environment Variables

Edit your `.env` file (copy from `.env.example` if needed):

```bash
# LLM Engine Configuration
OLLAMA_HOST=http://ollama:11434
OLLAMA_MODEL=llama3.2:3b

# Slack Configuration
SLACK_BOT_TOKEN=xoxb-your-bot-token-here
SLACK_CHANNEL_ID=C1234567890
```

**Getting Slack Credentials:**

1. Go to https://api.slack.com/apps
2. Create a new app or select existing one
3. Navigate to "OAuth & Permissions"
4. Add these Bot Token Scopes:
   - `chat:write`
   - `channels:history`
   - `channels:read`
   - `groups:history`
   - `groups:read`
   - `im:history`
   - `im:read`
5. Install app to workspace
6. Copy the "Bot User OAuth Token" (starts with `xoxb-`)
7. Invite bot to your channel: `/invite @YourBotName`
8. Get channel ID from channel details

### 2. Start Services

```bash
# Start Ollama and LLM Engine
docker-compose up -d ollama llm-engine

# Wait for services to be healthy
docker-compose ps
```

### 3. Pull AI Model

Run the setup script to download and test the AI model:

```bash
chmod +x scripts/setup-llm-engine.sh
./scripts/setup-llm-engine.sh
```

This will:
- Pull the `llama3.2:3b` model (~2GB download)
- Verify Ollama is working
- Test the LLM Engine API
- Run a sample chat completion

**Alternative Models:**

If you have more RAM, you can use larger models for better responses:

```bash
# 7B model (requires ~8GB RAM)
export OLLAMA_MODEL=llama3.2:7b
./scripts/setup-llm-engine.sh

# Or edit .env:
OLLAMA_MODEL=llama3.2:7b
```

### 4. Import N8N Workflow

1. Open N8N at http://localhost:5678
2. Login (default: admin / changeme)
3. Click "Workflows" → "Import from File"
4. Select `n8n/workflows/conversation-slack.json`
5. Click "Import"

### 5. Configure N8N Credentials

#### A. PostgreSQL Credentials

1. In the imported workflow, click any PostgreSQL node
2. Click "Credentials" → "Create New"
3. Name: `Proxmox AI DB`
4. Enter:
   - Host: `postgres`
   - Database: `proxmox_ai` (or your `POSTGRES_DB`)
   - User: `proxmox_user` (or your `POSTGRES_USER`)
   - Password: (your `POSTGRES_PASSWORD`)
   - Port: `5432`
5. Click "Save"

#### B. Slack API Credentials

1. Click any Slack node in the workflow
2. Click "Credentials" → "Create New"
3. Name: `Slack Bot`
4. Enter your `SLACK_BOT_TOKEN` (from step 1)
5. Click "Save"

### 6. Activate Workflow

1. In N8N, open the "Slack Conversation & AI Triage" workflow
2. Click the toggle in the top right to activate it
3. Workflow should show "Active" status

### 7. Test the Integration

#### Test 1: Basic Conversation

In your Slack channel, send:
```
@YourBot What's the status of the system?
```

Expected response: AI will summarize system state from database.

#### Test 2: Triage Mode

If you have an active alert, send:
```
@YourBot Tell me about alert #123
```

Or create a test alert:
```sql
INSERT INTO alerts (alert_type, severity, resource_type, resource_id, node_name, resource_name, title, description)
VALUES ('container_stopped', 'critical', 'container', 'nginx-test', 'pve-01', 'nginx-test', 'Container Stopped', 'nginx-test container has stopped unexpectedly');
```

Then ask:
```
@YourBot What's going on with nginx-test?
```

Expected: AI analyzes the alert and suggests actions (restart container, check logs, etc.)

#### Test 3: Action Approval

When AI suggests actions, you'll see interactive buttons. Click "Approve" to execute.

> **Note:** Action execution requires Phase 6 (Action Executor workflow). For Phase 3, approval buttons will appear but won't execute actions yet.

## Conversation Examples

### General Status Check
```
User: How many alerts are active?
AI: You currently have 3 active alerts: 2 warnings and 1 critical.
```

### Alert Triage
```
User: Container nginx keeps restarting
AI: 
Root Cause: Container restart loop likely due to configuration error or missing dependency

Impact: High - Web services are degraded

Recommended Actions:
1. Check container logs for error messages
2. Verify port 80/443 are not in use by another service
3. Restart container with docker logs monitoring

Prevention: Review container health checks and dependencies

Estimated Time: 10-15 minutes
```

### Follow-up Questions
```
User: How do I check the logs?
AI: You can check logs with:
`docker logs nginx -n 100`

Or in Proxmox:
1. Navigate to the container
2. Click "Console" or "Logs"
```

## Conversation Modes

The AI automatically switches modes based on context:

| Mode | Trigger | Purpose |
|------|---------|---------|
| **Triage** | Mention alert number or resource having issues | Diagnose problems, suggest fixes |
| **Conversation** | General questions | Answer questions, explain system state |
| **Approval** | (Future) | Request approval before executing actions |

## Troubleshooting

### LLM Engine not responding

```bash
# Check logs
docker logs proxmox-ai-llm-engine

# Restart service
docker-compose restart llm-engine
```

### Ollama model not loaded

```bash
# Check available models
docker exec proxmox-ai-ollama ollama list

# Pull model manually
docker exec proxmox-ai-ollama ollama pull llama3.2:3b
```

### Slack messages not triggering workflow

1. Verify workflow is active in N8N
2. Check Slack credentials are correct
3. Ensure bot is invited to the channel
4. Check N8N logs: `docker logs proxmox-ai-n8n`

### AI responses are slow

- Smaller model (`llama3.2:1b`) is faster but less accurate
- Larger model (`llama3.2:7b`) is slower but more accurate
- Consider GPU acceleration for Ollama (requires NVIDIA GPU setup)

### Database connection errors

```bash
# Verify PostgreSQL is running
docker ps | grep postgres

# Test connection
docker exec proxmox-ai-postgres psql -U proxmox_user -d proxmox_ai -c "SELECT COUNT(*) FROM alerts;"
```

## Performance Tuning

### Adjust LLM Parameters

In the N8N workflow, edit the "Call LLM Engine" node:

- **Temperature** (0.0-2.0): Lower = more focused, Higher = more creative
  - Recommended: 0.7 for triage, 0.5 for general conversation
- **Max Tokens** (100-4000): Maximum response length
  - Recommended: 2000 for detailed explanations, 500 for quick responses

### Ollama Optimization

For faster inference, edit `docker-compose.yml`:

```yaml
ollama:
  # ... existing config ...
  environment:
    - OLLAMA_NUM_PARALLEL=4       # Parallel requests
    - OLLAMA_MAX_LOADED_MODELS=2   # Keep models in memory
  deploy:
    resources:
      limits:
        memory: 8G                 # Allocate more RAM
```

## Next Steps

Once Phase 3 is working:

1. **Phase 4**: Add voice integration (Twilio + Sesame TTS)
2. **Phase 5**: Enhance monitoring with log analysis
3. **Phase 6**: Enable action execution (restart containers, rebuild, etc.)

## API Documentation

### LLM Engine API

**Base URL:** `http://localhost:8001`

#### Health Check
```bash
curl http://localhost:8001/health
```

#### Chat Completion
```bash
curl -X POST http://localhost:8001/api/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "What causes high CPU usage?"}
    ],
    "mode": "conversation",
    "temperature": 0.7
  }'
```

#### List Models
```bash
curl http://localhost:8001/api/models
```

## Security Considerations

1. **Slack Token**: Keep `SLACK_BOT_TOKEN` secret, never commit to git
2. **Bot Permissions**: Only grant minimum required Slack scopes
3. **Channel Access**: Bot only sees channels it's invited to
4. **Action Approval**: All destructive actions require approval (configured in prompts)
5. **Conversation Privacy**: All messages stored in PostgreSQL (consider encryption for sensitive data)

## Resources

- [Ollama Documentation](https://ollama.ai/docs)
- [N8N Slack Integration](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/)
- [Llama 3.2 Model Info](https://ollama.ai/library/llama3.2)

## Support

If you encounter issues:

1. Check logs: `docker-compose logs -f llm-engine ollama`
2. Verify all services healthy: `docker-compose ps`
3. Test LLM Engine: `curl http://localhost:8001/health`
4. Review N8N workflow execution history
