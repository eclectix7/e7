# E7 Guest System — Startup SaaS Infrastructure Pricing Addendum

> **Purpose:** Estimate the real-world operating costs of providing the E7 Guest System service so we can validate that our customer-facing tier pricing covers expenses with healthy margins.
>
> **Scope:** Baseline infrastructure and delivery costs only. No premium add-ons (short codes, dedicated phone numbers beyond the included, white-label domain setup fees, etc.) as specified.

---

## 1 — SMS Messaging (Twilio)

Twilio is the assumed provider. Pricing based on US/Canada long-code numbers.

| Item | Cost |
|---|---|
| Twilio phone number (1 long code) | **$1.15/mo** |
| Outbound SMS (US, per message) | **$0.0079** |
| Inbound SMS (US) | **$0.0079** |
| Toll-free number (optional upgrade path) | ~$2.00/mo |

### Projected Monthly SMS Costs by Tier

| Tier | Visitors/Mo | Msgs Sent (welcome + follow-ups) | Est. Inbound Replies | Total Msgs | Monthly SMS Cost |
|---|---|---|---|---|---|
| **Starter** (free tier) | ≤100 | 100 | ~20 | 120 | **$0.95** |
| **Pro** | ≤1,000 | 1,000 | ~200 | 1,200 | **$9.48** |
| **Enterprise** (assume avg) | ≤5,000 | 5,000 | ~1,000 | 6,000 | **$47.40** |
| **Scale** (future) | ≤20,000 | 20,000 | ~4,000 | 24,000 | **$189.60** |

> **Note:** A single Twilio number handles ~1 message/second. For the Pro+ tiers, consider a second number or a messaging service pool.

---

## 2 — Convex Backend

Convex provides serverless compute, database, and real-time subscriptions in one platform.

| Item | Cost |
|---|---|
| Free tier | 10M function invocations/mo, 1 GB data, 512 MB storage — **$0** |
| Pro plan | Unlimited functions, 10 GB data — **$29/mo** |
| Scale (hypothetical heavy usage) | Estimated $50–80/mo at the Enterprise tier load |

**Assumption:** For MVP and early launch, the **free tier** is sufficient. Pro plan activates when customer load outgrows free limits (roughly >500 active visitors/month with active messaging).

### What counts against Convex limits

| Operation | Frequency per visitor interaction |
|---|---|
| `visitors.create` | 1 write |
| `message_threads.addMessage` | 1–3 writes (welcome + follow-ups) |
| `audit_log.write` | 1–2 writes |
| Real-time subscription updates | Few reads/pushes per active staff user |

---

## 3 — Web Frontend Hosting (Netlify)

| Item | Cost |
|---|---|
| Starter tier (3 team members) | **$0** — 100 GB bandwidth/mo, 300 build min/mo |
| Pro tier (password-protected staging, SSO) | **$19/mo/member** |
| Enterprise (SLA, audit log) | Custom pricing |

**Assumption:** Starter tier is sufficient through MVP and early growth. Upgrade to Pro only if SSO or branch deploys with password protection are needed.

| Projected Bandwidth Usage | Estimate |
|---|---|
| Starter tier visitors (100/mo) | <1 GB/mo |
| Pro tier visitors (1,000/mo) | ~5 GB/mo |
| Enterprise (5,000/mo) | ~20 GB/mo |

> Netlify Starter includes 100 GB — unlikely to exceed this even at Enterprise scale unless serving large assets.

---

## 4 — Mobile App (Expo)

| Item | Cost |
|---|---|
| Expo free tier | Build pushes, OTA updates, 30K notification receipts/mo — **$0** |
| Expo Pro (priority builds, SSO, notifications overage) | **$9/mo** per seat (build) |
| Apple Developer Program | **$99/year** ($8.25/mo) |
| Google Play Developer (one-time) | **$25** (one-time; ~$2.08/mo amortized over year) |
| Push notification volume (Expo/EAS) | Covered under free tier; bills against APNs/FCM which are free |

**Assumption:** Expo free tier + Apple/Google developer accounts cover MVP through growth.

---

## 5 — Transactional Email (Resend)

For account invites, password resets, receipts, and data-export deliveries.

| Item | Cost |
|---|---|
| Resend free tier | 3,000 emails/mo — **$0** |
| Resend paid tier | 50,000 emails/mo — **$20/mo** |

**Assumption:** Free tier easily covers all transactional email through MVP and well past it.

---

## 6 — Monitoring & Analytics

| Service | Plan | Monthly Cost |
|---|---|---|
| Sentry (error tracking) | Developer (5K events/mo) | **$0** (free tier) |
| PostHog (product analytics) | Free (1M events/mo) | **$0** |
| Better Uptime (monitoring) | Starter (10 monitors) | **$0** (free tier) |

**Assumption:** All three platforms have generous free tiers that cover a startup through significant growth.

---

## 7 — Domain & DNS

| Item | Cost |
|---|---|
| Primary domain (e.g., `.com`) | **$12–15/year** (~$1.10/mo) |
| DNS (Cloudflare free) | **$0** |
| Optional: subdomain for white-label | Included in Cloudflare plan |

