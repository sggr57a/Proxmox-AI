# Learning System Documentation

How the Proxmox AI system learns from incidents and improves over time.

## Overview

The Learning System is a core feature that enables the monitoring system to:
1. **Remember** past incidents and their resolutions
2. **Suggest** proven solutions for similar new incidents
3. **Detect patterns** in failures and system behavior
4. **Improve accuracy** by learning from user feedback
5. **Reduce alert fatigue** by identifying false positives

## Architecture

```
┌──────────────────────────────────────────────────────┐
│              Learning Engine (Port 8004)              │
├──────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌──────────────────────────┐   │
│  │ Incident       │  │  Pattern Detector        │   │
│  │ Analyzer       │  │  - Temporal patterns     │   │
│  │                │  │  - Recurring errors      │   │
│  └────────────────┘  │  - Cascade failures      │   │
│                      │  - False positive        │   │
│  ┌────────────────┐  └──────────────────────────┘   │
│  │ Similarity     │                                  │
│  │ Engine (RAG)   │  ┌──────────────────────────┐   │
│  │ - Vector DB    │  │  Feedback Processor      │   │
│  │ - Embeddings   │  │  - User ratings          │   │
│  └────────────────┘  │  - Auto feedback         │   │
│                      │  - Score adjustment      │   │
└──────────────────────┴──────────────────────────────┘
           │                        │
           ▼                        ▼
┌─────────────────────┐   ┌──────────────────────┐
│  PostgreSQL         │   │  ChromaDB            │
│  - Incidents        │   │  - Vector embeddings │
│  - Feedback         │   │  - Similarity index  │
│  - Patterns         │   └──────────────────────┘
└─────────────────────┘
```

## Core Components

### 1. Incident Recording

Every time an alert is resolved, the system records:

```json
{
  "incident_id": "uuid",
  "alert_id": "alert-123",
  "entity_type": "container",
  "entity_name": "nginx-proxy",
  "symptoms": "Container restarting every 30 seconds. Port 80 bind failure.",
  "root_cause": "Port conflict - another service bound to port 80",
  "actions_taken": ["identify_conflicting_process", "stop_conflicting_container"],
  "outcome": "resolved",
  "resolution_time": 180,
  "effectiveness_score": 5,
  "timestamp": "2024-01-29T10:00:00Z"
}
```

**API Endpoint:**
```bash
curl -X POST http://localhost:8004/incidents \
  -H "Content-Type: application/json" \
  -d @incident.json
```

### 2. Vector Embeddings & Similarity Search (RAG)

Each incident is converted to a high-dimensional vector embedding that captures semantic meaning.

**How it works:**
1. New alert arrives: "Redis container keeps crashing"
2. System generates embedding for the alert
3. Searches incident database for similar past incidents
4. Returns top 3-5 matches with similarity scores (0.0-1.0)
5. LLM uses these matches as context for suggesting actions

**Example Query:**
```bash
curl -X POST http://localhost:8004/incidents/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "container restart loop port conflict",
    "limit": 5
  }'
```

**Example Response:**
```json
{
  "incidents": [
    {
      "incident_id": "uuid-1",
      "alert_id": "seed-001",
      "symptoms": "Container restarting every 30 seconds. Port 80 bind failure.",
      "root_cause": "Port conflict",
      "actions_taken": ["stop_conflicting_container"],
      "outcome": "resolved",
      "effectiveness_score": 5,
      "similarity": 0.92
    },
    {
      "incident_id": "uuid-2",
      "alert_id": "seed-003",
      "symptoms": "PostgreSQL restart loop. Port 5432 conflict.",
      "root_cause": "Port already in use",
      "actions_taken": ["stop_conflicting_container"],
      "outcome": "resolved",
      "effectiveness_score": 5,
      "similarity": 0.87
    }
  ]
}
```

