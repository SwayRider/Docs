# Phase 4: Operational Security Improvements

## Overview

All of Phase 4 is still ⏳ **PENDING** — no code found anywhere in the repo for any of these items.

- **Monitoring and alerting** — Prometheus metrics for auth health
- **Key rotation improvements** — audit logging and usage tracking (the rotation mechanism itself already works automatically — see note below — only the audit/tracking layer is missing)
- **Service client hardening** — auto-rotation with dual-secret grace period

---

## 1. Prometheus Metrics — ⏳ PENDING

### Configuration (target)
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Metrics Enabled | `METRICS_ENABLED` | true |
| Metrics Path | `METRICS_PATH` | /metrics |

### Metrics to Expose

| Metric | Type | Description |
|--------|------|-------------|
| `auth_login_total` | Counter | Login attempts (labels: success, mfa_required) |
| `auth_login_duration_seconds` | Histogram | Login request duration |
| `auth_register_total` | Counter | Registration attempts |
| `auth_refresh_total` | Counter | Token refresh attempts |
| `auth_rate_limit_hits_total` | Counter | Rate limit hits (labels: type) |
| `auth_account_lockouts_total` | Counter | Account lockouts |
| `auth_mfa_verifications_total` | Counter | MFA verifications (labels: success) |
| `auth_breached_passwords_total` | Counter | Breached password rejections |
| `auth_jwt_sign_total` | Counter | JWT signing operations |
| `auth_jwt_verify_total` | Counter | JWT verification attempts |
| `auth_jwt_key_rotations_total` | Counter | JWT key rotations |
| `auth_sessions_active` | Gauge | Active sessions |
| `auth_service_client_auth_total` | Counter | Service client authentications |
| `auth_service_client_rotations_total` | Counter | Service client secret rotations |

### New Package: `swlib/metrics/`

```go
type AuthMetrics struct {
    LoginTotal    *prometheus.CounterVec
    LoginDuration *prometheus.HistogramVec
    RegisterTotal prometheus.Counter
    // ... other metrics ...
}
```

---

## 2. Alerting Rules — ⏳ PENDING

### Prometheus Alertmanager Rules

```yaml
groups:
  - name: authservice
    rules:
      - alert: HighFailedLoginRate
        expr: rate(auth_login_total{success="false"}[5m]) > 10
        for: 5m

      - alert: AccountLockouts
        expr: increase(auth_account_lockouts_total[1h]) > 5

      - alert: HighMFAFailureRate
        expr: rate(auth_mfa_verifications_total{success="false"}[5m]) > 5

      - alert: JWTKeyRotationFailed
        expr: increase(auth_jwt_key_rotations_total[48h]) == 0

      - alert: AuthServiceDown
        expr: up{job="authservice"} == 0
```

---

## 3. Key Rotation Improvements — ⏳ PENDING (audit/tracking only)

### Current State

JWT keys already rotate automatically 3 days before expiration, guarded by Postgres advisory locks (`authservice/internal/db/jwt_keys.go`) — this part is done and working, not part of this phase's scope. What's missing:
- No audit trail of rotation events
- No usage tracking

### Improvements

#### Audit Logging
- Log rotation events to the audit log introduced in Phase 1
- Include new key validity period and rotation reason

#### Key Usage Tracking
```sql
ALTER TABLE jwt_keys ADD COLUMN usage_count INTEGER DEFAULT 0;
ALTER TABLE jwt_keys ADD COLUMN last_used_at TIMESTAMPTZ;
```

#### Expiration Cleanup
- Background routine to delete keys expired >7 days (allows for clock skew and pending tokens)

---

## 4. Service Client Hardening — Auto-Rotation with Dual-Secret — ⏳ PENDING

### Configuration (target)
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Secret Expiration | `SERVICE_CLIENT_SECRET_EXPIRY_DAYS` | 90 |
| Expiration Warning | `SERVICE_CLIENT_EXPIRY_WARNING_DAYS` | 14 |
| Rotation Grace Period | `SERVICE_CLIENT_ROTATION_GRACE_DAYS` | 7 |

`authservice/internal/db/service_clients.go` currently has none of `expires_at`, `client_secret_old`, or `usage_count` — this is greenfield work.

### Database Migration

