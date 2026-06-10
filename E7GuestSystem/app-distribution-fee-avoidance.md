# SaaS Distribution: Web/PWA + Free Mobile Apps Without App Store Fees

> **Context:** Investigating how to publish a multi-tenant SaaS product (e.g. E7 Guest System) as a Web/PWA while also offering free native mobile apps for account and user management — without triggering app store billing requirements or commission surcharges on subscriptions.

---

## User Roles & Access Methods

Before discussing distribution, it helps to clarify who uses what:

| User Type | Access Method | Purpose |
|---|---|---|
| **Account admins & staff** | Web dashboard and/or native mobile app | CRM, visitor management, messaging, reminders, analytics |
| **Visitors** | Browser only (web form via link or QR code) | One-time or repeat check-in; no app install required |

The native mobile app is exclusively a management tool for account holders. Visitors never see or interact with it. This distinction is critical to the distribution and app-store strategy below.

---

## The Core Problem

Apple and Google both take a 15–30% cut on digital goods sold through their platforms. If your SaaS charges $29/mo for a Pro subscription, Apple/Google would claim $4.35–$8.70 per transaction if that subscription is purchased *inside* an app downloaded from their store.

The goal: deliver a polished native mobile experience for admins and staff while keeping subscription purchasing on the web — completely outside the app store's billing system — and distributing the app itself at zero ongoing cost.

---

## Strategy Overview

```
┌─────────────────────────────────────────────────────┐
│               E7 Guest System SaaS                   │
│                                                       │
│   Web PWA (Primary Surface)                          │
│   ├── Visitor registration forms (public, no login)  │
│   ├── Account signup & subscription purchase (Stripe)│
│   ├── Admin dashboard & analytics                    │
│   └── All billing flows — stays in browser           │
├──────────────────────┬───────────────────────────────┤
│  Native Mobile App   │  Web PWA                      │
│  (Admin/Staff only)  │  (Everyone)                   │
│  • CRM & contacts    │  • Visitor forms              │
│  • Visitor mgmt      │  • Subscription purchase      │
│  • Messaging         │  • Dashboard / analytics      │
│  • Reminders         │  • Account settings           │
│  • Analytics         │                               │
│  • NO billing UI     │                               │
├──────────────────────┴───────────────────────────────┤
│  Subscribers pay on web → App reads auth only.       │
│  No transaction touches App Store or Play Store.     │
└──────────────────────────────────────────────────────┘
```

Three pillars:

1. **PWA as the primary product surface** — visitor-facing forms, account signup, subscription management, dashboards all live on the web. This is the only thing visitors ever interact with.
2. **Free native companion app** — admin/staff-only interface for on-the-go visitor management and messaging. No pricing in the app. No in-app purchases. No visitor-facing content.
3. **Subscription purchase happens only on the web** — this is the critical architectural decision that keeps Apple and Google out of your revenue.

---

## Pillar 1 — Progressive Web App (PWA)

A PWA runs in the browser but behaves like an installed app: offline support, push notifications, home-screen icon, full-screen mode.

### What the PWA covers

| Capability | Audience | Status |
|---|---|---|
| Visitor registration form | Visitors | ✅ Full support — this is the primary visitor touchpoint |
| QR code scan → form (via browser camera API) | Visitors | ✅ Android native; iOS Safari has limited support |
| Account signup & login | Admins/staff | ✅ Full support |
| Subscription purchase (Stripe checkout) | Admins/staff | ✅ Full support — no app store involved |
| Form configuration | Admins/staff | ✅ |
| Dashboard / analytics | Admins/staff | ✅ |
| Push notifications (via service worker) | Admins/staff | ✅ Android. iOS 16.4+ supports PWA push. |
| Offline form caching & queue | Visitors | ✅ via service worker + IndexedDB |

### PWA advantages

- **Zero distribution cost.** No app store, no developer program fees, no review process.
- **Instant updates.** No waiting for Apple/Google review. Deploy on Netlify and it's live.
- **Single codebase.** No divergent iOS/Android versions.
- **Discoverable via search engines** — organic traffic, no keyword competition in stores.
- **Shareable URL** — a church member can text a link; no app install friction for visitors.

### PWA limitations

| Limitation | Impact | Mitigation |
|---|---|---|
| iOS Safari has restricted background capabilities | Push notifications less reliable on iOS | Native companion app handles critical push use cases for staff |
| Camera/QR scanning in browser | iOS Safari camera UX is clunky | Native app for QR scanning; PWA fallback via manual URL entry. Visitors can also just type the URL. |
| No access to all native APIs | Bluetooth, NFC, advanced background processing unavailable | Not needed for this use case |
| iOS home-screen PWA data cap | ~50 MB per origin in some iOS versions | E7's data footprint is well below this |

**Verdict:** PWA is the primary delivery mechanism. It covers all visitor interactions and most admin tasks, eliminating app store dependency entirely.

---

## Pillar 2 — Free Native Companion App (Admin/Staff Only)

