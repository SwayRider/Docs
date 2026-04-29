# Milestone 006 - Polish & Play Store Release

**Status**: Planned

## Goals

Bring the Android app to production quality and publish it on the Google Play Store. This milestone covers error handling, offline and connectivity states, a first-run onboarding flow, analytics and crash reporting, and all Play Store submission requirements. At the end of this milestone, SwayRider is publicly available on the Play Store.

## Key Deliverables

### Error Handling & User Feedback

All network calls and backend interactions display meaningful error messages instead of crashes or silent failures. Transient errors (timeout, server error) offer a retry action. Permanent errors (auth failure, not found) navigate the user to an appropriate recovery state. Loading states are consistently indicated with progress indicators.

### Offline & Connectivity States

The app detects network connectivity loss and displays an appropriate offline indicator. Features that require connectivity (route planning, search) are gracefully disabled with an explanation. Previously loaded map tiles remain visible when offline.

### First-Run Onboarding

New users are guided through a short onboarding flow on first launch:
- App value proposition (2-3 screens)
- Location permission request with explanation
- Notification permission request (for navigation prompts)
- Account creation or login prompt

Onboarding is skippable and does not block app use.

### Analytics & Crash Reporting

A crash reporting tool (e.g. Firebase Crashlytics or Sentry) is integrated to surface runtime crashes in production. Basic anonymous usage analytics (screen views, key actions) are collected with user consent. No personally identifiable data is included in analytics events.

### Play Store Submission Requirements

All requirements for Google Play publication are met:
- Privacy policy URL linked from the app and Play Store listing
- App icon, screenshots, and feature graphic prepared for all required resolutions
- Play Store listing text (short description, full description) written
- Content rating questionnaire completed
- Target SDK and minimum SDK meet current Play Store requirements
- App passes Android Vitals baseline (no ANRs, no crashes on launch)

### Final QA Pass

A structured manual test pass is performed covering all user-facing flows: registration, login, route planning, navigation, account management, and ad display. All P0 and P1 issues are resolved before submission.

## Dependencies

- MILESTONE_005 (Account Management) — all user-facing features must be complete before production polish and release

## Acceptance Criteria

- [ ] All error states display user-friendly messages with recovery actions
- [ ] Offline state is detected and communicated to the user
- [ ] Onboarding flow runs on first launch for new users
- [ ] Location and notification permissions are requested with explanations
- [ ] Crash reporting is active and captures crashes in production builds
- [ ] Analytics events fire for key user actions without PII
- [ ] Privacy policy is accessible from within the app
- [ ] Play Store listing assets (icon, screenshots, descriptions) are complete
- [ ] App passes Android Vitals baseline (no launch crashes, no ANRs)
- [ ] App is published and publicly available on the Google Play Store
