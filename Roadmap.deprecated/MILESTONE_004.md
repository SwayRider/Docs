# MILESTONE_004 — Auth Security Hardening

## Overview

Implement critical and high-impact security improvements to the authentication service before opening the application to public testers.

## Scope

- **Phase**: MVP (Pre-Public)
- **Priority**: Critical
- **Dependencies**: MILESTONE_003
- **Blocks**: None (but required before public launch)

## Background

The AUTH_IMPROVEMENTS analysis identified several security gaps. This milestone addresses the most critical and high-impact items before public testing begins.

## Requirements

| ID | Name | Components | Status |
|----|------|------------|--------|
| [REQUIREMENT_001](./MILESTONE_004/REQUIREMENT_001.md) | Critical Security (Phase 1) | AuthService, swlib | Planned |
| [REQUIREMENT_002](./MILESTONE_004/REQUIREMENT_002.md) | High Impact Security (Phase 2) | AuthService, swlib | Planned |

## Affected Components

### Backend Services
- **AuthService**: Rate limiting, account lockout, audit logging
- **swlib**: Encryption, cookie security, TLS

### Infrastructure
- **Redis**: New dependency for rate limiting
- **Database**: New tables for login attempts, audit log, password history

## Success Criteria

1. Rate limiting prevents brute force attacks
2. Accounts lock after failed attempts
3. All auth events logged for audit
4. JWT private keys encrypted at rest
5. Cookies secured with SameSite=Strict
6. TLS enforced for database connections
7. Password breach detection active
8. Password history prevents reuse

## Timeline Estimate

| Requirement | Estimated Effort |
|-------------|------------------|
| REQUIREMENT_001 | 2 weeks |
| REQUIREMENT_002 | 2 weeks |
| **Total** | **4 weeks** |

## Environment Variables

```bash
# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Rate Limiting
RATE_LIMIT_IP_MAX=10
RATE_LIMIT_EMAIL_MAX=5

# Account Lockout
LOCKOUT_MAX_ATTEMPTS=5
LOCKOUT_PROGRESSIVE=true

# Audit
AUDIT_RETENTION_DAYS=90

# Encryption
ENCRYPTION_MASTER_KEY=<base64-256-bit>

# Cookie Security
COOKIE_SAMESITE=strict
COOKIE_SECURE=auto

# TLS
DB_SSL_MODE=require

# HIBP
HIBP_ENABLED=true

# Password History
PASSWORD_HISTORY_SIZE=5
```
