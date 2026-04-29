# REQUIREMENT_002 — High Impact Security (Phase 2)

## Overview

Implement high-impact security improvements: cookie security hardening, TLS enforcement, password breach detection, and password history.

## Context

- **Components**: AuthService, swlib
- **Priority**: High
- **Status**: Planned
- **Reference**: AUTH_IMPROVEMENTS_PHASE_02.md

## Requirements

### 1. Cookie Security Hardening
- SameSite=Strict by default (configurable)
- Secure flag auto-detection from request context
- Optional AES-256 encryption for cookie values
- Environment variable configuration

### 2. TLS Enforcement
- Database: `sslmode=require` in production
- gRPC: Optional TLS via configuration
- Environment variables for TLS settings
- Graceful fallback for development

### 3. Password Breach Detection (HaveIBeenPwned)
- k-anonymity model: only first 5 chars of SHA-1 hash sent
- Check during registration and password change
- Fail open if API unavailable (log warning)
- Configuration for enable/disable

### 4. Password History
- Store last 5 password hashes per user
- Check new password against history
- Prevent password reuse
- Daily cleanup of excess entries

## Acceptance Criteria

1. Cookies use SameSite=Strict in production
2. Secure flag set correctly based on context
3. Database connections use TLS in production
4. Breached passwords rejected on registration
5. Breached passwords rejected on password change
6. Password reuse prevented
7. All features configurable via environment variables

## Affected Files

### Backend
- `backend/swlib/http/cookies/cookie.go` — Cookie security
- `backend/swlib/hibp/` — New package
- `backend/services/authservice/internal/db/password_history.go` — New
- `backend/swlib/app/grpc.go` — TLS configuration

### Migrations
- `backend/services/authservice/migrations/0001_011_password_history.sql`