**Similarity Interpretation:**
- **>0.90**: Nearly identical issue
- **0.80-0.90**: Very similar, likely same root cause
- **0.70-0.80**: Somewhat similar, may share characteristics
- **<0.70**: Potentially unrelated

### 3. Pattern Detection

The system automatically identifies recurring patterns across incidents.

#### Temporal Patterns

**Definition:** Issues that occur at predictable times.

**Example:**
```json
{
  "pattern_type": "temporal",
  "description": "Database backup causes high load every Sunday 2am-3am",
  "occurrences": 12,
  "time_pattern": {
    "day_of_week": 0,
    "hour_range": [2, 3]
  },
  "recommendation": "Suppress high_load alerts during backup window or schedule backup during lower usage period"
}
```

**Detection Logic:**
- Analyzes incident timestamps
- Identifies recurring time patterns (hourly, daily, weekly)
- Groups incidents with >70% time correlation

#### Recurring Errors

**Definition:** Same error occurring multiple times.

**Example:**
```json
{
  "pattern_type": "recurring",
  "description": "Redis connection timeout - occurred 15 times in 7 days",
  "error_signature": "ETIMEDOUT: redis://localhost:6379",
  "occurrences": 15,
  "time_span_days": 7,
  "common_root_cause": "Redis max_clients limit reached",
  "recommendation": "Increase max_clients or implement connection pooling"
}
```

**Detection Logic:**
- Groups incidents by symptom similarity (>0.85)
- Counts occurrences within time window
- Flags patterns with >5 occurrences

#### Cascade Failures

**Definition:** One failure triggering multiple downstream failures.

**Example:**
```json
{
  "pattern_type": "cascade",
  "description": "Auth service failure causes API gateway and frontend failures",
  "primary_incident": "seed-014",
  "secondary_incidents": ["seed-015", "seed-016"],
  "time_correlation": "Within 5 minutes",
  "dependency_chain": ["auth-service", "api-gateway", "frontend-app"],
  "recommendation": "Implement circuit breaker or health check before alerting on downstream services"
}
```

**Detection Logic:**
- Analyzes incident timing
- Groups incidents within 5-minute window
- Identifies dependency relationships
- Marks secondary alerts as "cascade" to reduce noise

#### False Positive Patterns

**Definition:** Alerts that consistently don't indicate real problems.

**Example:**
```json
{
  "pattern_type": "false_positive",
  "description": "High CPU alert during nightly batch job (expected behavior)",
  "alert_type": "high_cpu",
  "entity_name": "batch-processor",
  "occurrences": 30,
  "user_feedback_average": 2.1,
  "recommendation": "Suppress high_cpu alerts for batch-processor between 2am-4am"
}
```

**Detection Logic:**
- Analyzes user feedback scores (<3.0 indicates false positive)
- Checks for "false_positive" outcome
- Flags alerts with >10 occurrences and low effectiveness

**API to detect patterns:**
```bash
curl http://localhost:8004/patterns/detect
```

### 4. Feedback Loop

After each action, the system collects feedback to improve future recommendations.

#### Explicit Feedback (User-Provided)

**Via Slack:**
After resolving an incident, user receives:
```
✅ Incident resolved!

How would you rate the suggested solution?
⭐⭐⭐⭐⭐ Excellent
⭐⭐⭐⭐ Good
⭐⭐⭐ Okay
⭐⭐ Poor
⭐ Didn't help
```

**Via Voice:**
System asks: "Did this solution resolve the issue? Please say yes or no."

**API for feedback:**
```bash
curl -X POST http://localhost:8004/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "incident_id": "uuid",
    "user": "admin",
    "rating": 5,
    "comment": "Perfect solution, resolved immediately"
  }'
```

#### Automatic Feedback

System monitors outcomes without user input:

**Success indicators:**
- Entity stays healthy for 24+ hours after action
- Same alert doesn't reoccur within 1 hour
- Action completes without errors

**Failure indicators:**
- Same alert reoccurs within 1 hour
- Entity still unhealthy after action
- Action execution failed

