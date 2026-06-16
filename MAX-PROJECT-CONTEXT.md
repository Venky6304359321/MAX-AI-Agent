# MAX — AI Project Context File
> **HOW TO USE THIS FILE:**
> At the start of every new Copilot / AI chat session, say:
> *"Read MAX-PROJECT-CONTEXT.md first. Then I will tell you today's issue only."*
> The AI will understand the full project instantly. You never repeat history.
> **Update this file whenever a major decision changes.**

---

## 1. What MAX Is (One Line)
MAX is a **24x7 AI Employee** — it monitors servers and applications, raises voice alerts, explains root causes with confidence scores, and never takes a production action without the owner's explicit approval.

---

## 2. The Two Pillars (Core Job of MAX)

### Pillar 1 — Server Checkup AI
- Monitors every server: CPU, memory, disk, uptime, SSL certificate expiry, domain health
- Servers: Ionos VMs, Plesk-managed hosts, any other Linux/Windows servers
- If a server is down, SSL is expiring in < 14 days, or disk > 90%: raises voice alert + toast notification
- Owner approves any action (restart, scale, renew cert) — MAX never acts alone

### Pillar 2 — Projects Checkup AI
- Monitors every application: HTTP response time, error rate, DB connection health, Kafka lag, logs
- Apps: SRJ (Recruitment), PG (Payroll), CRM, Auth Service, API Gateway, Notification Service, Email Gateway, File Storage, Report Engine, Interview Scheduler, Resume Parser, Analytics (12 total)
- If app is slow, erroring, or down: raises voice alert + toast with root cause + confidence score
- Owner approves any fix — MAX explains what, why, and how confident it is before asking

---

## 3. Tech Stack (Decided — Do Not Change Without Discussion)

| Layer | Technology | Notes |
|---|---|---|
| Frontend | Angular 17+ + PrimeNG 17+ | Dark Iron Man theme, responsive web + mobile |
| Backend API | ASP.NET Core 8 | JWT RBAC, Serilog logging, health checks |
| AI / Orchestration | Python FastAPI + LangGraph | Agent-based, one agent per responsibility |
| Streaming | Apache Kafka | Metrics + events from collectors |
| Database | PostgreSQL 15 + pgVector | All tables prefixed `MAX_` |
| Cache | Redis | Session + real-time state |
| Voice | OpenWakeWord + Whisper STT + Piper TTS | "Hey MAX" wake word |
| Real-time UI | SignalR (WebSocket) | Live incident + alert push |
| Observability | Prometheus + Grafana + Loki | MAX monitors itself too |
| Logging | Serilog (.NET) + structlog (Python) | Structured logs only — no Console.WriteLine |

---

## 4. Authentication (Decided)
- User types their **work email** on the login screen
- Backend validates email exists in `MAX_users` and `is_active = true`
- 6-digit OTP sent via SMTP email (SHA-256 hashed in DB, expires 5 min, max 3 attempts)
- On success: JWT issued with `role` + `approval_level` + `full_name` claims
- Angular stores JWT in **BehaviorSubject memory only** — never localStorage
- Router redirects to role-specific page based on JWT claim
- **Future-swap:** only `auth.service.ts` + `AuthController.cs` change if moving to SSO (Keycloak/Azure)

---

## 5. Roles & What They Can Do

| Role | Sees | Can Approve |
|---|---|---|
| SUPER_ADMIN | Everything — all servers + all 12 apps | L1 + L2 + L3 actions |
| DEVELOPER | Only their assigned apps | L1 on own apps |
| MARKETING | CRM data only, no infra | Nothing |
| OPERATIONS | Green/red status only | Nothing (can raise ticket) |
| EXECUTIVE | Business KPIs only | Nothing |

---

## 6. Core Rules (Non-Negotiable)
1. MAX **never** takes a production action without explicit human approval
2. Before every approved action: **snapshot saved** for 24h rollback
3. Every action logged to `MAX_audit_log` with WHO + WHEN + IP + DEVICE
4. If approver does not respond in 5 min: auto-escalate to next level
5. Every alert can be snoozed; after 3 snoozes MAX offers to raise the threshold
6. MAX always states **confidence score** before recommending a fix
7. Role enforced **server-side** on every API call — not just in the UI

---

## 7. Database — Key Tables

All tables are prefixed `MAX_`. Full schema in `MAX-Enterprise-AI-Platform.md` Section 16.

| Table | Purpose |
|---|---|
| `MAX_users` | All users — email, role, assigned apps, approval level |
| `MAX_otp_sessions` | OTP hashes (SHA-256), expiry, attempt count |
| `MAX_incidents` | Every incident — status, severity, confidence, root cause |
| `MAX_audit_log` | Every action — who, when, what, IP, device |
| `MAX_action_snapshots` | Pre-action state — kept 24h for rollback |
| `MAX_alert_thresholds` | Per-app tunable thresholds (CPU, memory, response, error rate) |
| `MAX_escalation_rules` | Who to escalate to and after how many minutes |
| `MAX_knowledge_base` | pgVector embeddings of past incidents — MAX's memory |
| `MAX_app_metrics` | Time-series metrics from collectors (partition by month) |
| `MAX_apps` | App registry — name, team owner, assigned roles |
| `MAX_server_inventory` | Server registry — hostname, IP, OS, control panel, environment |

