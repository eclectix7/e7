# Hermes Agent — Secure Installation Proposal for MacBook

**Created:** 2026-05-26
**Purpose:** Install Hermes on the new MacBook with security constraints that maximize utility while preventing harm
**Cross-ref:** Tasks requiring a running agent are marked `[PHASE X]` with references to `iOS-Deployment-Checklist.md`

---

## Design Principles

1. **Hermes is an assistant, not an admin.** It should not have the ability to bypass system security, escalate privileges, or access data outside its designated scope.
2. **Principle of least privilege.** Every permission is granted explicitly and for a defined purpose.
3. **Auditability.** All agent actions in terminal/file system should be traceable.
4. **Compartmentalization.** The agent operates in a constrained environment separate from signing credentials and production secrets.

---

## 1. Account & Access Architecture

### 1.1 macOS User Separation

| Account | Role | Hermes Access |
|---------|------|---------------|
| `admin` | System admin, installs, security approval | None (manual use only) |
| `dev` | Daily development, coding, debugging | **Hermes runs here** |
| `deploy` (optional) | Code signing, App Store submissions | **No agent access** |

- Hermes runs under the `dev` account only
- The `deploy` account holds signing certificates and Apple ID credentials
- `dev` → `deploy` escalation requires manual `su` or Fast User Switching (HITL)
- Cross-ref: `iOS-Deployment-Checklist.md §1.1` (non-admin dev account)

### 1.2 Apple ID Separation

| Apple ID | Purpose | Hermes Access |
|----------|---------|---------------|
| Personal iCloud (existing) | iCloud, iMessage, personal services, previously used for developer.apple.com | None for Hermes |
| E7 developer (existing or new — see decision in §0) | Xcode signing, App Store Connect | **Read-only via Xcode, no direct access** |
| Expo/EAS | Build service | **Token in Hermes env, no Apple ID** |

- Hermes never sees Apple ID passwords
- EAS tokens stored in Hermes env config only (not in shell profiles)
- Cross-ref: `iOS-Deployment-Checklist.md §1.3` (Apple ID strategy decision)
- **HITL decision needed:** Reuse existing Apple ID for E7 org enrollment, or create a new dedicated one?

---

## 2. Installation Steps

### 2.1 Prerequisites (Must Be Done First)

- [ ] MacBook setup complete through `iOS-Deployment-Checklist.md §1` and `§2.1`
- [ ] SSH keypair for GitHub active under `dev` account
- [ ] All E7 repos cloned and building

### 2.2 Hermes Installation

```bash
# From dev account
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
# or manual install per docs
```

- [ ] Install Hermes in user space (not system-wide)
- [ ] Verify `hermes` binary is in `~/.hermes/` or `~/bin/`
- [ ] Run `hermes doctor` to verify all dependencies

### 2.3 Hermes Configuration (Critical Security Section)

**`~/.hermes/config.yaml` — Key settings:**

```yaml
agent:
  max_turns: 20              # Limit runaway agent loops
  max_iterations: 20

platforms:
  telegram:
    enabled: true
    token: ${TELEGRAM_BOT_TOKEN}  # From .env, not hardcoded
    allowed_users:
      - 8879311460               # Kaizen only
    home_channel: 8879311460

security:
  # Block dangerous commands without approval
  exec_approval: required        # Terminal commands require HITL
  file_write_approval: optional  # Can be scoped further
  network_outbound: true
```

**`~/.hermes/.env`:**

```env
TELEGRAM_BOT_TOKEN=<your-token>
TELEGRAM_ALLOWED_USERS=8879311460
TELEGRAM_HOME_CHANNEL=8879311460
HERMES_MAX_ITERATIONS=20
```

---

## 3. File System Security

### 3.1 Allowed Directories

| Directory | Access Level | Purpose |
|-----------|-------------|---------|
| `~/e7/` | **Read + Write** | All project code, docs, planning |
| `~/e7/kaizen/` | **Read + Write** | Agent's own documentation and notes |
| `~/.hermes/` | **Read + Write** | Agent config, memory, skills |
| `~/.hermes/sessions/` | **Read only** | Past session history |
| `/tmp/` | **Read + Write** | Scratch space for builds |
| `~/Library/` | **No access** | macOS system/user library |
| `/etc/`, `/usr/` | **No access** | System directories |
| `/Applications/Xcode.app/` | **Read only** | For Xcode tooling inspection |

### 3.2 Protected Directories (Explicitly Blocked)

- `~/Keychain/` or any `.keychain` files
- Cloud credential files (e.g., `~/.aws/`, `~/.config/gcloud/`)
- Apple signing certificate storage area
- Any directory containing `.p12`, `.mobileprovision`, `.cer` files
- Parent directories of the `deploy` account

**Implementation:** Use macOS sandboxing or Hermes config `deny_dirs` if available, or document as policy and audit.

---

## 4. Network Security

### 4.1 Allowed Endpoints

