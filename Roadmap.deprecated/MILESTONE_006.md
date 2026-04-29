# MILESTONE_006 — Subscription & Payment

## Overview

Implement subscription management and payment processing to support the freemium business model with free and paid tiers.

## Scope

- **Phase**: Post-MVP
- **Priority**: High
- **Dependencies**: MILESTONE_005 (iOS app available)
- **Blocks**: None (but required for revenue)

## Background

SwayRider is a subscription-based service with a free tier and paid plans. Payment provider selection is a pending decision point.

## Requirements

| ID | Name | Components | Status |
|----|------|------------|--------|
| [REQUIREMENT_001](./MILESTONE_006/REQUIREMENT_001.md) | Subscription Management | Backend, Mobile | Planned |
| [REQUIREMENT_002](./MILESTONE_006/REQUIREMENT_002.md) | Payment Integration | Backend, Mobile | Planned |

## Affected Components

### Backend
- Subscription service (new)
- User tier management
- Feature gating logic

### Mobile Apps
- Subscription UI
- Payment flow
- Feature access control

### Infrastructure
- Payment provider integration
- Webhook handling

## Subscription Model

| Tier | Capabilities |
|------|-------------|
| **Free** | Basic route planning, limited map downloads, standard search |
| **Paid (Tier 1)** | Full turn-by-turn navigation, unlimited offline maps, POI access |
| **Paid (Tier 2)** | Group riding features, route sharing/community, priority support |

## Success Criteria

1. Users can view subscription options
2. Payment processing works on both platforms
3. Subscription status synced across devices
4. Feature access controlled by tier
5. Subscription renewal handled automatically
6. Cancellation flow works
7. Receipts and invoices generated

## Timeline Estimate

| Requirement | Estimated Effort |
|-------------|------------------|
| REQUIREMENT_001 | 3-4 weeks |
| REQUIREMENT_002 | 4-6 weeks |
| **Total** | **7-10 weeks** |

## Key Decisions

| Decision | Status | Notes |
|----------|--------|-------|
| Payment Provider | TBD | Stripe, RevenueCat, or platform-native |
| Subscription Sync | TBD | Server-side vs client-side validation |
| Trial Periods | TBD | Duration and tier |
| Family Sharing | TBD | Future consideration |
