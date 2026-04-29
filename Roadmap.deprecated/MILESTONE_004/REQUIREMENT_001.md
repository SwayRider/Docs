# REQUIREMENT_001 — Critical Security (Phase 1)

## Overview

Implement the most critical security improvements: rate limiting, account lockout, audit logging, and JWT private key encryption.

## Context

- **Components**: AuthService, swlib
- **Priority**: Critical
- **Status**: Planned
- **Reference**: AUTH_IMPROVEMENTS_PHASE_01.md

## Requirements

### 1. Rate Limiting (Redis)
- Login attempt throttling: 10 attempts per 60s per IP
- Email-based rate limiting: 5 attempts per 300s
- Sliding window counter pattern in Redis
- Key format: `ratelimit:ip:{ip}` and `ratelimit:email:{email}`
- Returns `codes.ResourceExhausted` when limited
- Resets counters on successful login

### 2. Account Lockout
- Progressive lockout after failed attempts:
  - 1-5 attempts: No lockout, just track
  - 6: Lock 15 min
  - 7: Lock 30 min
  - 8: Lock 1 hour
  - 9: Lock 4 hours
  - 10+: Lock 24 hours (max)
- Admin unlock capability
- Email notification on lockout

### 3. Audit Logging
- Log all authentication events:
  - Login success/failure
  - Logout
  - Token refresh
  - Registration
  - Email verification
  - Password change/reset
  - Account lock/unlock
  - Service client auth
  - Admin actions
- 90-day retention with daily cleanup

### 4. JWT Private Key Encryption
- AES-256-GCM encryption
- Master key from `ENCRYPTION_MASTER_KEY` env var
- Encrypt before storing, decrypt in-memory only
- Supports gradual migration

## Acceptance Criteria

1. Rate limiting active on login endpoint
2. Accounts lock after progressive failed attempts
3. All auth events logged to audit table
4. JWT private keys encrypted in database
5. Existing functionality unaffected
6. Tests pass for all new features

## Affected Files

### Backend
- `backend/swlib/ratelimit/` — New package
- `backend/swlib/encryption/` — New package
- `backend/services/authservice/internal/db/` — Login attempts, lockout
- `backend/services/authservice/internal/audit/` — Audit logging
- `backend/services/authservice/migrations/` — New tables

### Infrastructure
- `infra/dev/layer-00/compose.yaml` — Redis container
