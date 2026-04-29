# Key Decisions — SwayRider Roadmap

This file records the key architectural and strategic decisions that shape the roadmap, including decisions already made, their rationale, and decisions still open.

---

## Decided

### Mobile-First Strategy — Web Portal Removed

**Decision**: The React web authentication portal is removed from scope. SwayRider is a mobile-only product.

**Rationale**: The web portal added maintenance overhead without aligning with the product vision. All user-facing functionality is delivered through the Android (and future iOS) app.

**Status**: Implemented (portal already removed).

---

### Custom Vector Tiles (in-house, MapLibre)

**Decision**: Map tiles are generated in-house from OpenStreetMap using the Python data pipeline (Tippecanoe, Osmium) and served as MBTiles via TilesService. Rendering is done client-side with MapLibre.

**Rationale**: Full control over feature selection, styling, and data freshness. No per-request tile costs (vs. Mapbox/Google Maps). Offline support is a first-class requirement.

**Status**: Implemented.

---

### Regional Routing with Valhalla

**Decision**: Routing is handled by multiple regional Valhalla instances with border-crossing logic in RouterService.

**Rationale**: Horizontal scalability across geographies. Each region can be updated or scaled independently. Seamless multi-region routes are handled at the service layer.

**Status**: Implemented.

---

### Kotlin Multiplatform (KMP) for iOS

**Decision**: The iOS app will be developed using Kotlin Multiplatform, sharing business logic (network clients, auth, domain layer) with the Android app. Native UI on each platform (Jetpack Compose / SwiftUI).

**Rationale**: Code reuse reduces duplication for network and auth logic. Native UI ensures platform-appropriate user experience. iOS development begins after Android MVP.

**Status**: Planned (iOS not yet started).

---

### GDPR-Compliant Architecture (EU)

**Decision**: All user data is stored and processed within the EU. GDPR compliance is a hard requirement, including consent management for ads, right to erasure for account deletion, and no PII in analytics events.

**Rationale**: Target market is European motorcycle riders. Non-compliance would be a legal and reputational risk.

**Status**: Ongoing requirement across all milestones.

---

## Open Decisions

### Ad Network Selection (MILESTONE_004)

**Decision needed**: Select the mobile ad network to integrate into the Android app.

**Candidates**: AdMob (Google), Unity Ads, and others.

**Evaluation criteria**:
- Revenue share and fill rate for Western European markets
- GDPR compliance and Transparency & Consent Framework (TCF) support
- Android SDK size impact
- Ease of integration with Jetpack Compose

**Must be decided at the start of MILESTONE_004.**

---

### Offline Map Download Strategy (MILESTONE_003 or later)

**Decision needed**: How are map tiles made available offline?

**Options**:
- Bundled tiles (pre-packaged MBTiles included in APK or OBB) — simple but large download
- On-demand regional download — user selects regions to download; more flexible but requires download management UI
- Hybrid — lightweight base tiles bundled, detailed tiles downloaded on demand

**Must be decided before implementing offline map support.**

---

### Subscription Payment Provider (Post-MILESTONE_006)

**Decision needed**: Select the payment provider for the subscription model.

**Candidates**: Google Play Billing (mandatory for in-app purchases on Android), Stripe (for web/cross-platform billing), or a combination.

**Note**: Google Play Billing is required for any in-app subscription sold through the Play Store. A server-side subscription service will need to validate Play purchases.

**Must be decided before implementing the subscription model (post-MVP milestone).**

---

### Production Hosting / Cloud Provider (Post-MILESTONE_006)

**Decision needed**: Where is SwayRider hosted in production?

**Current state**: Self-hosted via Docker Compose on development infrastructure.

**Options**: Cloud provider (AWS, GCP, Azure, Hetzner Cloud) or colocation. Evaluated based on cost, latency to European users, and operational complexity at scale.

**Must be decided before Play Store release or shortly after, depending on load requirements.**
