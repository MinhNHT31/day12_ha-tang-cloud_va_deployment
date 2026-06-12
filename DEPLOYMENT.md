# Deployment Information

## Public URL
https://day12ha-tang-cloudvadeployment-production-c103.up.railway.app

## Platform
Railway

## Test Commands

### 1. Health Check (Liveness Probe)
```bash
curl -i https://day12ha-tang-cloudvadeployment-production-c103.up.railway.app/health
```
**Expected response:**
```json
{
  "status": "ok",
  "version": "1.0.0",
  "environment": "production",
  "uptime_seconds": 423.1,
  "total_requests": 4,
  "checks": {
    "llm": "mock"
  },
  "timestamp": "2026-06-12T11:05:58.690708+00:00"
}
```

### 2. Readiness Check (Readiness Probe)
```bash
curl -i https://day12ha-tang-cloudvadeployment-production-c103.up.railway.app/ready
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
curl -i -X POST https://day12ha-tang-cloudvadeployment-production-c103.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test_user", "question": "Hello"}'
```
- **With Correct API Key (Should succeed):**
```bash
curl -i -X POST https://day12ha-tang-cloudvadeployment-production-c103.up.railway.app/ask \
  -H "X-API-Key: test-key" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```
- **Repeat API test:**
```bash
curl -i -X POST https://day12ha-tang-cloudvadeployment-production-c103.up.railway.app/ask \
  -H "X-API-Key: test-key" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```
**Expected response:**
```json
{
  "question": "Hello",
  "answer": "Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.",
  "model": "gpt-4o-mini",
  "timestamp": "2026-06-12T11:06:05.101263+00:00"
}
```

## Environment Variables Set
- `PORT` (assigned dynamically by Railway, observed: `8080`)
- `ENVIRONMENT` (set to `production`)
- `AGENT_API_KEY` (custom production API key)
- `REDIS_URL` (Railway Redis connection string)
- `DAILY_BUDGET_USD` (set to `5.0`)
- `RATE_LIMIT_PER_MINUTE` (set to `20`)

## Screenshots
- **Service running**: Saved at `screenshots/running.png`
- **API test results**: Saved at `screenshots/test.png`
- **Combined terminal proof**: Saved at `screenshots/test-results.png`