```sql
-- +migrate Up
ALTER TABLE service_clients ADD COLUMN client_secret_old TEXT;
ALTER TABLE service_clients ADD COLUMN old_secret_expires_at TIMESTAMPTZ;
ALTER TABLE service_clients ADD COLUMN secret_rotated_at TIMESTAMPTZ;
ALTER TABLE service_clients ADD COLUMN expires_at TIMESTAMPTZ;
ALTER TABLE service_clients ADD COLUMN last_used_at TIMESTAMPTZ;
ALTER TABLE service_clients ADD COLUMN usage_count INTEGER DEFAULT 0;

-- +migrate Down
ALTER TABLE service_clients DROP COLUMN client_secret_old;
ALTER TABLE service_clients DROP COLUMN old_secret_expires_at;
ALTER TABLE service_clients DROP COLUMN secret_rotated_at;
ALTER TABLE service_clients DROP COLUMN expires_at;
ALTER TABLE service_clients DROP COLUMN last_used_at;
ALTER TABLE service_clients DROP COLUMN usage_count;
```
Migration number must be chosen against the actual current sequence in `authservice/migrations/` at implementation time.

### Auto-Rotation Flow

```
Day 0:     Secret created, expires_at = now + 90 days
Day 76:    Warning: Secret expires in 14 days
Day 83:    Auto-rotation triggered:
           - Generate new secret
           - Old secret → client_secret_old (valid 7 more days)
           - New secret → client_secret (primary)
           - old_secret_expires_at = now + 7 days
           - Audit log: "Secret auto-rotated"
Day 83-90: Both secrets work (grace period)
Day 90:    Old secret deleted, only new secret works
           - Client must have updated by now
```

### New gRPC Endpoints

```protobuf
rpc GetServiceClientSecret(GetServiceClientSecretRequest) returns (GetServiceClientSecretResponse) {
  option (google.api.http) = {
    get: "/api/v1/auth/service-clients/{client_id}/secret"
  };
}

rpc GetServiceClientStatus(GetServiceClientStatusRequest) returns (GetServiceClientStatusResponse) {
  option (google.api.http) = {
    get: "/api/v1/auth/service-clients/{client_id}/status"
  };
}

rpc RotateServiceClientSecret(RotateServiceClientSecretRequest) returns (RotateServiceClientSecretResponse) {
  option (google.api.http) = {
    post: "/api/v1/auth/service-clients/rotate-secret"
    body: "*"
  };
}
```

### Modified Model

```go
type ServiceClientInternal struct {
    ClientID           string
    ClientSecretHash   string
    ClientSecretOld    sql.NullString  // Previous secret during grace period
    OldSecretExpiresAt sql.NullTime    // When old secret stops working
    SecretRotatedAt    sql.NullTime    // When rotation happened
    ExpiresAt          sql.NullTime    // When current secret expires
    Name               string
    Description        string
    Scopes             []string
    LastUsedAt         sql.NullTime
    UsageCount         int
}
```

### Authentication Logic (dual-secret check)

```go
func (s *AuthServer) GetToken(ctx context.Context, req *authv1.GetTokenRequest) (*authv1.GetTokenResponse, error) {
    client, err := s.DB().GetServiceClientByID(ctx, req.ClientId)
    if err != nil {
        return nil, status.Error(codes.NotFound, "client not found")
    }

    secretOk, _ := crypto.VerifyPassword(client.ClientSecretHash, req.ClientSecret)

    if !secretOk && client.ClientSecretOld.Valid &&
       client.OldSecretExpiresAt.Valid && client.OldSecretExpiresAt.Time.After(time.Now()) {
        secretOk, _ = crypto.VerifyPassword(client.ClientSecretOld.String, req.ClientSecret)
        if secretOk {
            s.Logger().Warnf("Service client %s authenticated with old secret", client.Name)
        }
    }

    if !secretOk {
        return nil, status.Error(codes.Unauthenticated, "invalid secret")
    }

    s.DB().UpdateServiceClientUsage(ctx, client.ClientID)
    // ... continue with token generation ...
}
```

### Background Auto-Rotation Routine

