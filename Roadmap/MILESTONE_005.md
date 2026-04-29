# Milestone 005 - Account Management

**Status**: Planned

## Goals

Give users full control over their account from within the Android app. Users can view and edit their profile, change their password, and permanently delete their account. All flows are backed by AuthService and comply with GDPR right-to-erasure requirements.

## Key Deliverables

### User Profile Screen

A dedicated profile screen is accessible from the app's main navigation. It displays the user's current account information: display name, email address, and account creation date. The user can edit their display name and save changes via AuthService.

### Change Password Flow

From the profile screen, the user can initiate a password change. The flow requires the current password for verification, then accepts and confirms a new password with validation (minimum length, complexity). On success, all existing refresh tokens are invalidated and the user is prompted to log in again.

### Delete Account Flow

The user can permanently delete their account from the profile screen. A confirmation dialog explains that the action is irreversible and all personal data will be removed. On confirmation, AuthService deletes the user record and all associated data (tokens, password history, audit log entries). The app logs the user out and returns to the login screen. This satisfies the GDPR right to erasure.

### AuthService — Profile & Account Deletion API

AuthService exposes the necessary gRPC endpoints:
- Update display name
- Change password (with current password verification and token invalidation)
- Delete account (full data erasure)

## Dependencies

- MILESTONE_004 (Advertisement Integration) — the app must be release-ready with monetization before finalizing account management

## Acceptance Criteria

- [ ] User profile screen displays name, email, and account creation date
- [ ] Display name can be edited and saved
- [ ] Change password requires current password verification
- [ ] Successful password change invalidates all existing refresh tokens
- [ ] Delete account requires explicit confirmation
- [ ] Account deletion removes all user data from the database
- [ ] App returns to login screen after account deletion
- [ ] All flows work end-to-end on a physical Android device
- [ ] GDPR right-to-erasure is fully satisfied by account deletion
