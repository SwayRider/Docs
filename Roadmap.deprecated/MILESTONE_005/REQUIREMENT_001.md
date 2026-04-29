# REQUIREMENT_001 — KMP Shared Module

## Overview

Create a Kotlin Multiplatform shared module containing network clients, authentication logic, domain models, and API integration code that can be used by both Android and iOS apps.

## Context

- **Components**: Shared Kotlin Code
- **Priority**: High
- **Status**: Planned

## Requirements

### Shared Module Structure
```
shared/
├── build.gradle.kts
├── src/
│   ├── commonMain/        # Shared code
│   │   ├── kotlin/
│   │   │   ├── data/      # Network clients, repositories
│   │   │   ├── domain/    # Models, use cases
│   │   │   └── di/        # Dependency injection
│   ├── commonTest/
│   ├── androidMain/       # Android-specific
│   └── iosMain/           # iOS-specific
```

### Shared Components
- **Network Layer**: Ktor HTTP client
- **Authentication**: JWT handling, token management
- **Domain Models**: Route, Location, POI, User
- **Repositories**: Data access abstraction
- **State Management**: StateFlow/LiveData bridges

### Platform Expectations
- `expect`/`actual` for platform-specific implementations
- iOS-specific APIs (Keychain, notifications)
- Android-specific APIs (location, permissions)

## Acceptance Criteria

1. Module compiles for Android and iOS targets
2. Network clients work on both platforms
3. Auth logic shared and functional
4. Domain models identical across platforms
5. Unit tests pass for shared code
6. iOS framework generated successfully

## Affected Files

### New Directory
- `shared/` — KMP module root

### Modified
- `mobile/android/app/build.gradle.kts` — Add shared dependency
- `settings.gradle.kts` — Include shared module