```go
func serviceClientAutoRotation(a app.App) {
    lg := a.Logger().Derive(log.WithFunction("serviceClientAutoRotation"))
    ticker := time.NewTicker(24 * time.Hour)
    defer a.BackgroundWaitGroup().Done()

    for {
        select {
        case <-ticker.C:
            warningDays := getConfigInt("SERVICE_CLIENT_EXPIRY_WARNING_DAYS", 14)
            rotationGraceDays := getConfigInt("SERVICE_CLIENT_ROTATION_GRACE_DAYS", 7)

            expiringClients, err := a.Database().GetClientsNeedingRotation(ctx, warningDays, rotationGraceDays)
            if err != nil {
                lg.Errorf("Failed to get expiring clients: %v", err)
                continue
            }

            for _, client := range expiringClients {
                if client.ClientSecretOld.Valid &&
                   client.OldSecretExpiresAt.Valid &&
                   client.OldSecretExpiresAt.Time.After(time.Now()) {
                    continue // Already in grace period
                }

                newSecret, err := crypto.GenerateSecureRandomString(64)
                if err != nil {
                    lg.Errorf("Failed to generate secret for %s: %v", client.Name, err)
                    continue
                }

                hashedSecret, err := crypto.CalculatePasswordHash(newSecret)
                if err != nil {
                    lg.Errorf("Failed to hash secret for %s: %v", client.Name, err)
                    continue
                }

                oldSecretExpiresAt := time.Now().AddDate(0, 0, rotationGraceDays)
                err = a.Database().RotateServiceClientSecret(ctx, client.ClientID, hashedSecret, oldSecretExpiresAt)
                if err != nil {
                    lg.Errorf("Failed to rotate secret for %s: %v", client.Name, err)
                    continue
                }

                lg.Infof("Auto-rotated secret for service client %s", client.Name)

                a.AuditLogger().Log(ctx, audit.Event{
                    EventType: audit.EventServiceClientSecretRotation,
                    Metadata: map[string]interface{}{
                        "client_id":   client.ClientID,
                        "client_name": client.Name,
                        "rotation":    "automatic",
                        "old_expires": oldSecretExpiresAt,
                    },
                })
            }

            err = a.Database().CleanupExpiredOldSecrets(ctx)
            if err != nil {
                lg.Errorf("Failed to cleanup expired secrets: %v", err)
            }

        case <-a.BackgroundContext().Done():
            return
        }
    }
}
```

### Manual Rotation Endpoint

```go
func (s *AuthServer) RotateServiceClientSecret(ctx context.Context, req *authv1.RotateServiceClientSecretRequest) (*authv1.RotateServiceClientSecretResponse, error) {
    claims, err := s.getClaimsFromContext(ctx)
    if err != nil || !claims.IsAdmin {
        return nil, status.Error(codes.PermissionDenied, "admin access required")
    }

    newSecret, err := crypto.GenerateSecureRandomString(64)
    if err != nil {
        return nil, status.Error(codes.Internal, "failed to generate secret")
    }

    hashedSecret, err := crypto.CalculatePasswordHash(newSecret)
    if err != nil {
        return nil, status.Error(codes.Internal, "failed to hash secret")
    }

    graceDays := getConfigInt("SERVICE_CLIENT_ROTATION_GRACE_DAYS", 7)
    oldSecretExpiresAt := time.Now().AddDate(0, 0, graceDays)

    err = s.DB().RotateServiceClientSecret(ctx, req.ClientId, hashedSecret, oldSecretExpiresAt)
    if err != nil {
        return nil, err
    }

    s.AuditLog(ctx, audit.Event{
        EventType: audit.EventServiceClientSecretRotation,
        Metadata: map[string]interface{}{
            "client_id":   req.ClientId,
            "rotation":    "manual",
            "admin_id":    claims.Subject,
            "old_expires": oldSecretExpiresAt,
        },
    })

    expiryDays := getConfigInt("SERVICE_CLIENT_SECRET_EXPIRY_DAYS", 90)
    newExpiresAt := time.Now().AddDate(0, 0, expiryDays)

    return &authv1.RotateServiceClientSecretResponse{
        ClientId:         req.ClientId,
        ClientSecret:     newSecret, // Only time secret is shown in plaintext
        ExpiresAt:        timestamppb.New(newExpiresAt),
        OldSecretExpires: timestamppb.New(oldSecretExpiresAt),
        Message:          "Secret rotated successfully. Old secret valid for 7 days.",
    }, nil
}
```

---

## Environment Variables

```bash
# Metrics
METRICS_ENABLED=true
METRICS_PATH=/metrics

# Service Client Hardening
SERVICE_CLIENT_SECRET_EXPIRY_DAYS=90
SERVICE_CLIENT_EXPIRY_WARNING_DAYS=14
SERVICE_CLIENT_ROTATION_GRACE_DAYS=7
```

---

## Testing Strategy

### Unit Tests
- Metrics registration and recording
- Service client dual-secret authentication
- Auto-rotation logic
- Grace period expiration

### Integration Tests
- Prometheus scraping endpoint
- Service client authentication during grace period
- Secret rotation round-trip
- Old secret expiration

### Load Tests
- Metrics overhead measurement
- High-volume metric recording

### Manual Testing
- Verify metrics in Prometheus
- Test alerting rules
- Verify service client rotation flow
- Test old/new secret authentication
