# Deployment Information

## Public URL
https://<your-agent-url>

## Platform
Railway / Render

## Test Commands

### 1. Health Check (200)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' "https://<your-agent-url>/health"
```

### 2. Readiness Check (200)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' "https://<your-agent-url>/ready"
```

### 3. Auth Check — Missing API Key (401)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "https://<your-agent-url>/ask" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

### 4. API Test — With API Key (200)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "https://<your-agent-url>/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -d '{"question": "Hello"}'
```

## Environment Variables Set
- `PORT` (nền tảng tự cấp hoặc đặt `8000`)
- `REDIS_URL` (trỏ tới Redis service trên cloud)
- `AGENT_API_KEY` (đặt trên dashboard nền tảng, không ghi trong code)
- `ENVIRONMENT=production`
- `RATE_LIMIT_PER_MINUTE=10`
- `DAILY_BUDGET_USD=1.0`

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