| Endpoint | Direction | Purpose |
|----------|-----------|---------|
| `api.telegram.org` | Outbound | Bot communication | 
| `github.com` / `api.github.com` | Outbound | Git operations, PR management |
| `api.expo.dev` / `expo.dev` | Outbound | EAS build/submit |
| `*.convex.cloud` | Outbound | Backend API calls |
| `openrouter.ai` / LLM provider | Outbound | Model inference |
| `api.openai.com` | Outbound | Image generation (if used) |
| `duckduckgo.com` / `google.com` | Outbound | Web search |
| `149.154.160.0/20` | Outbound | Telegram API IPs |

### 4.2 Blocked Endpoints

- Any endpoint not explicitly allowed
- `localhost` services on unexpected ports (prevent SSRF to internal services)
- Direct access to Apple ID / App Store Connect APIs (handled only by Xcode/CLI tools)

---

## 5. Operational Constraints

### 5.1 Commands Requiring Human Approval (HITL)

- `sudo` or any privilege escalation
- `codesign`, `security` commands (signing-related)
- `xcrun altool`, `fastlane deliver` (App Store submission)
- `eas submit` (submitting builds to App Store)
- Any `git push` to `main`/`master` branches
- `npm publish` or any public package publish
- `rm -rf` outside of `~/e7/` and `/tmp/`
- Any command targeting the `deploy` account's directories
- Installing new system packages (`brew install` requires approval first time)

### 5.2 Automatable Commands (No Approval Needed)

- `git commit`, `git branch`, `git merge` (feature branches only)
- `npm install`, `npm run`, `npm test`
- `eas build` (building — not submitting)
- `cd`, `ls`, `cat`, `grep`, standard inspection
- File reads/writes within allowed directories
- `hermes` CLI self-management
- `cd` + project navigation

### 5.3 Agent Loop Protection

```yaml
agent:
  max_turns: 20
  timeout: 300
  
# Disable kanban auto-dispatch (learned from incident)
kanban:
  dispatch_in_gateway: false    # NEVER auto-execute kanban tasks
```

- Budget cap to prevent token burn
- No recursive agent spawning
- Kanban is view/organize only (see `§5.3`)

---

## 6. Credential & Secret Management

### 6.1 What Hermes CAN Access

| Secret | Storage | Access |
|--------|---------|--------|
| Telegram bot token | `~/.hermes/.env` | Full |
| GitHub token (if any) | `~/.hermes/.env` or keychain | Read |
| EAS token | `~/.hermes/.env` | Read |
| Convex deployment URL | Project config in `~/e7/` | Full |
| Convex admin key | `~/.hermes/.env` | Use only, log redacted |

### 6.2 What Hermes CANNOT Access

| Secret | Storage | Notes |
|--------|---------|-------|
| Apple ID password | `deploy` account keychain | HITL only |
| Distribution certificates (.p12) | `deploy` account keychain | HITL only |
| Provisioning profiles | `deploy` account | Deploy process only |
| App Store Connect API key | `deploy` account | Submit process only |
| Expo password | Personal memory / 1Password | Manual login only |
| GitHub personal access token | Keychain or manual | Hermes uses SSH, not HTTPS+token |

### 6.3 Secret Rotation Policy

- Rotate Telegram bot token if leaked (regenerate via BotFather)
- Rotate GitHub SSH keys annually
- Rotate EAS tokens after any team member change
- Hermes memory never stores raw secrets (see `§7`)

---

## 7. Hermes Memory & Persistence

### 7.1 What Gets Persisted

- Project documentation and architecture notes
- Coding conventions and patterns
- Open questions and HITL decision lists
- Environment facts (OS, installed tools, paths)

### 7.2 What Must NEVER Be Persisted

- Passwords, API keys, tokens, or signing credentials
- Apple ID credentials
- Private key material
- Any data from the `deploy` account

### 7.3 Memory Redaction

- Hermes is configured with `secret_redaction: ENABLED`
- Tool output, logs, and chat responses are scrubbed
- `.env` file values are treated as sensitive and excluded from logging

---

## 8. Cross-Reference with Deployment Checklist

| Deployment Task | Phase | Hermes Can Help? | Notes |
|-----------------|-------|-------------------|-------|
| MacBook OS setup | §1 | Partial — can guide, not execute (admin tasks) | HITL for admin account actions |
| Apple Developer enrollment | §1.3 | No — org enrollment is manual | Requires legal authority + DUNS |
| Xcode + CLI tools install | §2.1 | Yes — via `brew install` after approval | First-time `brew install` needs HITL |
| GitHub SSH setup | §2.2 | Yes — keygen, config, testing | — |
| Clone repos + verify builds | §2.2 | Yes — full automation | — |
| EAS configuration | §2.3 | Partial — config files yes, login is HITL | — |
| Register test devices | §2.4 | Guide only — requires Xcode GUI | — |
| Certificates & provisioning | §3.1 | No — must be manual/HITL | Uses `deploy` account |
| EAS credentials | §3.2 | Guide + config file edits | Actual cert upload may need HITL |
| TestFlight submission | §3.3 | Partial — `eas submit` needs approval | HITL gate before submission |
| App Review submission | §3.4 | No — App Store Connect web UI | — |
| Hermes install on Mac | §2 (this doc) | Once installed, self-managing | Bootstrap manually |
| Local LLM setup (Ollama) | §11 (this doc) | Yes — `brew install ollama`, model pulls | After Hermes install; add as secondary provider |

