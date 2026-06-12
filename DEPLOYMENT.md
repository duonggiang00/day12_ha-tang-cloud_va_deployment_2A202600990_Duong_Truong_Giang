# Deployment Information

## Public URL
https://upbeat-hope-production-27a0.up.railway.app

## Platform
Railway

## Test Commands

### Health Check
```bash
curl https://upbeat-hope-production-27a0.up.railway.app/health
```

### API Test (with authentication)
```bash
curl -X POST https://upbeat-hope-production-27a0.up.railway.app/ask \
  -H "X-API-Key: my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```

## Environment Variables Set
- PORT
- AGENT_API_KEY

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
