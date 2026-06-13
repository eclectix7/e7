# E7 LLC — MacBook iOS Deployment Checklist

**Created:** 2026-05-26
**Purpose:** Complete setup guide for new MacBook + iOS deployment pipeline
**Cross-ref:** For secure agent setup, see `Hermes-Secure-Install-Proposal.md`

---

## Phase 0 — Prerequisites (Before MacBook Arrives)

These can be done today from any device:

- [ ] Verify E7 LLC DUNS number is current and accessible (you'll need it for Apple)
- [ ] Confirm E7 LLC legal entity name exactly as registered with Apple/DUNS
- [ ] Log in to [developer.apple.com](https://developer.apple.com) with your **existing Apple ID** and check account status:
  - If the old developer account is still visible but expired → you may be able to **renew** rather than re-enroll (faster)
  - If the old account is completely gone → you'll need to **re-enroll as Organization** from scratch
  - **Note:** Apple does not allow converting an Individual enrollment to Organization. If your old account was Individual, you need a fresh Organization enrollment with DUNS.
- [ ] Ensure you have access to the E7 LLC Google Workspace admin (for org-level Apple ID verification)
- [ ] Review GitHub org settings: confirm E7 LLC GitHub org has proper team/role structure
- [ ] Ensure Expo account (under E7 org email) has owner-level access to all 4 app projects
- [ ] Make a list of all provisioning profiles, certificates, and signing identities you've used before (check old laptop/keychain if accessible)

---

## Phase 1 — MacBook Hardware & OS Setup

### 1.0 Hardware Reference

| Spec | Detail |
|------|--------|
| Model | MacBook Pro 2026, M5 Max |
| RAM | 36GB unified memory (expandable to 48/64/128GB at purchase) |
| Storage | 1TB SSD |
| GPU | M5 Max integrated (hardware-accelerated ML inferencing) |
| Memory bandwidth | ~546 GB/s (M5 Max tier) |

**Implications for development:**
- 36GB unified memory fits models up to ~30-40B parameters at Q4 quantization comfortably
- 70B models at Q4 (~35GB) would barely fit and run slowly — not recommended at 36GB
- If local LLM use becomes critical, consider upgrading to 64GB or 128GB at next purchase
- 1TB SSD is sufficient for Xcode toolchain + simulators + multiple project workspaces
- Cross-ref: `Hermes-Secure-Install-Proposal.md §11` (Local LLM Analysis)

### 1.1 Initial Configuration

- [ ] Unbox, power on, complete macOS Setup Assistant
- [ ] **Create a dedicated non-admin dev user account** (e.g., `dev`) — do NOT use admin account for daily development
- [ ] Keep admin account separate for installs/security approvals only
- [ ] Enable FileVault full-disk encryption (System Settings → Privacy & Security → FileVault)
- [ ] Enable firewall (System Settings → Network → Firewall)
- [ ] Set up Time Machine backup to external drive or NAS
- [ ] Configure auto-lock / screen saver with password requirement

### 1.2 Security Hardening

- [ ] Disable remote login (SSH) unless explicitly needed — if needed, key-based auth only
- [ ] Disable remote management / screen sharing unless needed
- [ ] Review and restrict app permissions (Camera, Microphone, Full Disk Access, etc.)
- [ ] Set up a firmware password to prevent boot from external media
- [ ] Enable Find My Mac

### 1.3 Apple ID & Developer Program

- [ ] Sign in with your **existing personal iCloud Apple ID** (for iCloud services, iMessage, etc.)
  - This is the same Apple ID you previously used to access developer.apple.com and install Xcode
- [ ] Determine whether to reuse or create a new Apple ID for E7 LLC development:
  - **Option A (recommended):** Use your existing Apple ID as the E7 LLC developer account if it's not already enrolled as an individual developer
  - **Option B:** Create a dedicated Apple ID (e.g., `dev@e7llc.com`) for clean separation
  - Cross-ref: `Hermes-Secure-Install-Proposal.md §1.2` (Apple ID separation)
- [ ] Enroll E7 LLC in the **Apple Developer Program** ($99/year) as an **Organization**:
  - Use the E7 LLC legal entity name exactly as on DUNS
  - Provide DUNS number when prompted
  - You'll need to verify you have legal authority to bind the organization
  - **Note:** Previous refund was likely due to missing/invalid DUNS — have it ready this time
  - **Note:** If your existing Apple ID was previously enrolled (even if expired/refunded), you may be able to renew instead of re-enrolling. Check at developer.apple.com → Account → Membership.
- [ ] Accept the latest Program License Agreement once enrolled
- [ ] Enable **two-factor authentication** on the E7 developer Apple ID
- [ ] Add a second trusted Apple ID as an **Admin** role in App Store Connect under the E7 org (for redundancy)

---

## Phase 2 — Development Environment

### 2.1 Core Tools

- [ ] Install **Xcode** from the Mac App Store (full Xcode, not just CLT)
  - This is required for iOS Simulator, code signing, and App Store submission
  - After install, run `xcode-select --install` and accept the license
  - Open Xcode once to complete first-launch components
- [ ] Install **Xcode Command Line Tools** (if not auto-installed with Xcode)
- [ ] Install **Homebrew** (`brew.sh`)
- [ ] Install **Node.js** via Homebrew or nvm:
  ```bash
  brew install nvm
  nvm install --lts
  nvm alias default lts/*
  ```
- [ ] Install **npm/yarn/pnpm** (ensure compatibility with your Expo projects)
- [ ] Install **Git** (`brew install git`) and configure:
  ```bash
  git config --global user.name "Todd Mullen"
  git config --global user.dev.email "your-e7-email"
  ```
- [ ] Set up **SSH keypair** for GitHub:
  ```bash
  ssh-keygen -t ed25519 -C "your-e7-email"
  ```
  - Add public key to GitHub → E7 org → your account
  - Store private key in macOS Keychain
- [ ] Install **Watchman** (`brew install watchman`) — required for React Native
- [ ] Install **CocoaPods** (`sudo gem install cocoapods` or `brew install cocoapods`)
- [ ] Install **EAS CLI** globally:
  ```bash
  npm install -g eas-cli
  ```

### 2.2 GitHub & Repository Setup

- [ ] Clone all E7 app repos from GitHub org
- [ ] Verify each project builds locally (`npm install` + run)
- [ ] Review existing `eas.json` and `app.json`/`app.config.js` in each project
- [ ] Confirm bundle IDs are consistent and use E7 LLC reverse-domain notation (e.g., `com.e7llc.appname`)
- [ ] Document all bundle IDs in a central reference

### 2.3 Expo / EAS Configuration

- [ ] Log in to EAS: `eas login` (use E7 developer account)
- [ ] Configure profiles in `eas.json` for each app (development, preview, production)
- [ ] Set up **EAS Submit** credentials for each app
- [ ] Set up **EAS Build** — test a development build
- [ ] Register devices for ad-hoc distribution (your iPhone, any test devices)
  - Get device UUIDs: Xcode → Window → Devices and Simulators

### 2.4 iOS Device Setup (Physical iPhone)

- [ ] Update iPhone to latest iOS version
- [ ] Install **Apple Developer app** from App Store (for TestFlight management)
- [ ] Register iPhone UUID in Apple Developer portal (Devices)
- [ ] Trust developer certificate on iPhone after first install
- [ ] Enable Developer Mode on iPhone (Settings → Privacy & Security → Developer Mode)

---

## Phase 3 — Signing, Provisioning & App Store Connect

### 3.1 Certificates & Provisioning

- [ ] In Apple Developer portal → Certificates, Identifiers & Profiles:
  - Create **iOS Distribution Certificate** (App Store type)
  - Create/verify **App IDs** for each of the 4 apps
  - Create **Provisioning Profiles** (App Store distribution type)
- [ ] In App Store Connect:
  - Create app entries for each of the 4 E7 apps
  - Fill in all required metadata (privacy policy URL, age rating, screenshots, description)
  - Set up **App Privacy** declarations (data collection types)

### 3.2 EAS Credentials Integration

- [ ] Run `eas credentials` for each app
  - Select "iOS" → "Production"
  - Either let EAS manage credentials or upload your own
  - Ensure distribution certificate, provisioning profile, and push notification key are all set
- [ ] Build a production binary for at least one app as a test:
  ```bash
  eas build --platform ios --profile preview
  ```

### 3.3 TestFlight Submission

- [ ] Upload a build to TestFlight for at least one app
- [ ] Add internal testers (your own accounts)
- [ ] Verify build appears in TestFlight on your iPhone
- [ ] Test install → launch → core functionality flow on physical device

### 3.4 App Review & Launch Prep

- [ ] Complete **App Review Information** in App Store Connect for each app
  - Demo account credentials (if app has auth)
  - Contact information
  - Notes for reviewer
- [ ] Submit one app for **App Review** as pilot
- [ ] Iterate on review feedback
- [ ] Once approved, schedule release (manual release recommended for first launch)
- [ ] Repeat for remaining 3 apps

---

## Phase 4 — Backend & Cloud (Convex)

- [ ] Ensure all 4 apps point to correct Convex deployment URLs (production vs preview)
- [ ] Verify Convex environment variables:
  - JWT secrets
  - API keys
  - Any iOS-specific config (e.g., deep link domains)
- [ ] Set up **custom domain** for Convex if needed
- [ ] Review Convex auth configuration for iOS JWT tokens
- [ ] Test API endpoints from a simulated iOS environment (postman/curl)

---

## Phase 5 — Hermes Agent Setup on MacBook

- [ ] See `Hermes-Secure-Install-Proposal.md` for full installation guide
- [ ] Complete Mac setup (Phases 1-2) before agent installation
- [ ] Review security constraints document before granting agent access
- Cross-ref: `Hermes-Secure-Install-Proposal.md` (complete doc)

---

## Open Questions / Blockers

1. **Existing Apple Developer account status** — Log in with your existing Apple ID at developer.apple.com and determine: (a) expired but renewable org enrollment, (b) expired individual enrollment (must re-enroll as org), or (c) no account (fresh enrollment). This determines whether you pay $99 for renewal vs. new enrollment, and whether DUNS verification is needed again.
2. **Apple ID strategy for E7** — Reuse existing Apple ID or create dedicated `dev@e7llc.com`? Option A is simpler; Option B gives cleaner separation.
3. **Bundle ID audit** — Need to verify existing Expo project bundle IDs are correct and not using personal/team IDs.
4. **Convex auth for iOS** — Need to confirm JWT flow works from a real iOS device (not just Expo Go / dev client).
5. **App Store metadata** — Screenshots, descriptions, privacy policy URL for all 4 apps.
6. **Local LLM viability** — 36GB is workable for 14B-30B models. If Hermes on MacBook needs to run locally (offline/privacy), this is sufficient for coding assistance. For full agent workloads with tool calls, cloud APIs still provide better quality. See `Hermes-Secure-Install-Proposal.md §11`.
7. **DUNS verification time** — Apple org enrollment with DUNS can take 1-3 business days. **Start early.**