**Assumption:** Cloudflare free plan handles DNS, CDN, and basic DDoS protection.

---

## 8 — Stripe Billing

| Item | Cost |
|---|---|
| Stripe billing fees | **2.9% + $0.30** per successful charge |
| Stripe billing (annual subscription) | Typically discounted to **2.0% + $0.30** by Stripe for high volume |

### Example Stripe Costs

| Monthly Subscription Revenue | Stripe Fees (2.9% + $0.30) | Net After Fees |
|---|---|---|
| $29 (1 Pro customer) | $1.14 | $27.86 |
| $290 (10 Pro customers) | $11.41 | $278.59 |
| $990 (10 Enterprise customers) | $31.71 | $958.29 |
| $5,000/mo revenue | $172.00 | $4,828.00 |

---

## 9 — Miscellaneous / Startup Operational Costs

| Item | Estimated Cost |
|---|---|
| Vercel/Netlify premium domain hosting (if needed) | $0–20/mo |
| 1Password or Bitwarden (team password manager) | $4–8/mo per seat |
| GitHub Pro (private repos, larger storage) | $4/mo per seat |
| Legal review (GDPR compliance, ToS, DPA) | $500–1,500 one-time |
| Accounting software (Wave — free, or QuickBooks) | $0–30/mo |

---

## 10 — Total Monthly Cost Summary — By Scale

### MVP Phase (first 3 months, pre-revenue)

| Cost Center | Monthly |
|---|---|
| Twilio (100 visitors) | $0.95 |
| Convex free tier | $0 |
| Netlify Starter | $0 |
| Expo + Apple + Google | $10.33 |
| Resend | $0 |
| Sentry + PostHog + Uptime | $0 |
| Domain + Cloudflare | $1.10 |
| Stripe billing (on revenue) | volume-dependent |
| **Total fixed overhead** | **~$12.38/mo** |

### Growth Phase (100–1,000 visitors/month, ~50 paying Pro accounts)

| Cost Center | Monthly |
|---|---|
| Twilio (~6,000 msgs) | $47.40 |
| Convex Pro | $29 |
| Netlify Starter→Pro | $0–19 |
| Expo + Apple + Google | $10.33 |
| Resend | $0 |
| Sentry + PostHog + Uptime | $0 |
| Domain + Cloudflare | $1.10 |
| Stripe fees (~$1,450 revenue) | ~$44.95 |
| **Total** | **~$86–105/mo** |

### Scale Phase (5,000+ visitors/month, 10+ Enterprise accounts)

| Cost Center | Monthly |
|---|---|
| Twilio (~30,000 msgs) | $284.40 |
| Convex Pro+ | $50–80 |
| Netlify Pro | $19 |
| Expo Pro + Apple + Google | $19.33 |
| Resend paid | $0–20 |
| Sentry/PostHog/Uptime | $0 (free tier likely still sufficient) |
| Domain + Cloudflare | $1.10 |
| Stripe fees (~$9,900 revenue) | ~$317 |
| **Total** | **~$435–485/mo** |

---

## 11 — Margin Analysis at Proposed Tier Prices

| Tier | Price/Cust | Est. Costs/Cust* | Gross Margin |
|---|---|---|---|
| **Starter** (free) | $0 | ~$0.10 (SMS at 100 visitors) | N/A (acquisition funnel) |
| **Pro** | $29/mo | ~$1.50 (pro-rata SMS + backend) | **~95%** |
| **Enterprise** | $99/mo | ~$3.00 (pro-rata SMS + backend) | **~97%** |

_*Per-customer costs are rough allocations of shared infrastructure + SMS at assumed average usage. SMS is the dominant variable cost._

### Key Takeaway

The proposed pricing tiers have **very healthy margins** even at low subscriber counts. The primary variable cost driver is **SMS volume**, not platform infrastructure. Free-tier users cost nearly nothing. The main financial risk is not unit economics but **customer acquisition volume** — the SaaS needs enough paying subscribers to cover the fixed costs of development time, legal setup, and tooling (~$200–$500/mo in founder/developer time even at MVP stage).

---

## 12 — Notes & Open Questions

1. **SMS short codes** (5–6 digit numbers) cost ~$500–1,000/mo to lease and require carrier approval. Excluded per scope — long codes are sufficient for now.
2. **Toll-free numbers** (~$2/mo) may be worth activating if opt-out rates suggest carrier filtering on long codes.
3. **Convex pricing could change** — they are still in growth-stage pricing. Monitor for plan restructuring.
4. **Email deliverability:** Resend's free tier is generous, but if transactional email volume grows past 3K/mo, budget $20/mo.
5. **Should SMS costs be passed through or bundled?** At 1,000 visitors/mo on Pro tier, SMS cost is ~$9.48 against $29 revenue. Bundling is viable. At Enterprise scale (5K visitors), SMS is ~$47 against $99 — still comfortable but worth monitoring.

---

*Last updated: June 2026*
*Companion to: E7GuestSystem.md*