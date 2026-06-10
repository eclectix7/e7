## Deferred & Future Features

> These features are **explicitly out of scope for the MVP** (Phase 1). They will be evaluated, prioritized, and scheduled in future development cycles.

---

### 1 · Fully Custom Location Forms (DEFERRED)

> **Why deferred:** MVP uses standardized form templates with configurable field toggles (name, party size, local/moving, etc.). Building a drag-and-drop custom form builder with conditional logic requires significant frontend and validation infrastructure that adds risk and scope to the initial release.

**What this includes when revisited:**
- Drag-and-drop form builder UI for account admins
- Custom field types: file uploads, multi-select, date pickers, radio buttons, conditional branching logic
- Per-field validation rules & required/optional toggles
- Form preview mode before publishing
- Version history for form revisions
- A/B testing between form variants to optimize conversion

**Estimated phase:** Phase 2 (Weeks 11–18), after core visitor flow is stable

---

### 2 · Public REST API for Third-Party Integrations

> Allow churches/businesses to sync visitor data with external tools (ChurchTrac, Planning Center, Mailchimp, etc.).

**What this includes:**
- OAuth2-based API authentication per account
- Endpoints: `GET /visitors`, `POST /visitors/webhook`, `GET /analytics`
- Webhook subscriptions for real-time event notifications (new visitor, SMS sent, opt-out)
- Rate limiting and usage tracking per API key

**Estimated phase:** Phase 4 (Weeks 19+)

---

### 3 · Scan & Extract Analog Visitor Info

> Digitize paper visitor cards or analog sign-in sheets by scanning and extracting visitor data (OCR).

**Estimated phase:** Future — requires OCR/AI evaluation

---

### 4 · AI-Assisted Follow-Up Suggestions

> Suggest personalized follow-up messages based on visitor form responses, visit frequency, and notes.

**Estimated phase:** Phase 5+ — requires stable data pipeline and LLM integration

---

### 5 · Referral Tracking Program

> Track which existing visitors referred new visitors. Reward or recognize top referrers.

**Estimated phase:** Phase 4

---

### 6 · Advanced Reporting & Scheduled Email Digests

> Automated weekly/monthly PDF reports sent to admin email with visitor trends, follow-up completion rates, and growth metrics.

**Estimated phase:** Phase 3

---

### 7 · White-Label / Custom Branding (Enterprise)

> Custom domain, logo, color theme, and branded email templates for enterprise-tier customers.

**Estimated phase:** Phase 3

---

*This section should be reviewed and reprioritized at each planning cycle.*