---

## 9. Ongoing Monitoring

- Weekly review of `~/.hermes/logs/gateway.log` for anomalies
- Monitor token usage (Hermes dashboard or OpenRouter billing)
- Audit `~/.hermes/memory/` quarterly for stale or sensitive data
- Verify `~/.hermes/.env` permissions are `600` (owner-read only)
- If running local Ollama: monitor disk space (models are 5-20GB each) and update models quarterly

---

## 10. Bootstrapping Order (Do This First)

1. Complete MacBook setup through `iOS-Deployment-Checklist.md §1`
2. Complete dev tool installs through `§2.1`
3. Clone repos and verify builds (`§2.2`)
4. **Then** install Hermes per `§2.2` of this document
5. Configure Hermes security per `§2.3` and `§3`
6. Test Telegram connectivity from MacBook
7. Hand off remaining deployment tasks with this doc as the constraint reference

---

## 11. Local LLM Analysis for M5 Max (36GB)

### 11.1 What Fits

At 36GB unified memory, the practical model tiers are:

| Model Size | Quantization | RAM Required | Fits? | Est. Speed | Use Case |
|------------|-------------|--------------|-------|------------|----------|
| 7B (e.g. Qwen3-7B) | Q4_K_M | ~4.5GB | Yes, comfortable | 50-80 tok/s | Fast coding assistant, autocomplete |
| 14B (e.g. Qwen3-14B) | Q4_K_M | ~8.5GB | Yes, comfortable | 30-50 tok/s | Quality coding + reasoning |
| 30B (e.g. Qwen3-30B) | Q4_K_M | ~17GB | Yes, moderate | 15-25 tok/s | Near-frontier quality locally |
| 34B (e.g. Qwen2.5-32B) | Q4_K_M | ~19GB | Yes, moderate | 12-20 tok/s | Good reasoning, coding |
| 70B (e.g. Llama 3.3 70B) | Q4_K_M | ~38GB | **No** (exceeds 36GB) | N/A | — |
| 70B | Q2_K | ~22GB | Yes, tight | 5-10 tok/s | Lossy, not recommended |

**Sweet spot for 36GB: 14B-30B models at Q4_K_M.**

### 11.2 Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| [Ollama](https://ollama.com) | Model manager + local API server | Easiest setup, `ollama pull`, `ollama run` |
| [LM Studio](https://lmstudio.ai) | GUI model manager + local server | Good for experimentation |
| [llama.cpp](https://github.com/ggerganov/llama.cpp) | Backend runtime | Used by both above; Metal GPU acceleration built in |

**Recommended setup:**
```bash
brew install ollama
ollama pull qwen3:14b-q4_k_m    # Primary local model
ollama pull qwen3:7b-q4_k_m     # Fast model for quick tasks
```

Ollama exposes an OpenAI-compatible API at `http://localhost:11434/v1`, which Hermes can use as a model provider.

### 11.3 Should Hermes Run Locally or in the Cloud?

| Factor | Local (M5 Max) | Cloud (OpenRouter/etc) |
|--------|----------------|----------------------|
| **Cost** | Free after hardware; 25-45W power | ~$5-15/1M tokens; adds up with agent tool calls |
| **Speed** | 15-50 tok/s (good for coding) | 50-200+ tok/s (provider dependent) |
| **Quality** | 14B-30B class (~GPT-3.5 tier) | GPT-4o/Claude class (significantly better reasoning) |
| **Offline** | Works without internet | Requires internet |
| **Privacy** | All data stays on-device | Data sent to provider |
| **Tool calling** | Weak-medium on local models | Reliable, mature |
| **Context window** | 32K-128K (model dependent) | 128K-1M+ |
| **Agent loops** | May struggle with multi-step reasoning | Reliable |

**Recommendation for E7:**

- **Hermes primary:** Keep using cloud providers (OpenRouter) for now. The quality difference for multi-step agent work (planning, code review, architecture decisions) is significant.
- **Local LLM as supplementary:** Perfect for quick coding tasks, offline work, privacy-sensitive code reviews, and situations where internet is unreliable.
- **Hybrid approach:** Use Ollama as a secondary Hermes provider. Route quick/cheap tasks to local, complex reasoning to cloud.

### 11.4 If You Upgrade RAM Later

If you upgrade to 64GB or 128GB in a future machine:

| RAM | What becomes viable |
|-----|-------------------|
| 48GB | 34B Q4 models comfortably, 70B at Q3 |
| 64GB | 70B Q4 models at 15-20 tok/s |
| 128GB | 70B Q8 / 100B+ class models |

**Don't buy more RAM now for LLM purposes.** 36GB is a solid starting point, and the cloud quality gap justifies keeping complex work cloud-bound for now. Re-evaluate in 12-18 months when 70B-class quality comes to smaller models.
