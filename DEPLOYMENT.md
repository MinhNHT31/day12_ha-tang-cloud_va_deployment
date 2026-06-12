# Deployment Information

## Public URL
https://day12-production-0d22.up.railway.app

## Platform
Railway

## Test Commands

### 1. Health Check (Liveness Probe)
```bash
curl -i https://day12-agent-production.up.railway.app/health
```
**Expected response:**
```json
{
  "status": "ok",
  "version": "1.0.0",
  "environment": "production",
  "uptime_seconds": 124.5,
  "total_requests": 0,
  "checks": {
    "llm": "mock"
  },
  "timestamp": "2026-06-12T07:55:00Z"
}
```

### 2. Readiness Check (Readiness Probe)
```bash
curl -i https://day12-agent-production.up.railway.app/ready
```
**Expected response:**
```json
{
  "ready": true
}
```

### 3. API Test (With Authentication and user_id session tracking)
- **Without API Key (Should fail with 401):**
```bash
curl -i -X POST https://day12-agent-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test_user", "question": "Hello"}'
```
- **With Correct API Key (Should succeed):**
```bash
curl -i -X POST https://day12-agent-production.up.railway.app/ask \
  -H "X-API-Key: your-secret-api-key" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test_user", "question": "My name is Alice"}'
```
- **Conversation history reference test:**
```bash
curl -i -X POST https://day12-agent-production.up.railway.app/ask \
  -H "X-API-Key: your-secret-api-key" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test_user", "question": "What is my name?"}'
```
**Expected response:**
```json
{
  "question": "What is my name?",
  "answer": "Your name is Alice.",
  "model": "gpt-4o-mini",
  "timestamp": "2026-06-12T07:56:00Z"
}
```

## Environment Variables Set
- `PORT` (assigned dynamically by platform, default: `8000`)
- `ENVIRONMENT` (set to `production`)
- `AGENT_API_KEY` (custom production api key, e.g. `your-secret-api-key`)
- `REDIS_URL` (Redis database connection string, e.g., `redis://default:password@host:port`)
- `DAILY_BUDGET_USD` (set to `5.0`)
- `RATE_LIMIT_PER_MINUTE` (set to `20`)

## Screenshots
- **Deployment dashboard**: Saved at `screenshots/dashboard.png`
- **Service running**: Saved at `screenshots/running.png`
- **Test results**: Saved at `screenshots/test.png`