**Scoring:**
```python
if same_alert_within_1_hour:
    effectiveness_score = 2  # Failed
elif entity_healthy_24_hours:
    effectiveness_score = 5  # Success
else:
    effectiveness_score = 3  # Uncertain
```

### 5. Continuous Improvement

The system adjusts behavior based on accumulated learnings.

#### Confidence Scoring

Each suggested action has a confidence score (0.0-1.0):

```
Confidence = (successful_attempts / total_attempts) * similarity_score
```

**Example:**
- Action "restart_container" for "port conflict" issue
- 15 successful resolutions out of 17 attempts
- Current incident similarity: 0.88
- Confidence: (15/17) * 0.88 = 0.78

**Thresholds:**
- **>0.80**: Auto-execute (if autonomous action)
- **0.60-0.80**: Suggest with high priority
- **0.40-0.60**: Suggest with medium priority
- **<0.40**: Suggest as alternative option

#### Alert Threshold Adjustment

Based on false positive patterns:

```yaml
# Before learning:
high_cpu:
  threshold: 80%
  
# After detecting false positives during backup:
high_cpu:
  threshold: 80%
  suppression_rules:
    - entity: batch-processor
      time_range: [2, 4]  # 2am-4am
      reason: "Expected during nightly batch job"
```

#### Action Prioritization

Suggested actions are ranked by:
1. **Similarity to past successes** (RAG search)
2. **Effectiveness score** (from feedback)
3. **Recency** (recent incidents weighted higher)
4. **User preference** (learns which actions user approves)

## Learning Timeline & Expectations

### Week 1: Initial Training

**Data needed:** 20-50 incidents (seed with `./scripts/seed-learning-data.sh`)

**Capabilities:**
- Basic similarity search works
- Pattern detection begins (limited patterns)
- Suggestions available but conservative

**Metrics:**
- Alert accuracy: ~65%
- False positive rate: ~35%
- Action success rate: ~70%

### Week 4: Early Learning

**Data accumulated:** 100-200 incidents

**Capabilities:**
- RAG system provides relevant suggestions (>0.8 similarity)
- Temporal patterns identified (daily/weekly cycles)
- Recurring errors detected and flagged
- False positives being suppressed

**Metrics:**
- Alert accuracy: ~80%
- False positive rate: ~20%
- Action success rate: ~85%
- Avg resolution time: -40% improvement

### Month 3: Mature Learning

**Data accumulated:** 500+ incidents

**Capabilities:**
- High-confidence autonomous actions
- Cascade failure prediction
- Proactive alerting before outages
- Context-aware suggestions
- User-specific preference learning

**Metrics:**
- Alert accuracy: ~90%
- False positive rate: ~10%
- Action success rate: ~92%
- Avg resolution time: -60% improvement

### Month 6+: Advanced Learning

**Data accumulated:** 1000+ incidents

**Capabilities:**
- Anomaly prediction (before failure occurs)
- Complex pattern recognition
- Multi-step action plans
- Infrastructure optimization suggestions
- Seasonal pattern awareness

**Metrics:**
- Alert accuracy: ~95%
- False positive rate: ~5%
- Action success rate: ~94%
- Avg resolution time: -70% improvement
- Proactive prevention: 20-30% of potential incidents

## Viewing Learning Progress

### Metrics Dashboard

```bash
curl http://localhost:8004/metrics
```

**Response:**
```json
{
  "total_incidents": 156,
  "incidents_last_7_days": 23,
  "incidents_last_30_days": 98,
  "patterns_detected": 12,
  "avg_resolution_time_minutes": 8.5,
  "alert_accuracy": 0.87,
  "action_success_rate": 0.89,
  "false_positive_rate": 0.13,
  "improvement_vs_baseline": {
    "resolution_time": "-52%",
    "accuracy": "+22%",
    "success_rate": "+19%"
  }
}
```