A lightweight native app built with Expo (React Native) that provides the admin/staff mobile interface. The key: **the app contains no purchasable content, no subscription screen, and no paywall.** It is a free CRM tool for people who already have an account on the web.

### Important: visitors never see this app

The native app is not marketed to, downloadable by, or useful for visitors. Visitors interact exclusively through the PWA in their browser. The app exists solely to give account admins and staff a better mobile experience for managing their data on the go.

### What the companion app does (free, no in-app purchase)

- Login to an existing E7 Guest System account (web-based auth token)
- View and manage visitors at assigned locations
- Read and reply to visitor messages (via in-app or SMS relay)
- Set and manage reminders to follow up with visitors
- View contact notes and history
- Browse basic analytics

### What the companion app does NOT do

- ❌ Display any pricing or subscription options
- ❌ Offer any upgrade, purchase, or subscription flow
- ❌ Process any payment
- ❌ Gate any feature behind a paywall
- ❌ Serve any visitor-facing content

### Architecture: Auth via web session

```
1. User logs in on web PWA → receives auth token/cookie from Convex
2. User opens native app → presented with a "Link your account" screen
3. App opens a web-view or external browser for one-time OAuth login
4. Native app receives a session token, stores it securely (Expo SecureStore)
5. All subsequent API calls use this token — no payment involved
```

This pattern is common and accepted by both stores. Examples: Slack, Discord, Notion, and Shopify all have free mobile apps where the subscription is managed on web. None of them offer subscription purchase inside the app — or if they do, they also offer the "manage on web" escape hatch and pay the Apple/Google commission on that subset only.

For E7, since **all subscriptions happen on the PWA**, the app is simply a free management tool.

---

## Pillar 3 — Distribution Channels for the Free App

### Option A: Publish on Google Play & App Store (simplest)

| Platform | Cost | Effort | Notes |
|---|---|---|---|
| Google Play | $25 one-time developer fee | Low — Expo EAS builds and submits | Free app listing, no commission since no IAP |
| Apple App Store | $99/year developer fee | Medium — Apple review process | Free app listing, no commission since no IAP |

**Annual cost: $124** ($99 Apple + $25 Google one-time). This is fixed and recurs only on the Apple side.

**Key consideration:** Even as a free app, Apple and Google may flag the app if they detect subscription-related language or screenshots. Guidelines:
- Do not mention pricing in the app listing.
- Do not show any subscription UI in the app.
- Screenshots should demonstrate the management interface (visitor list, message threads, reminders) — not billing.
- App store metadata can say something like: "Companion app for E7 Guest System. Manage visitors, messages, and reminders on the go. Subscription management available on the web."
- Include a link to the web PWA for account setup (Apple allows external links in free apps under certain conditions — updated per 2024 App Store guidelines).

### Option B: PWA-Only (Zero Store Fees)

Skip the native app entirely. The PWA, when saved to the home screen, looks and behaves like a native app on both platforms.

| Pros | Cons |
|---|---|
| $0 distribution cost | No presence in app stores (discoverability gap) |
| No review process | iOS Safari limitations (QR scanning, push reliability) |
| Single codebase, single deployment | Staff users may not perceive it as "an app" |

**Verdict for early stage:** Viable for MVP. All visitor interaction is through the browser anyway. If staff users need better push notifications or QR scanning, add the native app later.

### Option C: Sideloading & Alternative Distribution

For reaching staff users without paying Apple/Google:

| Method | Platform | Cost | Notes |
|---|---|---|---|
| **TestFlight** | iOS | Free (Apple Developer Program required, $99/yr) | Up to 10,000 external testers. Good for beta. |
| **Ad-hoc distribution** | iOS | Free (requires $99 dev program) | Up to 100 devices per year. Manual UDID registration. |
| **AltStore PAL** / **Sideloadly** | iOS | Free–$80/yr (AltStore PAL subscription) | Allows self-hosted IPA installation without developer program. Requires re-signing every 7 days (free tier) or 7 days with paid AltStore PAL. Viable for small church staff deployments. |
| **APK direct install** | Android | $0 | Android allows sideloading by default. Generate a signed APK via Expo and distribute via URL or file sharing. |
| **Apple Business Manager** | iOS | $0 (requires Apple Business Manager account) | Designed for organizations. Allows silent app deployment to managed devices. Churches with an MDM solution can push the app to staff devices without App Store. |
| **Google Managed Google Play** | Android | $0 | Similar — organizations can privately publish apps to their managed devices. |

**For a church-focused SaaS:** Option C methods are particularly relevant. Many churches manage staff devices through an IT team and can deploy via MDM or simple APK sideloading.

### Option D: Hybrid — PWA Primary, Native App Optional (Recommended)

This is the recommended approach and ties all three pillars together:

