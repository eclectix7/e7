# E7 Guest System — Flow Chart

> Self-contained rendered version: `E7GuestSystem-FlowChart.html`
> Mermaid source below for version control. Render at [mermaid.live](https://mermaid.live).

---

## 1 · Visitor Check-In Flow (Web / Public)

```mermaid
flowchart TD
    A([Visitor arrives at location]) --> B{Has QR code<br>or shared link?}
    B -- Scans QR<br>or opens link --> C[Land on<br>visitor form page]
    C --> D[Fill out form<br>• Name(s)<br>• Party size<br>• Local / Moving<br>• Friends/Family in church<br>• Reason for visit<br>• Prayer request / Follow-up opt-in]
    D --> E{Opted in to<br>SMS follow-up?}

    E -- Yes --> F[System sends automated<br>SMS welcome message<br>via Twilio<br>using admin template]
    E -- No --> G[Record visitor only<br>No SMS sent]

    F --> H[Log visitor registration<br>in database<br>• Status: new<br>• Timestamp<br>• IP + consent record]
    G --> H

    H --> I[Push notification sent<br>to account admin / staff]
    I --> J[Visitor lands on<br>Thank You screen]
    J --> K([End])

    style C fill:#0d4429,stroke:#2e7d32,color:#e6edf3
    style F fill:#0c2d57,stroke:#1565c0,color:#e6edf3
    style G fill:#4e3b00,stroke:#f9a825,color:#e6edf3
    style H fill:#3b1a52,stroke:#7b1fa2,color:#e6edf3
    style I fill:#0c2d57,stroke:#1565c0,color:#e6edf3
    style J fill:#0d4429,stroke:#2e7d32,color:#e6edf3
```

---

## 2 · Account Admin Onboarding Flow

```mermaid
flowchart TD
    A([Admin signs up]) --> B{Auth method}
    B -- Email + Password --> C[Convex email/password auth]
    B -- Google OAuth --> D[Convex Google provider]
    B -- Apple OAuth --> E[Convex Apple custom provider]

    C --> F[Account created<br>• Account ID<br>• Stripe customer<br>• Default: Starter tier]
    D --> F
    E --> F

    F --> G[Admin sets up organization<br>• Church or business name<br>• Timezone<br>• Notification preferences]

    G --> H[Create Location]
    H --> I[Configure form fields<br>• Standard templates<br>• Enable/disable fields<br>• Set required or optional]
    I --> J[Set admin SMS reply template<br>• Max 140 chars<br>• Variables: visitor name, church name, etc.]
    J --> K[Generate QR code<br>• Signed short-lived URL<br>• Download PNG / print-ready PDF]
    K --> L[Share link or print QR<br>for visitor check-in]

    L --> M([Location live and<br>ready for visitors])

    style F fill:#0c2d57,stroke:#1565c0,color:#e6edf3
    style H fill:#0d4429,stroke:#2e7d32,color:#e6edf3
    style I fill:#4e3b00,stroke:#ef6c00,color:#e6edf3
    style J fill:#4e3b00,stroke:#ef6c00,color:#e6edf3
    style K fill:#4e3b00,stroke:#ef6c00,color:#e6edf3
    style L fill:#0d4429,stroke:#2e7d32,color:#e6edf3
    style M fill:#1a3a1f,stroke:#388e3c,color:#e6edf3
```

---

## 3 · Staff Follow-Up Flow (App / Web Dashboard)

```mermaid
flowchart TD
    subgraph Trigger
        A[Push notification<br>New visitor registered] --> B[Staff opens<br>app or web dashboard]
    end

    B --> C{Action}
    C --> D[View visitor list<br>• Activity log<br>• Filter by date / location / status]
    C --> E[View visitor profile<br>• Full form responses<br>• Notes from other staff]
    C --> F[Start message thread<br>• SMS to visitor via Twilio<br>• In-app chat if visitor responds]

    D --> G([Review complete])
    E --> H[Add notes / update<br>contact info and status]
    F --> I{Visitor responds?}
    I -- Yes --> J[Continue conversation<br>in message thread]
    I -- No --> K[Mark as contacted<br>Set follow-up reminder]

    H --> L{Set reminder?}
    L -- Yes --> M[Schedule reminder<br>• Per visitor or general<br>• Push / email / SMS channel]
    L -- No --> G

    M --> N(Reminder fires at<br>scheduled time)
    N --> O[Staff sees reminder<br>in dashboard / app]
    O --> P{Take action?}
    P -- Yes --> F
    P -- No --> Q[Dismiss / mark complete]

    Q --> R([End])

    style A fill:#0c2d57,stroke:#1565c0,color:#e6edf3
    style B fill:#0d3320,stroke:#388e3c,color:#e6edf3
    style C fill:#4e3b00,stroke:#f9a825,color:#e6edf3
    style F fill:#0d3320,stroke:#388e3c,color:#e6edf3
    style H fill:#3b1a52,stroke:#7b1fa2,color:#e6edf3
    style M fill:#0c3440,stroke:#00838f,color:#e6edf3
    style R fill:#3d4148,stroke:#455a64,color:#e6edf3
```

---

## 4 · Inbound SMS Processing Flow

```mermaid
flowchart TD
    A([Visitor sends<br>inbound SMS]) --> B{Twilio webhook<br>receives message}
    B --> C[Parse phone number<br>Match to visitor record]
    C --> D{Message content?}

    D -- Opt-out keyword STOP<br>UNSUBSCRIBE or QUIT --> E[Update visitor status<br>Set to opted_out<br>Log opt-out event]
    E --> F[Send final acknowledgement SMS<br>You have been unsubscribed]

    D -- Normal message --> G[Create or append to<br>message thread<br>visitor to staff]
    G --> H[Push notification<br>to assigned staff]
    H --> I[Staff replies from<br>app or web dashboard]
    I --> J[Outbound SMS sent<br>via Twilio]

    D -- Unrecognized keyword --> K[Log as unknown<br>Optional: auto-reply<br>with help text]

    F --> L([End])
    J --> L
    K --> L

    style A fill:#0c2d57,stroke:#1565c0,color:#e6edf3
    style E fill:#3b0a0a,stroke:#c62828,color:#e6edf3
    style F fill:#3b0a0a,stroke:#c62828,color:#e6edf3
    style G fill:#0d3320,stroke:#388e3c,color:#e6edf3
    style H fill:#0c2d57,stroke:#1565c0,color:#e6edf3
    style J fill:#0c2d57,stroke:#1565c0,color:#e6edf3
    style K fill:#4e3b00,stroke:#ef6c00,color:#e6edf3
```

---

## 5 · System Architecture Overview

```mermaid
flowchart LR
    subgraph Frontend
        F1[Vite plus React PWA<br>Visitor forms and admin dashboard]
        F2[Expo App<br>Staff mobile CRM and messaging]
    end

    subgraph Backend
        B1[Convex<br>Auth, DB, Server functions]
        B2[Twilio<br>SMS send, receive, webhooks]
        B3[Resend<br>Transactional email]
        B4[Stripe<br>Subscription billing]
    end

    subgraph Infrastructure
        I1[Netlify<br>Frontend hosting and CI/CD]
        I2[GitHub Actions<br>Lint, test, deploy]
        I3[Sentry<br>Error tracking]
        I4[PostHog<br>Analytics]
    end

    F1 -- API calls --> B1
    F2 -- API calls --> B1
    B1 -- SMS send --> B2
    B2 -- Webhooks --> B1
    B1 -- Email send --> B3
    B1 -- Billing --> B4
    B1 -- Deploy --> I1
    B1 -- CI/CD --> I2
    B1 -- Error logs --> I3
    F1 and F2 -- Usage data --> I4

    style F1 fill:#0d3320,stroke:#27ae60,color:#e6edf3
    style F2 fill:#0d3320,stroke:#27ae60,color:#e6edf3
    style B1 fill:#0c2d57,stroke:#2980b9,color:#e6edf3
    style B2 fill:#4e3b00,stroke:#e67e22,color:#e6edf3
    style B3 fill:#4e3b00,stroke:#e67e22,color:#e6edf3
    style B4 fill:#4e3b00,stroke:#e67e22,color:#e6edf3
    style I1 fill:#1a1a40,stroke:#8e44ad,color:#e6edf3
    style I2 fill:#1a1a40,stroke:#8e44ad,color:#e6edf3
    style I3 fill:#1a1a40,stroke:#8e44ad,color:#e6edf3
    style I4 fill:#1a1a40,stroke:#8e44ad,color:#e6edf3
```

---

*Generated for E7 Guest System -- 2026*