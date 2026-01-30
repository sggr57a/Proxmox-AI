# Phase 4 - Voice Integration Setup Guide

This guide covers the setup and configuration of the voice integration features for the Proxmox AI monitoring system, including Sesame AI TTS and Twilio voice calls.

## Overview

Phase 4 adds:
- **Sesame AI TTS Service**: Text-to-speech using the Sesame CSM 1B model
- **Twilio Bridge Service**: Voice call initiation and conversation handling
- **Voice Conversation Workflow**: N8N workflow for managing voice-based alerts and conversations

## Prerequisites

1. **Twilio Account**:
   - Sign up at [twilio.com](https://www.twilio.com)
   - Get your Account SID and Auth Token
   - Purchase a phone number with voice capabilities

2. **Public Webhook URL** (for Twilio callbacks):
   - Use ngrok: `ngrok http 8003`
   - Or configure your firewall/router to forward port 8003
   - Or deploy behind a reverse proxy with HTTPS

3. **Previous Phases**:
   - Phase 1: Core infrastructure
   - Phase 2: Alert system
   - Phase 3: LLM integration

## Installation Steps

### 1. Configure Environment Variables

Copy `.env.example` to `.env` and update the following:

```bash
# Twilio Configuration
TWILIO_ACCOUNT_SID=your_actual_account_sid
TWILIO_AUTH_TOKEN=your_actual_auth_token
TWILIO_PHONE_NUMBER=+15551234567
ALERT_PHONE_NUMBER=+15559876543

# Voice Services (usually no changes needed)
SESAME_TTS_URL=http://sesame-tts:8002
LLM_ENGINE_URL=http://llm-engine:8001
N8N_WEBHOOK_URL=http://n8n:5678
```

### 2. Build and Start Services

```bash
# Build the new services
docker-compose build sesame-tts twilio-bridge

# Start all services
docker-compose up -d

# Check service health
docker-compose ps
```

### 3. Verify Services

Run the test script:

```bash
./scripts/test-voice-integration.sh
```

Expected output:
- ✓ Sesame TTS service is healthy
- ✓ TTS synthesis successful
- ✓ Twilio Bridge service is healthy
- ✓ All connectivity checks pass

### 4. Import N8N Workflow

1. Open N8N: http://localhost:5678
2. Login with your N8N credentials
3. Go to **Workflows** → **Import from File**
4. Select: `n8n/workflows/conversation-voice.json`
5. Activate the workflow

### 5. Configure Twilio Webhooks

If using ngrok or a public URL:

1. Start ngrok (if needed):
   ```bash
   ngrok http 8003
   ```

2. Update your Twilio phone number webhooks:
   - **Voice URL**: `https://your-domain.ngrok.io/api/voice/webhook/answer`
   - **Status Callback URL**: `https://your-domain.ngrok.io/api/voice/webhook/status`

Note: For production, replace ngrok URL with your actual domain.

### 6. Update N8N Webhook URL

If using a public URL for Twilio callbacks, update the environment variable:

```bash
N8N_WEBHOOK_URL=https://your-domain.ngrok.io
```

Then restart the twilio-bridge service:

```bash
docker-compose restart twilio-bridge
```

## Testing Voice Integration

### Test 1: Manual Voice Call

Initiate a test call:

```bash
curl -X POST http://localhost:8003/api/voice/call/initiate \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number": "+15559876543",
    "alert_id": "test-alert-001",
    "alert_summary": "Test alert: Container nginx has stopped on node pve1.",
    "alert_context": {
      "severity": "critical",
      "resource_type": "container",
      "resource_name": "nginx"
    }
  }'
```

You should receive a phone call with the alert details.

### Test 2: Voice Conversation Flow

1. Answer the call
2. Listen to the alert
3. Try saying:
   - "What happened?" - Get more details
   - "Approve" - Approve the recommended action
   - "Dismiss" - Dismiss the alert
   - "Tell me more" - Continue conversation

### Test 3: Integration with Alert Router

Trigger an alert outside business hours:

```bash
# Manually insert a test alert
docker exec proxmox-ai-postgres psql -U proxmox_user -d proxmox_ai -c "
INSERT INTO alerts (alert_type, severity, resource_type, resource_id, resource_name, node_name, title, description, status, created_at)
VALUES ('container_stopped', 'critical', 'container', '100', 'nginx', 'pve1', 'Container Stopped', 'Container nginx on pve1 has stopped unexpectedly', 'active', NOW());
"
```

If outside business hours (Mon-Fri 7am-4pm), the alert router should trigger a voice call.

## Architecture

### Services

1. **Sesame TTS (Port 8002)**
   - Converts text to speech using Sesame CSM 1B model
   - Endpoints:
     - `GET /health` - Health check
     - `POST /api/tts/synthesize` - Generate speech (returns base64 audio)
     - `POST /api/tts/synthesize/stream` - Stream audio directly

2. **Twilio Bridge (Port 8003)**
   - Manages Twilio voice calls and webhooks
   - Integrates with Sesame TTS and LLM Engine
   - Endpoints:
     - `GET /health` - Health check
     - `POST /api/voice/call/initiate` - Initiate outbound call
     - `POST /api/voice/webhook/answer` - Twilio answer webhook
     - `POST /api/voice/webhook/gather` - Twilio speech input handler
     - `POST /api/voice/webhook/status` - Call status updates
     - `GET /api/voice/conversations` - List recent conversations

### Data Flow

```
Alert (Outside Business Hours)
  ↓
Alert Router Workflow (N8N)
  ↓
Voice Initiate Webhook (N8N)
  ↓
Twilio Bridge: Initiate Call
  ↓
Twilio: Outbound Call
  ↓
User Answers
  ↓
Twilio Bridge: Answer Webhook
  ↓
LLM Engine: Generate Response
  ↓
Sesame TTS: Convert to Speech (optional, using Twilio Polly for now)
  ↓
User Speaks Command
  ↓
Twilio: Speech-to-Text
  ↓
Twilio Bridge: Gather Webhook
  ↓
LLM Engine: Process Command
  ↓
Action Approval/Dismissal
  ↓
Database Update & Action Queue
```

### Database Tables

The Twilio Bridge creates a `voice_conversations` table:

```sql
CREATE TABLE voice_conversations (
    id SERIAL PRIMARY KEY,
    call_sid VARCHAR(255) UNIQUE NOT NULL,
    alert_id VARCHAR(255),
    phone_number VARCHAR(50) NOT NULL,
    status VARCHAR(50) DEFAULT 'active',
    context JSONB,
    messages JSONB DEFAULT '[]',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

## Troubleshooting

### Issue: Twilio calls not working

**Check:**
1. Twilio credentials are correct in `.env`
2. Phone numbers are in E.164 format (+15551234567)
3. Twilio account has sufficient funds
4. Twilio phone number has voice capabilities

**Verify:**
```bash
docker logs proxmox-ai-twilio-bridge
```

### Issue: TTS not generating audio

**Check:**
1. Sesame TTS container has sufficient memory (at least 4GB)
2. Model download completed successfully

**Verify:**
```bash
docker logs proxmox-ai-sesame-tts
docker exec proxmox-ai-sesame-tts ls -lh /app/models/sesame-csm-1b/
```

### Issue: Webhook callbacks not received

**Check:**
1. ngrok is running and webhook URL is updated in Twilio console
2. Firewall allows inbound traffic on webhook port
3. N8N_WEBHOOK_URL environment variable is correct

**Verify:**
```bash
curl https://your-domain.ngrok.io/api/voice/webhook/answer
```

### Issue: Voice conversations not saved

**Check:**
1. Database connection is working
2. voice_conversations table exists

**Verify:**
```bash
docker exec proxmox-ai-postgres psql -U proxmox_user -d proxmox_ai -c "SELECT * FROM voice_conversations LIMIT 5;"
```

## Advanced Configuration

### Using Custom TTS Voice

The current implementation uses Twilio's Polly voices for simplicity. To use Sesame TTS instead:

1. Modify `twilio-bridge/app.py` to call Sesame TTS
2. Convert the base64 audio to a Twilio-compatible format
3. Use `<Play>` TwiML instead of `<Say>`

### Multiple Phone Numbers

To support multiple alert recipients:

1. Add alert routing logic in the alert-router workflow
2. Query contacts from database based on alert severity
3. Call multiple numbers in parallel or sequence

### Voice Commands

Current supported commands:
- "Approve" - Approve recommended action
- "Dismiss" / "Ignore" - Dismiss alert
- Conversational questions - Handled by LLM

To add more commands, modify `twilio-bridge/app.py` webhook handlers.

## Performance Considerations

1. **Sesame TTS**: 
   - Model loading takes 30-60 seconds on first start
   - Each synthesis request takes 2-5 seconds
   - Requires 4-8GB RAM depending on model size

2. **Twilio Bridge**:
   - Lightweight, minimal resources
   - Rate limited by Twilio API (check your account limits)

3. **Concurrent Calls**:
   - Default: 1 concurrent call per Twilio phone number
   - Upgrade Twilio plan for more concurrent calls

## Security Notes

1. **Webhook Security**:
   - Current implementation accepts all webhook requests
   - For production, add Twilio signature validation
   - Use HTTPS with valid SSL certificates

2. **Credentials**:
   - Never commit `.env` to version control
   - Rotate Twilio Auth Token periodically
   - Use environment-specific credentials

3. **PII and Recording**:
   - Voice conversations stored in database
   - Review data retention policies
   - Consider disabling Twilio call recording

## Next Steps

After completing Phase 4:

- **Phase 5**: Advanced monitoring (log analysis, Docker monitoring, network checks)
- **Phase 6**: Action execution & safeguards
- **Phase 7**: Learning & memory system
- **Phase 8**: Testing, hardening & documentation

## Support

For issues specific to:
- **Twilio**: Check [Twilio Documentation](https://www.twilio.com/docs)
- **Sesame AI**: Check [HuggingFace Model Page](https://huggingface.co/sesame-ai/csm-1b)
- **N8N**: Check [N8N Documentation](https://docs.n8n.io)

## References

- [Twilio Voice API](https://www.twilio.com/docs/voice)
- [TwiML Reference](https://www.twilio.com/docs/voice/twiml)
- [Sesame CSM Model](https://huggingface.co/sesame-ai/csm-1b)
- [N8N Webhook Triggers](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)