1. **All users start with the PWA.** Visitors use the browser form; staff use the web dashboard for signup, billing, and day-to-day management.
2. **Power users install the native companion app** for push notifications, smoother camera/QR experience, and offline access. The app is for staff/admins only.
3. **The app is free and has no billing.** Users who want to support the service purchase subscriptions through the web PWA.
4. **Distribute the native app** through App Store and Play Store for convenience, or sideload for zero cost.

---

## Revenue Model Interaction

Here is exactly how money flows and where app stores do (or don't) get involved:

```
Church staff member subscribes at $29/mo
         │
         ▼
   Web PWA (Stripe Checkout)
         │
         ├── Apple/Google: $0 (transaction happened in browser, not in app)
         ├── Stripe: ~$0.87 (2.9% + $0.30)
         └── You receive: ~$28.13

Staff member installs free companion app
         │
         ▼
   App contacts Convex API (reads/writes CRM data only)
         │
         ├── Apple/Google: $0 (app is free, no IAP)
         ├── Your server costs: covered by subscription revenue
         └── You receive: $0 additional cost

Visitor scans QR code and fills out form
         │
         ▼
   Browser loads PWA form → submits to Convex
         │
         ├── Apple/Google: $0 (browser, not an app store product)
         ├── SMS sent via Twilio: ~$0.0079
         └── You receive: $0 (visitor interaction is covered by the org's subscription)
```

**Key legal note:** Apple's App Store Review Guidelines (§3.1.3) allow "reader" apps — apps that let users access content or services they already have access to. Since the companion app is free and simply provides an alternative interface to a web-based subscription service (with no in-app purchase mechanism), it qualifies under both Apple and Google policies.

Google Play's In-App Products policy states that Google Play Billing must be used for *digital goods and services purchased inside the app.* An app that accesses a web-purchased subscription via authentication does not trigger Google Play Billing requirements.

---

## Comparison: Different Distribution Models Side by Side

| Factor | PWA Only | PWA + App Store Apps | PWA + Sideloading |
|---|---|---|---|
| **Annual distribution cost** | $0 | ~$124 ($99 Apple + $25 Google) | $0–$80 (AltStore PAL) |
| **App Store commission** | $0 | $0 (free app, no IAP) | $0 |
| **App review required** | No | Yes (2–7 days typical) | No |
| **Push notifications** | Limited on iOS | Full native support | Full native support |
| **QR camera scanning** | Partial on iOS | Full native support | Full native support |
| **Offline support** | Service worker | Full native | Full native |
| **User discoverability** | Search engines only | App store search | Word of mouth / direct link |
| **Update speed** | Instant (deploy) | Apple/Google review delay | Manual re-sign/install |
| **Maintenance burden** | One codebase | Two builds, two review cycles | Two builds, manual distribution |
| **Trust / perception (staff)** | Medium ("it's a website") | High ("it's in the App Store") | Low–Medium |
| **Visitor experience** | Identical across all options | N/A (visitors use browser only) | N/A (visitors use browser only) |

---

## Recommendations for E7 Guest System

### Phase 1 (MVP): PWA Only

- Build and ship the PWA. This covers all visitor interactions and the admin web dashboard.
- Collect feedback from staff on pain points (camera scanning, notifications, offline).
- All billing is on the web. Zero app store involvement.
- Staff use the web dashboard on mobile browsers in the meantime.

### Phase 2 (Growth): Add Free Companion App to Stores

- Once the PWA is stable, build a thin native companion app using Expo for staff/admin.
- Submit to Google Play ($25) and App Store ($99/yr).
- Keep all subscription purchasing in the PWA. The app is a management tool only.
- Update the PWA to prompt staff users to install the native app for push notifications and offline access.
- Visitors continue using the browser — they never see or need the app.

### Phase 3 (Scale): Alternative Distribution

- Offer sideloading packages (APK for Android, AltStore for iOS) for churches with managed device fleets.
- Explore Apple Business Manager and Google Managed Google Play for organizational deployments.
- Consider a dedicated church admin portal for bulk device enrollment.

---

## Summary

- **Visitors never use the native app.** All visitor interaction (registration forms, SMS follow-up) happens through the browser-based PWA.
- **The native app is exclusively for account admins and staff** as a CRM/management tool — not a visitor-facing product.
- **You can publish the full SaaS product as a PWA with zero app store costs.** All subscriptions are purchased on the web via Stripe, bypassing Apple and Google commission entirely.
- **A free native companion app for mobile management costs $0 in ongoing store fees.** Google Play has a $25 one-time registration; Apple charges $99/year. Neither charges commission on a free app with no in-app purchases.
- **The "web purchase + free app" pattern is well-established** (Slack, Notion, Discord, Shopify) and is explicitly permitted by both Apple and Google as long as the app does not offer subscription purchase flows.
- **Start with PWA only.** Add native apps only when mobile-specific features (push notifications, QR scanning, offline) justify the added complexity for staff users.

---

*Last updated: June 2026*
*Related documents: ../E7GuestSystem/E7GuestSystem.md, ../E7GuestSystem/pricing-addendum.md*