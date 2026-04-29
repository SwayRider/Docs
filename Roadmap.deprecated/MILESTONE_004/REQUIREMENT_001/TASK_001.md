# TASK_001 — Rate Limiting

## Overview

Implement Redis-based rate limiting for authentication endpoints.

## Context

- **Requirement**: REQUIREMENT_001 (MILESTONE_004)
- **Component**: AuthService, swlib
- **Estimated Effort**: 2 days

## Implementation

### Redis Integration
Add Redis dependency to swlib:
- Package: `backend/swlib/ratelimit/`
- Redis client: `go-redis/redis/v9`
- Connection pool configuration

### Rate Limiting Logic
```
Key format:
- ratelimit:ip:{ip} — IP-based limiting
- ratelimit:email:{email} — Email-based limiting

Window: Sliding window counter
- IP: 10 attempts per 60 seconds
- Email: 5 attempts per 300 seconds
```

### Integration Points
- Login endpoint: Both IP and email limiting
- Registration endpoint: Email limiting
- Password reset: Email limiting

### Error Response
- gRPC: `codes.ResourceExhausted`
- HTTP: `429 Too Many Requests`
- Header: `Retry-After: {seconds}`

## Acceptance Criteria

- [ ] Redis connection established
- [ ] Rate limiting active on login
- [ ] Both IP and email limits enforced
- [ ] Proper error responses returned
- [ ] Counters reset on success
- [ ] Configuration via environment variables

## Files to Create/Modify

### New Files
- `backend/swlib/ratelimit/ratelimit.go`
- `backend/swlib/ratelimit/redis.go`
- `backend/swlib/ratelimit/ratelimit_test.go`

### Modified Files
- `backend/services/authservice/internal/server/authentication.go`
- `infra/dev/layer-00/compose.yaml` — Add Redis container

## Environment Variables

```bash
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
RATE_LIMIT_IP_MAX=10
RATE_LIMIT_IP_WINDOW=60
RATE_LIMIT_EMAIL_MAX=5
RATE_LIMIT_EMAIL_WINDOW=300
```