---

## 8. Phases (Build Order)

> **Before any phase starts:** copy this file (`MAX-PROJECT-CONTEXT.md`) into the project repo root. Every AI session and every new developer reads this file first. Update Section 11 (Current Status) at the end of every sprint.

### Phase 1 — Foundation (Sprint 1–2)
**Goal:** Secure core + first working monitoring pipeline for one app + one server

- PostgreSQL schema + seed data
- ASP.NET Core: Serilog, health checks, CORS, rate limiting, error contract
- Auth: email → OTP → JWT → role redirect
- Angular login screen + role-based routing
- **Pillar 1 start:** one server collector (Ionos) → Kafka → `MAX_server_inventory` + `MAX_app_metrics`
- **Pillar 2 start:** one app collector (SRJ) → Kafka → incident creation
- Basic incident list in dashboard
- Sprint 1 exit: can login, server metrics flow in, SRJ metrics flow in, incidents visible

### Phase 2 — AI Monitoring + Intelligence (Sprint 3–5)
**Goal:** Both pillars fully AI-driven with confidence, approvals, escalation, rollback

- **Pillar 1 full:** all servers monitored — SSL expiry alerts, domain health, disk/CPU/mem
- **Pillar 2 full:** all 12 apps monitored — root cause AI, confidence score, fix proposals
- Incident approve / deny / snooze / rollback flow
- Escalation chain with live countdown timer
- Voice: "Hey MAX" → STT → response → TTS
- SignalR real-time push to Angular UI
- Alert threshold tuning (snooze 3× → offer threshold change)
- pgVector knowledge base: MAX learns from every resolved incident

### Phase 3 — Autonomous AI Employee (Sprint 6–8)
**Goal:** Business AI + mobile maturity + Teams/Slack + full automation with human gating

- Business AI: job postings, candidate search, reports, hiring predictions
- Microsoft Teams + Slack bot integration
- Mobile-optimised UX (approval on phone, voice on mobile)
- Multi-server collector automation (Ansible / containerised collectors)
- Secure secrets store (Vault / environment-based)
- Cross-app event correlation (one event in App A predicts failure in App B)
- Self-monitoring: MAX raises an incident if its own service goes down

---

## 9. Project Files (What Each File Is)

| File | Purpose |
|---|---|
| `MAX-Enterprise-AI-Platform.md` | Master spec — full architecture, DB schema, APIs, coding standards, all phases |
| `MAX-PROJECT-CONTEXT.md` | **This file** — quick AI context brief for every new chat session |
| `index.html` | UI prototype — live demo of the dashboard with email login |
| `venkatesh.html` | DEPRECATED — old name-list login — ignore |
| `server_inventory.csv` | Template for listing all servers (fill before Pillar 1 dev) |
| `MULTI_SERVER_PILOT.md` | Pilot plan for testing 2 apps on 2 servers |
| `README.md` | Developer quickstart and share package list |
| `MAX-AI-Platform-Phases-Professional-Final.pptx` | Stakeholder slide deck |

---

## 10. Coding Standards (Summary — Full Detail in Section 28 of Main MD)

| Rule | Value |
|---|---|
| Logging | Serilog (.NET) + structlog (Python) — structured, no string concat |
| DB table names | `MAX_` prefix + `snake_case` |
| Controller names | `PascalCase` + `Controller` suffix |
| Angular components | `kebab-case` folder, `PascalCase` class + `Component` suffix |
| Python files | `snake_case.py`, classes `PascalCase` |
| Secrets | Environment variables only — never in code or git |
| Error responses | Standard JSON shape: `{ success, errorCode, message, timestamp }` |
| Async | All I/O must be async/await — no `.Result` or `.Wait()` |
| JWT storage | BehaviorSubject memory only — never localStorage |
| Git commits | `type(scope): description` — e.g. `feat(auth): add OTP flow` |

---

## 11. Current Status (Update This Every Sprint)

| Item | Status |
|---|---|
| Full architecture design | ✅ Complete |
| DB schema | ✅ Complete |
| API endpoints defined | ✅ Complete |
| Coding standards defined | ✅ Complete |
| UI prototype (`index.html`) | ✅ Complete — email login + dashboard demo |
| Pre-dev review (gaps fixed) | ✅ Complete — v3.0 |
| Server inventory template | ✅ Created |
| Sprint 1 development | 🔲 Not started |
| Pillar 1 (server monitoring) | 🔲 Not started |
| Pillar 2 (app monitoring) | 🔲 Not started |

---

## 12. How to Use This File in a New Chat

**Paste this at the start of every new session:**

```
I am building the MAX Enterprise AI Platform.
Read MAX-PROJECT-CONTEXT.md — it has everything: vision, tech stack, 
two pillars, roles, phases, DB tables, coding standards, and current status.
Today's issue is: [DESCRIBE ONLY THE NEW ISSUE HERE]
```

That's it. The AI has the full picture. You only describe what's new.
