# REQUIREMENT_002 — iOS App

## Overview

Develop the iOS application using SwiftUI, consuming the KMP shared module for business logic while maintaining native iOS UI/UX.

## Context

- **Components**: iOS (SwiftUI)
- **Priority**: High
- **Status**: Planned

## Requirements

### App Features (MVP Parity)
- All Android MVP features:
  - Map viewing with MapLibre
  - Route planning
  - Turn-by-turn navigation
  - Location search
  - Offline maps
  - Points of interest
  - User authentication

### iOS-Specific
- SwiftUI native UI
- iOS design guidelines (Human Interface Guidelines)
- iOS-specific features:
  - Apple Maps integration (optional)
  - iOS notifications
  - Keychain for secure storage
  - Widgets (future)

### Architecture
- MVVM pattern
- Combine for reactive programming
- Swift Package Manager for dependencies
- SwiftUI previews for development

## Acceptance Criteria

1. App builds and runs on iOS 16+
2. All Android MVP features implemented
3. UI follows iOS design guidelines
4. Performance matches Android app
5. Offline functionality works
6. Navigation works with voice guidance
7. App Store ready

## Affected Files

### New Directory
- `mobile/ios/` — iOS app root

### Integration
- KMP framework imported
- Shared models used
- Platform-specific code isolated