### Incident History

```bash
# Recent incidents
curl http://localhost:8004/incidents/recent?limit=20

# Successful resolutions only
curl http://localhost:8004/incidents?outcome=resolved&effectiveness_score=5

# Failed resolutions
curl http://localhost:8004/incidents?outcome=unresolved
```

### Pattern Summary

```bash
curl http://localhost:8004/patterns/summary
```

**Response:**
```json
{
  "total_patterns": 12,
  "by_type": {
    "temporal": 3,
    "recurring": 5,
    "cascade": 2,
    "false_positive": 2
  },
  "recent_patterns": [
    {
      "type": "recurring",
      "description": "Nginx config reload fails due to syntax error",
      "occurrences": 8,
      "recommendation": "Validate config before reload"
    }
  ]
}
```

## Maintaining the Learning System

### Data Retention

**Default retention (defined in database/init.sql):**
- Incidents: 12 months
- Feedback: 12 months
- Patterns: Indefinite (manually reviewed)
- Embeddings: Regenerated on demand

**Cleanup:**
```bash
# Clean old data (automatically runs daily at 2am via cron)
docker-compose exec postgres psql -U proxmox_user -d proxmox_ai -c \
  "DELETE FROM incidents WHERE created_at < NOW() - INTERVAL '12 months';"
```

### Retraining

**Embeddings regeneration** (after major data changes):
```bash
curl -X POST http://localhost:8004/embeddings/regenerate
```

**Pattern detection** (runs automatically daily):
```bash
# Manual trigger
curl -X POST http://localhost:8004/patterns/detect
```

### Exporting Knowledge

**Export incident database** (for backup or analysis):
```bash
curl http://localhost:8004/incidents/export > incidents-export.json
```

**Import to another system:**
```bash
curl -X POST http://localhost:8004/incidents/import \
  -H "Content-Type: application/json" \
  -d @incidents-export.json
```

## Privacy & Compliance

### Data Anonymization

After 12 months, personal identifiers are removed:

```sql
UPDATE incidents 
SET user_name = 'anonymized',
    user_id = 'anonymized'
WHERE created_at < NOW() - INTERVAL '12 months';
```

### Sensitive Data Filtering

Incident recordings automatically filter:
- Passwords and API keys
- Personal identifiable information (PII)
- Internal IP addresses (replaced with placeholders)

### GDPR Compliance

**Right to be forgotten:**
```bash
curl -X DELETE http://localhost:8004/user-data/<user-id>
```

**Data export request:**
```bash
curl http://localhost:8004/user-data/<user-id>/export
```

## Best Practices

### 1. Seed Initial Data

Don't wait for real incidents - seed with synthetic data:
```bash
./scripts/seed-learning-data.sh
```

### 2. Provide Explicit Feedback

Rate solutions when prompted - improves learning speed by 2-3x.

### 3. Review Patterns Monthly

Check detected patterns and adjust alert rules:
```bash
curl http://localhost:8004/patterns/summary
```

### 4. Validate Suggestions

For first 2-4 weeks, manually review suggested actions before auto-approval.

### 5. Maintain Data Quality

Remove incorrect incident records:
```bash
curl -X DELETE http://localhost:8004/incidents/<incident-id>
```

### 6. Monitor Metrics

Track improvement weekly:
```bash
curl http://localhost:8004/metrics | jq '.improvement_vs_baseline'
```

## Troubleshooting Learning Issues

See [TROUBLESHOOTING.md - Learning System Issues](TROUBLESHOOTING.md#learning-system-issues) for common problems and solutions.

## Further Reading

- [RAG (Retrieval-Augmented Generation)](https://arxiv.org/abs/2005.11401)
- [Vector Embeddings](https://www.pinecone.io/learn/vector-embeddings/)
- [ChromaDB Documentation](https://docs.trychroma.com/)
- [Sentence Transformers](https://www.sbert.net/)
