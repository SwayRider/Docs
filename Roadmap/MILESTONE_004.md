# Milestone 004 - Advertisement Integration

**Status**: Planned

## Goals

Integrate a mobile ad network into the Android app to generate revenue from free-tier users. Ads are shown in non-intrusive placements (e.g. home screen banner, interstitial between route planning actions) and are suppressed for paid subscribers. The integration is GDPR-compliant with proper consent management.

## Key Deliverables

### Ad Network SDK Integration

Integrate the selected ad network SDK into the Android app. The SDK handles ad loading, display, and impression/click reporting.

> **Open Decision**: Ad network to be selected. Candidates include AdMob, Unity Ads, and others. Evaluate based on revenue share, GDPR compliance, SDK size, and fill rate for European markets. This decision must be taken at the start of this milestone.

### Consent Management (GDPR)

Implement a consent management flow compliant with GDPR and the ad network's requirements. The user is presented with a consent dialog on first launch (or after consent reset). Consent choice is stored and respected. No personalized ads are shown without explicit consent.

### Ad Placements

Ads are shown in designated placements for free-tier users:
- Banner ad on the home/map screen (non-obstructing)
- Interstitial ad between route planning sessions (with frequency cap)

Ad placements are defined as reusable UI components so placements can be adjusted without structural changes.

### Subscription-Based Ad Suppression

Paid subscribers (Tier 1 and above) do not see ads. The app checks the user's subscription status (via AuthService JWT claims or a dedicated entitlement check) and disables all ad placements for paying users.

### Ad Revenue Reporting Baseline

Basic ad impression and click metrics are available via the ad network's dashboard. No custom analytics backend is required at this stage.

## Dependencies

- MILESTONE_003 (Turn-by-Turn Navigation) — the app must be feature-complete and stable before adding ad monetization
- Ad network selection decision must be made before implementation begins

## Acceptance Criteria

- [ ] Ad network selected and decision documented in KEY_DECISIONS.md
- [ ] Ad SDK integrated into the Android build without exceeding acceptable APK size increase
- [ ] GDPR consent dialog is shown on first launch for EU users
- [ ] Consent choice is persisted and respected across sessions
- [ ] Banner ad displays on home/map screen for free-tier users
- [ ] Interstitial ad displays between route planning sessions with frequency cap
- [ ] No ads are shown to users with an active paid subscription
- [ ] Ads do not obstruct navigation UI during active navigation
- [ ] Integration passes Google Play policy review for ad content
