# MAX Enterprise AI Platform
## Complete Technical Blueprint — Build-Ready Design Document
**Version:** 3.0 — June 2026 | **Author:** Venkatesh N (Super Admin / Tech Lead)
**Status:** Pre-Development Review Complete — Ready for Sprint 1

---

## 1. Vision

MAX is **not a monitoring dashboard**.

MAX is a **24x7 AI Employee** — modeled exactly on Iron Man's MAX:

| Iron Man MAX | Your MAX |
|---|---|
| Always watching | 24x7 monitoring all apps |
| Speaks proactively | Alerts without being asked |
| Answers instantly | Voice/chat Q&A under 2 seconds |
| Never acts without Tony's word | Never touches production without approval |
| Remembers everything | pgVector knowledge base grows forever |
| No screens needed | Employees just talk |

> **One Line:** MAX knows everything, tells you everything, but does nothing without your word.

---

## 2. Core Principles (Non-Negotiable)

1. **Permission First** — MAX never takes a production action without explicit human approval
2. **Role Isolation** — each user sees and can ask only what their role allows
3. **Confidence Transparency** — MAX always states confidence level before recommending a fix
4. **Full Audit Trail** — every login, approval, denial, rollback logged with WHO + WHEN + DEVICE + IP
5. **Escalation Chain** — if approver is unavailable, incident escalates automatically after 5 min
6. **Rollback Ready** — before every approved action, MAX snapshots state for 24h rollback
7. **False Positive Tuning** — every alert can be snoozed; after 3 snoozes MAX offers threshold change

---

## 3. How Users Interact

Every employee has a MAX widget embedded in their browser (Angular + PrimeNG SPA).

### Interaction Channels
- **Voice** — "Hey MAX" wake word -> Whisper STT -> LLM -> Piper TTS response
- **Chat** — text input in MAX sidebar panel
- **Microsoft Teams / Slack bot** (Phase 3)
- **Email notifications** (triggered by MAX)
- **Mobile App** — same Angular app, responsive layout
- **Proactive Toasts** — MAX pushes p-toast notifications without being asked

### Voice Wake Word Flow
```
User says "Hey MAX"
      |
OpenWakeWord detects wake phrase
      |
Microphone activates (was NOT listening before)
      |
Whisper (speech-to-text) transcribes
      |
LangGraph orchestrator classifies intent
      |
Planner selects agent + tools
      |
If action needed -> asks permission
      |
On approval -> executes
      |
Piper TTS speaks response
      |
Microphone deactivates
```

### Example Voice Conversation
```
User:   "Hey MAX, why is SRJ slow today?"
MAX: "I am 94% confident the root cause is a missing index on
         CandidateSkills. PostgreSQL response increased 250%.
         Recommended fix: create the index. Should I do it?"
User:   "Yes"
MAX: "Index created. SRJ response time back to 95ms.
         Rollback snapshot saved. Audit log updated."
```

---

## 4. Confidence Score System

Before any fix, MAX states its confidence:

| Confidence | Range | What MAX Says |
|---|---|---|
| High | 85-100% | "94% confident. Same pattern as INC-1988. Safe to approve." |
| Medium | 50-84% | "67% confident. Two causes identified. Review logs first." |
| Low | < 50% | "41% confident. Root cause unclear. Manual investigation recommended." |

**Why:** Prevents MAX from recommending wrong fixes with false certainty.

Confidence calculation (Python):
```python
def calculate_confidence(event, knowledge_hits, pattern_match):
    base = 0.5
    if knowledge_hits > 0:       base += 0.25  # seen before in KB
    if pattern_match > 0.8:      base += 0.15  # strong historical match
    if event.error_code_known:   base += 0.10  # known error code
    return min(base, 1.0)
```

---

## 5. Permission Model

### Action Levels
| Level | Example Action | Who Can Approve |
|---|---|---|
| L0 | Read logs, show metrics, answer questions | Any authenticated user |
| L1 | Restart service, clear cache, create index | DEVELOPER (own apps) or SUPER_ADMIN |
| L2 | Scale infra, modify DB schema, change config | SUPER_ADMIN only |
| L3 | Delete data, drop table, purge queue | SUPER_ADMIN + typed confirmation |

### Decision Tree
```
MAX detects problem
      |
Root cause analysis -> Confidence score calculated
      |
MAX notifies + states confidence + suggests fix
      |
  APPROVE -> Snapshot -> Execute -> Health Check -> Log to audit
  DENY    -> Log denial -> Keep monitoring -> Raise ticket
  SNOOZE  -> Snooze 15 min -> Re-raise at timer -> Tune if 3x snoozed
```

---

## 6. Role-Based Access Control

### Access Matrix
| Role | Apps Visible | Approve | Ask MAX About |
|---|---|---|---|
| SUPER_ADMIN | All 12 | L1 + L2 + L3 | Everything |
| DEVELOPER | Assigned apps only | L1 own apps | Own apps only |
| MARKETING | None (infra hidden) | None | CRM + campaigns |
| OPERATIONS | Status only (green/red) | None | Status + raise ticket |
| EXECUTIVE | None (infra hidden) | None | Business KPIs only |

### Out-of-Role Refusal Examples
```
Marketing -> "Show server logs"
MAX: "You don't have access to technical data.
         I can check if CRM is working — would that help?"

Developer -> "Restart production DB"
MAX: "That requires Super Admin approval.
         Alert sent to Venkatesh. Want me to raise a ticket?"
```

---

## 7. Escalation Chain

If no approval within 5 minutes of a CRITICAL incident:

```
Critical incident raised
      |
Notification sent to Primary Approver (Venkatesh N)
      |
5:00 countdown shown on screen
      |
No response in 5 min
      |
Escalation Level 1 -> Backup Admin notified + sound alert
      |
No response in 5 more min
      |
Escalation Level 2 -> Manager / CEO notified
      |
Still no response
      |
Auto-raise high-priority ticket + attempt L1 safe actions only
```

DB Table:
```sql
CREATE TABLE MAX_escalation_rules (
  id                SERIAL PRIMARY KEY,
  incident_severity VARCHAR(20) NOT NULL,
  wait_minutes      INTEGER     NOT NULL DEFAULT 5,
  escalate_to_user  INTEGER     REFERENCES MAX_users(id),
  escalate_to_role  VARCHAR(20),
  created_at        TIMESTAMP   DEFAULT NOW()
);
```

---

## 8. Rollback System

```
Approval received
      |
Pre-action snapshot saved to MAX_action_snapshots:
  - Service config
  - Container image version
  - DB state (for schema changes)
  - Environment variables
      |
Action executed
      |
Health check run (30 seconds)
      |
PASS -> Snapshot kept 24h then auto-purged
FAIL -> MAX alerts immediately + offers instant rollback
```

DB Table:
```sql
CREATE TABLE MAX_action_snapshots (
  id              SERIAL PRIMARY KEY,
  incident_id     INTEGER NOT NULL REFERENCES MAX_incidents(id),
  action_desc     TEXT    NOT NULL,
  snapshot_data   JSONB   NOT NULL,
  taken_at        TIMESTAMP DEFAULT NOW(),
  expires_at      TIMESTAMP,
  rolled_back     BOOLEAN   DEFAULT FALSE,
  rolled_back_by  INTEGER   REFERENCES MAX_users(id),
  rolled_back_at  TIMESTAMP
);
```

---

## 9. Alert Threshold Tuning (Anti-False-Positive)

```
User snoozes same alert 3 times
      |
MAX: "You have snoozed this memory alert 3 times.
         Current threshold: 80%. Want to raise it to 90%?"
      |
User approves -> saved to DB -> MAX recalibrates for that app
```

DB Table:
```sql
CREATE TABLE MAX_alert_thresholds (
  id            SERIAL PRIMARY KEY,
  app_id        INTEGER     REFERENCES MAX_apps(id),  -- NULL = global default
  metric_type   VARCHAR(50) NOT NULL
                CHECK (metric_type IN ('CPU','MEMORY','RESPONSE_TIME','ERROR_RATE')),
  severity      VARCHAR(20) NOT NULL,
  threshold     NUMERIC(8,2) NOT NULL,
  snoozed_count INTEGER     DEFAULT 0,
  updated_by    INTEGER     REFERENCES MAX_users(id),
  updated_at    TIMESTAMP   DEFAULT NOW()
);
```

---

## 10. Phase 1 — AI Monitoring Architecture

### Data Flow
```
App1  -> Collector1 --|
App2  -> Collector2   |
...                   |-> Kafka -> Priority Scheduler -> AI Workers
App12 -> Collector12--|
```

### Collector (Zero Intelligence — Sidecar per App)
Pushes every 5-10 seconds:
- CPU, Memory, Disk usage
- HTTP response times
- Application logs (structured JSON)
- DB connection pool usage
- Exception / error events
- Active user counts

### Priority Scheduler
```
CRITICAL -> first | HIGH -> second | MEDIUM -> third | LOW -> last
Critical always preempts. Low waits until all critical + high are cleared.
```

### AI Worker (All Intelligence — Central Pool)
- Root Cause Analysis
- Confidence Score calculation
- Failure Prediction (ML models on metric trends)
- Event Correlation (cross-app patterns)
- Knowledge Base lookup via pgVector
- Notification dispatch (voice + toast + email + Teams)
- Action proposal with permission request

---

## 11. Phase 2 — Business AI Assistant

```
Voice or chat request received
      |
Intent detection (LangGraph)
      |
Knowledge search (previous postings, policies, templates)
      |
API Planner decides which APIs to call
      |
Confirms with user before executing
      |
Executes + reports result + logs to audit
```

### Supported Business Operations
- Create and publish job postings (from templates)
- Search and rank candidates by match score
- Send interview invitation emails
- Schedule interviews
- Generate weekly/monthly reports
- Answer any business question
- Predict next month's hiring needs

---

## 12. Complete Architecture

```
Angular + PrimeNG SPA (Browser / Mobile)
      |
ASP.NET Core Web API  <-- JWT (role-based, all endpoints protected)
      |
LangGraph Orchestrator (Python FastAPI)
  |-- MonitoringAgent    BusinessAgent    NotificationAgent
  |-- AutoHealAgent      KnowledgeAgent   ChatAgent
  |-- EscalationAgent    RollbackAgent    ThresholdAgent
      |                              |
Kafka (streaming)           PostgreSQL + pgVector + Redis
      |
Priority Scheduler -> AI Workers -> Root Cause Engine
      |
Applications (12+) with Lightweight Collectors
```

---

## 13. Frontend — Angular + PrimeNG

### Tech Stack
| Layer | Technology |
|---|---|
| Framework | Angular 17+ |
| UI Library | PrimeNG 17+ |
| Theme | Custom Iron Man dark (CSS variables) |
| Icons | PrimeIcons + FontAwesome |
| Charts | p-chart (Chart.js) |
| Real-time | Angular WebSocket via RxJS / SignalR |
| State | NgRx or BehaviorSubject services |
| Auth | JWT in memory (BehaviorSubject — never localStorage) |
| Voice | Web Speech API + OpenWakeWord WebSocket |
| Sounds | AudioService with 6 sound types |
| Fonts | Orbitron (MAX logo) + Rajdhani (body) |

### Folder Structure
```
MAX-ui/src/app/
|-- auth/
|   |-- login/                   <- 3-step screen (Step1: Email, Step2: OTP, Step3: Redirect)
|   |-- auth.guard.ts
|   `-- auth.service.ts
|-- core/
|   |-- MAX.service.ts        <- all API calls
|   |-- websocket.service.ts     <- real-time events from SignalR
|   |-- voice.service.ts         <- wake word + STT + TTS
|   |-- audio.service.ts         <- 6 alert sounds
|   `-- toast.service.ts         <- proactive PrimeNG p-toast notifications
|-- layout/
|   |-- topnav/
|   |-- sidebar/
|   `-- chat-panel/              <- MAX chat sidebar (all roles)
|-- dashboard/                   <- admin full dashboard
|-- incidents/                   <- incidents + approve/deny/snooze/rollback
|-- apps/                        <- 12 app health cards + metrics
|-- business/                    <- business AI tab
|-- admin/                       <- user management + full audit log
|-- role-views/
|   |-- developer/               <- assigned apps only
|   |-- marketing/               <- CRM + campaigns only
|   |-- operations/              <- green/red status + raise ticket
|   `-- executive/               <- business KPIs only
`-- shared/
    |-- confidence-bar/          <- reusable confidence % component
    |-- escalation-timer/        <- live countdown component
    `-- rollback-badge/          <- rollback available badge
```

### PrimeNG Components
| UI Element | PrimeNG Component |
|---|---|
| Notifications | p-toast (severity: error/warn/info/success) |
| Incident table | p-table with row actions |
| App health cards | p-card + p-tag |
| Charts | p-chart (line + bar) |
| Sidebar nav | p-panelMenu |
| OTP input | Custom 6-box p-inputText |
| Approval dialog | p-confirmDialog |
| Audit log | p-timeline |
| Role badges | p-tag with severity |
| Loading | p-progressSpinner |
| Confidence bar | p-progressBar (dynamic color) |
| Wake modal | p-dialog with animation |
| Escalation timer | p-knob or custom countdown |

### voice.service.ts (key structure)
```typescript
@Injectable({ providedIn: 'root' })
export class VoiceService {

  // Connect to Python OpenWakeWord via WebSocket
  startWakeWordDetection(): void { ... }

  // Called when "Hey MAX" detected
  onWakeWordDetected(): void {
    this.activateMicrophone();
    this.audioService.playActivationSound();
    this.showWakeModal();
  }

  // Web Speech API (Chrome native STT fallback)
  startSpeechRecognition(): Observable<string> { ... }

  // POST to Python /ai/tts -> returns MP3 stream
  speak(text: string): Observable<void> { ... }

  deactivateMicrophone(): void { ... }
}
```

### audio.service.ts (all 6 sound types)
```typescript
@Injectable({ providedIn: 'root' })
export class AudioService {
  private sounds = {
    activation:   new Audio('/assets/sounds/MAX-activate.mp3'),   // wake acknowledged
    critical:     new Audio('/assets/sounds/critical-alert.mp3'),    // CRITICAL incident
    warning:      new Audio('/assets/sounds/warning-beep.mp3'),      // HIGH priority
    success:      new Audio('/assets/sounds/action-complete.mp3'),   // approved + done
    escalation:   new Audio('/assets/sounds/escalation-alert.mp3'),  // timer expired
    notification: new Audio('/assets/sounds/soft-notify.mp3'),       // LOW priority info
  };

  playActivationSound(): void  { this.sounds.activation.play(); }
  playCriticalAlert(): void    { this.sounds.critical.play(); }
  playWarning(): void          { this.sounds.warning.play(); }
  playSuccess(): void          { this.sounds.success.play(); }
  playEscalation(): void       { this.sounds.escalation.play(); }
  playNotification(): void     { this.sounds.notification.play(); }
}
```

### WebSocket Real-Time Event Types
```typescript
// Connects to: ws://MAX-api/hubs/events (SignalR)
interface MAXEvent {
  type: 'INCIDENT_NEW' | 'INCIDENT_UPDATE' | 'APP_STATUS_CHANGE'
      | 'PROACTIVE_ALERT' | 'ESCALATION_TRIGGERED' | 'ACTION_COMPLETE'
      | 'THRESHOLD_CROSSED' | 'HEALTH_CHECK';
  severity: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW' | 'INFO';
  payload: any;
  timestamp: string;
}
// CRITICAL         -> playCriticalAlert() + p-toast error + TTS speak
// ESCALATION       -> update timer UI + playEscalation()
// ACTION_COMPLETE  -> playSuccess() + update incident status
// HEALTH_CHECK     -> playNotification() + update app cards
```

---

## 14. Backend — ASP.NET Core Web API

### Folder Structure
```
MAXApi/
|-- Controllers/
|   |-- AuthController.cs           <- OTP send/verify, JWT issue
|   |-- UsersController.cs          <- CRUD (admin only)
|   |-- IncidentsController.cs      <- list, approve, deny, snooze, rollback
|   |-- AppsController.cs           <- health status (role-filtered)
|   |-- AuditController.cs          <- full audit log queries
|   |-- ActionsController.cs        <- execute approved action + snapshot
|   |-- BusinessController.cs       <- job postings, candidates, reports
|   |-- ThresholdsController.cs     <- tune alert thresholds
|   `-- EscalationController.cs     <- rules + active escalations
|-- Services/
|   |-- OtpService.cs               <- generate, SHA-256 hash, verify
|   |-- EmailService.cs             <- SMTP email (OTP + alerts)
|   |-- JwtService.cs               <- issue + validate JWT
|   |-- AuditService.cs             <- write audit entries
|   |-- SnapshotService.cs          <- take + restore snapshots
|   |-- EscalationService.cs        <- timer logic + escalation chain
|   |-- WebSocketHub.cs             <- SignalR hub for real-time events
|   `-- MAXAiService.cs          <- HTTP client to Python FastAPI
|-- Middleware/
|   |-- RoleAuthorizationMiddleware.cs
|   `-- AuditMiddleware.cs          <- auto-logs every authenticated request
|-- Models/
|   |-- MAXUser.cs
|   |-- OtpSession.cs
|   |-- Incident.cs
|   |-- AuditLog.cs
|   |-- ActionSnapshot.cs
|   `-- AlertThreshold.cs
`-- Program.cs
```

### All API Endpoints
```
-- AUTH --------------------------------------------------
POST   /api/auth/validate-email { email }     -> check user exists + is_active
POST   /api/auth/otp/send    { email }        -> generate + email OTP
POST   /api/auth/otp/verify  { email, otp }   -> verify -> JWT returned
POST   /api/auth/logout                       -> invalidate session

-- INCIDENTS [JWT required] ------------------------------
GET    /api/incidents                         -> list (role-filtered)
GET    /api/incidents/{id}                    -> detail + confidence score + evidence
POST   /api/incidents/{id}/approve            -> approve fix (L1/L2 level check)
POST   /api/incidents/{id}/deny               -> deny + log
POST   /api/incidents/{id}/snooze { mins }    -> snooze alert
POST   /api/incidents/{id}/rollback           -> restore snapshot

-- APPS [JWT required] -----------------------------------
GET    /api/apps                              -> health list (role-filtered)
GET    /api/apps/{name}/metrics               -> metrics + history chart data

-- AUDIT [JWT + Admin] -----------------------------------
GET    /api/audit                             -> full log with filters
GET    /api/audit/user/{id}                   -> per-user audit

-- USERS [JWT + Admin] -----------------------------------
GET    /api/users
POST   /api/users                             -> create user
PUT    /api/users/{id}
DELETE /api/users/{id}                        -> sets is_active = false

-- THRESHOLDS [JWT + Admin] ------------------------------
GET    /api/thresholds
PUT    /api/thresholds/{id}                   -> tune alert threshold + log change

-- BUSINESS [JWT required] -------------------------------
POST   /api/business/job-posting              -> create posting (confirm first)
POST   /api/business/candidate-search         -> find + rank by score
POST   /api/business/send-invitations         -> send emails + log
GET    /api/business/reports/{type}           -> weekly | monthly report

-- ESCALATION [JWT + Admin] ------------------------------
GET    /api/escalation/rules
PUT    /api/escalation/rules/{id}
GET    /api/escalation/active                 -> currently escalated incidents
```

### JWT Token Payload
```json
{
  "sub":            "1",
  "name":           "Venkatesh N",
  "role":           "SUPER_ADMIN",
  "apps":           ["ALL"],
  "approval_level": "L2",
  "iat":            1718073600,
  "exp":            1718160000
}
```

### Role-Based Authorization in .NET
```csharp
// Program.cs
builder.Services.AddAuthorization(options => {
  options.AddPolicy("AdminOnly",   p => p.RequireClaim("role", "SUPER_ADMIN"));
  options.AddPolicy("L1AndAbove",  p => p.RequireClaim("role", "SUPER_ADMIN", "DEVELOPER"));
  options.AddPolicy("AnyRole",     p => p.RequireAuthenticatedUser());
});

// On controllers:
[Authorize(Policy = "AdminOnly")]    // users, audit, thresholds
[Authorize(Policy = "L1AndAbove")]   // incident approval
[Authorize(Policy = "AnyRole")]      // view, query, chat
```

### OTP Hash in C#
```csharp
public string HashOtp(string otp) {
  using var sha = SHA256.Create();
  var bytes = sha.ComputeHash(Encoding.UTF8.GetBytes(otp));
  return Convert.ToHexString(bytes).ToLower();
}
```

---

## 15a. Engineering Standards for Every Developer
This project must be written so every developer can understand it quickly, maintain it easily, and extend it safely in the future.

### Coding Standards
- Use consistent naming conventions across frontend and backend (PascalCase for classes, camelCase for variables, kebab-case for files/components).
- Keep responsibility small: each service, controller, component, or agent should do one thing well.
- Favor explicit contracts: strong typing, clear request/response models, and well-defined API schemas.
- Document intent, not just implementation: comments should explain why a decision exists, not restate the code.
- Use consistent formatting and linting: Prettier/ESLint for frontend, EditorConfig and StyleCop/formatting rules for backend.
- Add README and design notes for every module that has a nontrivial workflow.
- Keep shared logic in reusable libraries/services so behavior is centralized and easy to change.
- Write developer-friendly error messages and log context-rich details for debugging.

### Team-Friendly Practices
- Maintain a clear folder structure with `auth`, `core`, `layout`, `dashboard`, `business`, and `admin` separation.
- Use role-based components and services so authorization logic is easy to audit.
- Prefer data-driven UI patterns and configuration objects over hard-coded values.
- Include example payloads and sample flows in documentation for onboarding new developers.
- Review code with a focus on readability and maintainability, not just correctness.

## 15b. Web + Mobile Future-Ready Design
This system should be built with a web-first implementation that is also mobile-ready from day one.

### Web and Mobile Benefits
- Responsive layout: a single Angular app can adapt from desktop to tablet to phone, reducing duplicate work.
- Shared components: using reusable UI components for cards, modals, dialogs, and notifications makes web and mobile look and behave consistently.
- Progressive enhancement: support offline-friendly behavior and lightweight screens, so the platform works on mobile browsers too.
- Future app portability: a clean separation between UI and API means the same backend can support a native mobile app later.
- Unified interaction model: voice, chat, alerts, approvals, and audit history should work the same way on web and mobile.
- Faster rollout: building for mobile readiness now avoids expensive refactors later.

### What to plan for now
- Design responsive components and grids, not fixed desktop layouts.
- Keep UI state in services so the same business logic can be reused by mobile views.
- Use API endpoints that return the right data for both dashboards and small screens.
- Build notifications and approval flows with mobile use cases in mind (touch-friendly, compact, clear actions).

## 15c. AI Service — Python FastAPI + LangGraph

### Folder Structure
```
MAX-ai/
|-- main.py
|-- orchestrator/
|   |-- langgraph_flow.py       <- state machine
|   |-- intent_classifier.py    <- what does user want?
|   `-- api_planner.py          <- which .NET APIs to call?
|-- agents/
|   |-- monitoring_agent.py
|   |-- business_agent.py
|   |-- root_cause_agent.py     <- RCA + confidence score
|   |-- knowledge_agent.py      <- pgVector semantic search
|   |-- escalation_agent.py
|   `-- rollback_agent.py
|-- tools/
|   |-- kafka_consumer.py
|   |-- redis_cache.py
|   |-- db_tools.py             <- PostgreSQL queries
|   `-- notification_tools.py
`-- models/
    |-- llm.py                  <- Ollama (local) or OpenAI config
    |-- embeddings.py           <- text-embedding for pgVector
    `-- tts.py                  <- Piper TTS or ElevenLabs
```

### LangGraph States
```
IDLE -> WAKE_DETECTED -> INTENT_CLASSIFIED -> AGENT_SELECTED
  -> ANALYSIS_DONE -> CONFIDENCE_CALCULATED -> AWAITING_APPROVAL
  -> ACTION_SNAPSHOT -> ACTION_EXECUTING -> HEALTH_CHECK -> CLOSED
```

### Key FastAPI Endpoints
```
POST /ai/analyze
  Body:    { incident_id, app_name, metrics, logs, error_codes }
  Returns: { root_cause, confidence, evidence[], recommended_action,
             action_level, rollback_possible }

POST /ai/chat
  Body:    { user_id, role, assigned_apps, message, voice: bool }
  Returns: { response_text, response_audio_url, action_required,
             action_proposal }

WS   /ai/wake-word-stream
  WebSocket: streams mic audio -> returns wake word detection events

POST /ai/tts
  Body:    { text, voice_id }
  Returns: audio/mp3 stream (Piper TTS)
```

---

## 16. PostgreSQL — Complete Database Schema

```sql
-- Enable vector extension for pgVector semantic search
CREATE EXTENSION IF NOT EXISTS vector;

-- ============================================================
-- USERS & AUTH
-- ============================================================

CREATE TABLE MAX_users (
  id              SERIAL PRIMARY KEY,
  full_name       VARCHAR(100) NOT NULL,
  email           VARCHAR(150) NOT NULL UNIQUE,
  role            VARCHAR(20)  NOT NULL
                  CHECK (role IN ('SUPER_ADMIN','DEVELOPER','MARKETING','OPERATIONS','EXECUTIVE')),
  apps_assigned   TEXT[]       DEFAULT '{}',   -- '{SRJ,AUTH_SERVICE}' or '{ALL}'
  approval_level  VARCHAR(5)   DEFAULT 'L0'
                  CHECK (approval_level IN ('L0','L1','L2')),
  is_active       BOOLEAN      DEFAULT TRUE,
  created_by      INTEGER      REFERENCES MAX_users(id),
  created_at      TIMESTAMP    DEFAULT NOW(),
  last_login      TIMESTAMP,
  last_machine    VARCHAR(100),
  last_ip         VARCHAR(50)
);

CREATE TABLE MAX_otp_sessions (
  id          SERIAL PRIMARY KEY,
  user_id     INTEGER     NOT NULL REFERENCES MAX_users(id),
  otp_hash    VARCHAR(64) NOT NULL,   -- SHA-256 hash, never plain text
  machine_id  VARCHAR(100),
  ip_address  VARCHAR(50),
  created_at  TIMESTAMP   DEFAULT NOW(),
  expires_at  TIMESTAMP   NOT NULL,   -- created_at + 5 minutes
  used        BOOLEAN     DEFAULT FALSE,
  attempts    INTEGER     DEFAULT 0   -- locked after 3
);

-- ============================================================
-- INCIDENTS
-- ============================================================

CREATE TABLE MAX_incidents (
  id                 SERIAL PRIMARY KEY,
  inc_number         VARCHAR(20) UNIQUE NOT NULL,  -- INC-2041
  app_name           VARCHAR(100) NOT NULL,
  title              TEXT NOT NULL,
  root_cause         TEXT,
  confidence         NUMERIC(5,2),                 -- 0.00 to 100.00
  severity           VARCHAR(20) NOT NULL
                     CHECK (severity IN ('CRITICAL','HIGH','MEDIUM','LOW')),
  approval_level     VARCHAR(5)  NOT NULL
                     CHECK (approval_level IN ('L0','L1','L2','L3')),
  status             VARCHAR(30) DEFAULT 'OPEN'
                     CHECK (status IN ('OPEN','AWAITING_APPROVAL','SNOOZED',
                                       'APPROVED','EXECUTING','RESOLVED',
                                       'DENIED','ESCALATED','ROLLED_BACK')),
  recommended_action TEXT,
  evidence           JSONB,           -- array of evidence strings
  snoozed_count      INTEGER   DEFAULT 0,
  snoozed_until      TIMESTAMP,
  escalated_at       TIMESTAMP,
  created_at         TIMESTAMP DEFAULT NOW(),
  resolved_at        TIMESTAMP,
  resolved_by        INTEGER   REFERENCES MAX_users(id)
);

-- ============================================================
-- ROLLBACK SNAPSHOTS
-- ============================================================

CREATE TABLE MAX_action_snapshots (
  id              SERIAL PRIMARY KEY,
  incident_id     INTEGER NOT NULL REFERENCES MAX_incidents(id),
  action_desc     TEXT    NOT NULL,
  snapshot_data   JSONB   NOT NULL,  -- { service_config, image_version, env_vars, db_state }
  taken_at        TIMESTAMP DEFAULT NOW(),
  expires_at      TIMESTAMP,         -- taken_at + 24h
  rolled_back     BOOLEAN   DEFAULT FALSE,
  rolled_back_by  INTEGER   REFERENCES MAX_users(id),
  rolled_back_at  TIMESTAMP
);

-- ============================================================
-- FULL AUDIT LOG (compliance)
-- ============================================================

CREATE TABLE MAX_audit_log (
  id                     SERIAL PRIMARY KEY,
  user_id                INTEGER      REFERENCES MAX_users(id),
  user_name              VARCHAR(100),
  action_type            VARCHAR(50)  NOT NULL
                         CHECK (action_type IN (
                           'LOGIN','LOGOUT','APPROVE','DENY','SNOOZE','ROLLBACK',
                           'QUERY','BUSINESS_ACTION','USER_CREATED','USER_DEACTIVATED',
                           'THRESHOLD_CHANGED','ESCALATION_TRIGGERED'
                         )),
  description            TEXT,
  incident_id            INTEGER      REFERENCES MAX_incidents(id),
  confidence_at_approval NUMERIC(5,2),   -- what confidence was when approved
  machine_id             VARCHAR(100),
  browser                VARCHAR(100),
  ip_address             VARCHAR(50),
  timestamp              TIMESTAMP    DEFAULT NOW()
);

-- ============================================================
-- APPS & METRICS
-- ============================================================

CREATE TABLE MAX_apps (
  id              SERIAL PRIMARY KEY,
  name            VARCHAR(100) UNIQUE NOT NULL,
  display_name    VARCHAR(100),
  team_owner      VARCHAR(100),
  assigned_roles  TEXT[],   -- which roles can see this app
  is_active       BOOLEAN DEFAULT TRUE
);

CREATE TABLE MAX_app_metrics (
  id          BIGSERIAL PRIMARY KEY,
  app_id      INTEGER   NOT NULL REFERENCES MAX_apps(id),
  cpu_pct     NUMERIC(5,2),
  mem_pct     NUMERIC(5,2),
  response_ms INTEGER,
  error_count INTEGER   DEFAULT 0,
  user_count  INTEGER   DEFAULT 0,
  recorded_at TIMESTAMP DEFAULT NOW()
);
-- Recommended: partition MAX_app_metrics by month for performance

-- ============================================================
-- ALERT THRESHOLDS (tunable per app)
-- ============================================================

CREATE TABLE MAX_alert_thresholds (
  id            SERIAL PRIMARY KEY,
  app_id        INTEGER     REFERENCES MAX_apps(id),  -- NULL = global default
  metric_type   VARCHAR(50) NOT NULL
                CHECK (metric_type IN ('CPU','MEMORY','RESPONSE_TIME','ERROR_RATE')),
  severity      VARCHAR(20) NOT NULL,
  threshold     NUMERIC(8,2) NOT NULL,
  snoozed_count INTEGER     DEFAULT 0,
  updated_by    INTEGER     REFERENCES MAX_users(id),
  updated_at    TIMESTAMP   DEFAULT NOW()
);

-- ============================================================
-- ESCALATION RULES
-- ============================================================

CREATE TABLE MAX_escalation_rules (
  id                SERIAL PRIMARY KEY,
  incident_severity VARCHAR(20) NOT NULL,
  wait_minutes      INTEGER     NOT NULL DEFAULT 5,
  escalate_to_user  INTEGER     REFERENCES MAX_users(id),
  escalate_to_role  VARCHAR(20),
  created_at        TIMESTAMP   DEFAULT NOW()
);

-- ============================================================
-- KNOWLEDGE BASE (pgVector semantic search)
-- ============================================================

CREATE TABLE MAX_knowledge_base (
  id              SERIAL PRIMARY KEY,
  incident_id     INTEGER   REFERENCES MAX_incidents(id),
  problem_summary TEXT      NOT NULL,
  root_cause      TEXT,
  solution        TEXT,
  outcome         TEXT,
  app_name        VARCHAR(100),
  tags            TEXT[],
  embedding       VECTOR(1536),      -- text-embedding-3-small or local model
  confidence_avg  NUMERIC(5,2),      -- average confidence score when this pattern used
  use_count       INTEGER   DEFAULT 1,
  created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX ON MAX_knowledge_base
  USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- ============================================================
-- BUSINESS AI ACTIONS
-- ============================================================

CREATE TABLE MAX_business_actions (
  id            SERIAL PRIMARY KEY,
  user_id       INTEGER REFERENCES MAX_users(id),
  action_type   VARCHAR(50) NOT NULL,  -- JOB_POSTING | CANDIDATE_SEARCH | INVITE_SEND | REPORT
  description   TEXT,
  payload       JSONB,
  status        VARCHAR(20) DEFAULT 'PENDING',
  result        JSONB,
  created_at    TIMESTAMP DEFAULT NOW(),
  completed_at  TIMESTAMP
);

-- ============================================================
-- SAMPLE DATA
-- ============================================================

INSERT INTO MAX_users (full_name, email, role, apps_assigned, approval_level) VALUES
  ('Venkatesh N', 'venkatesh@gmail.com', 'SUPER_ADMIN', '{ALL}',                 'L2'),
  ('Ravi Kumar',  'ravi@gmail.com',      'DEVELOPER',   '{SRJ,AUTH_SERVICE}',    'L1'),
  ('Sneha M',     'sneha@gmail.com',     'MARKETING',   '{CRM}',                 'L0'),
  ('Arun S',      'arun@gmail.com',      'OPERATIONS',  '{READ_ONLY}',           'L0'),
  ('Kiran',       'kiran@gmail.com',     'EXECUTIVE',   '{REPORTS}',             'L0');

INSERT INTO MAX_apps (name, display_name, team_owner, assigned_roles) VALUES
  ('SRJ',          'SRJ — Recruitment Platform',  'Ravi Kumar',  '{SUPER_ADMIN,DEVELOPER}'),
  ('PG',           'PG — Payroll Service',         'Venkatesh N', '{SUPER_ADMIN}'),
  ('CRM',          'CRM Platform',                 'Sneha M',     '{SUPER_ADMIN,MARKETING}'),
  ('AUTH_SERVICE', 'Auth Service',                 'Ravi Kumar',  '{SUPER_ADMIN,DEVELOPER}'),
  ('API_GATEWAY',  'API Gateway',                  'Venkatesh N', '{SUPER_ADMIN}'),
  ('NOTIFICATION', 'Notification Service',         'Venkatesh N', '{SUPER_ADMIN}'),
  ('EMAIL_GW',     'Email Gateway',                'Venkatesh N', '{SUPER_ADMIN}'),
  ('FILE_STORAGE', 'File Storage',                 'Venkatesh N', '{SUPER_ADMIN}'),
  ('REPORT_ENG',   'Report Engine',                'Venkatesh N', '{SUPER_ADMIN}'),
  ('INTERVIEWS',   'Interview Scheduler',           'Venkatesh N', '{SUPER_ADMIN}'),
  ('RESUME',       'Resume Parser',                'Ravi Kumar',  '{SUPER_ADMIN,DEVELOPER}'),
  ('ANALYTICS',    'Analytics Dashboard',          'Venkatesh N', '{SUPER_ADMIN,EXECUTIVE}');

INSERT INTO MAX_escalation_rules (incident_severity, wait_minutes, escalate_to_role) VALUES
  ('CRITICAL', 5,  'SUPER_ADMIN'),
  ('HIGH',     10, 'SUPER_ADMIN'),
  ('MEDIUM',   30, 'SUPER_ADMIN');

INSERT INTO MAX_alert_thresholds (app_id, metric_type, severity, threshold) VALUES
  (NULL, 'CPU',           'CRITICAL', 90.0),
  (NULL, 'CPU',           'HIGH',     75.0),
  (NULL, 'MEMORY',        'CRITICAL', 95.0),
  (NULL, 'MEMORY',        'HIGH',     80.0),
  (NULL, 'RESPONSE_TIME', 'CRITICAL', 2000),
  (NULL, 'RESPONSE_TIME', 'HIGH',     500),
  (NULL, 'ERROR_RATE',    'HIGH',     5.0);
```

---

## 17. Email-Based Authentication — Complete Flow

> **Design:** User types their email. Backend validates it exists in DB and is active. OTP is sent. On success, user is redirected to their role-based page. No name selection from a list.

```
User opens MAX login screen
      |
Step 1 — Enter Email
  User types their work email address
  POST /api/auth/validate-email { email }
    -> Check MAX_users WHERE email = ? AND is_active = true
    -> If not found: "Email not registered. Contact admin."
    -> If found: proceed
      |
Step 2 — OTP Sent Automatically
  POST /api/auth/otp/send { email }
    -> 6-digit OTP generated
    -> SHA-256 hashed + stored in MAX_otp_sessions (expires 5 min)
    -> Email sent via SMTP: "Your MAX OTP is 847291 — valid 5 minutes"
      |
Step 3 — Enter OTP
  User enters 6-digit OTP (auto-advance per digit, backspace navigates back)
  POST /api/auth/otp/verify { email, otp }
    -> Hash input -> compare to DB hash
    -> Check expiry (5 min) + attempt count (max 3 -> 15 min lockout)
    -> On success: JWT returned with role + approval_level + full_name claims
      |
Step 4 — Redirect to Role Page
  JWT stored in Angular BehaviorSubject (memory only — NOT localStorage)
  Router reads role claim from JWT:
    SUPER_ADMIN  -> /dashboard  (full view)
    DEVELOPER    -> /developer  (assigned apps only)
    MARKETING    -> /marketing  (CRM + campaigns only)
    OPERATIONS   -> /operations (status + raise ticket only)
    EXECUTIVE    -> /executive  (business KPIs only)
      |
All API calls include Bearer JWT header
Role enforced server-side on every request
Every action auto-logged to MAX_audit_log with device + browser + IP
```

### How to swap auth later (modular design)
The auth module is isolated in `auth/auth.service.ts` and `AuthController.cs`.
To switch to SSO (Keycloak / Azure AD / Google Workspace) in future:
- Replace `POST /api/auth/validate-email` + OTP flow with SSO redirect
- Auth service returns the same JWT shape — rest of app stays unchanged

### OTP Rules
| Rule | Value |
|---|---|
| Length | 6 digits |
| Validity | 5 minutes |
| Max wrong attempts | 3 — then 15 minute lockout |
| Storage | SHA-256 hash only — never plain text |
| Delivery | Email via SMTP (free) or WhatsApp (optional) |
| Reuse | Single-use — used = true after verification |

### Security Summary
| Threat | Protection |
|---|---|
| Guess OTP | 6 digits + 5 min expiry + 3 attempt lockout |
| Replay OTP | Single-use flag in DB |
| Stolen from DB | SHA-256 hash only — never plain text |
| Role bypass | Role enforced server-side from JWT + DB — not client |
| Employee leaves | Admin sets is_active = false — instant revoke |
| Who did what | Full audit log — every action with device + IP + browser |

---

## 18. Infrastructure

### Current Server — IONOS Linux
```
CPU:  8 cores  |  RAM: 32 GB  |  Disk: 500 GB SSD

Runs:
- Kafka + Zookeeper
- Redis 7
- PostgreSQL 15 + pgVector
- Ollama (Qwen 7B / Llama 3)
- Python FastAPI (AI service)
- ASP.NET Core 8 (API)
- Nginx + Angular SPA
- Prometheus + Grafana + Loki
```

### Docker Compose Services
```yaml
services:
  MAX-ui:        # Nginx serving Angular build — port 80/443
  MAX-api:       # ASP.NET Core 8 — port 5000
  MAX-ai:        # Python FastAPI — port 8000
  postgres:         # PostgreSQL 15 + pgVector extension — port 5432
  redis:            # Redis 7 — port 6379
  kafka:            # Apache Kafka — port 9092
  zookeeper:        # Kafka dependency — port 2181
  ollama:           # Local LLM inference — port 11434
  prometheus:       # Metrics collection — port 9090
  grafana:          # Metrics dashboards — port 3000
  loki:             # Log aggregation — port 3100
```

### Future Scaling
```
128 GB RAM + GPU   -> faster Ollama inference
Separate PG cluster with read replicas
Multi-broker Kafka cluster
Kubernetes with horizontal AI worker auto-scaling
Cloudflare CDN for Angular SPA
```

---

## 19. Technology Stack Summary

### Free / Open Source (Software Cost = Rs 0)
| Category | Technology |
|---|---|
| Frontend | Angular 17+ |
| UI Components | PrimeNG 17+ |
| Backend API | ASP.NET Core 8 |
| AI Service | Python FastAPI |
| AI Orchestration | LangGraph |
| LLM | Ollama (Qwen / Llama 3 7B) |
| Speech to Text | Whisper |
| Text to Speech | Piper TTS |
| Wake Word | OpenWakeWord |
| Database | PostgreSQL 15 |
| Vector Search | pgVector |
| Cache | Redis 7 |
| Streaming | Apache Kafka |
| Metrics | Prometheus |
| Dashboards | Grafana |
| Logs | Loki |
| Containers | Docker / Kubernetes |

### Optional Paid (Premium Quality)
| Category | Technology |
|---|---|
| Better LLM | OpenAI GPT-4o / Claude 3.5 / Gemini |
| Better TTS | ElevenLabs / Azure Neural TTS |
| WhatsApp alerts | WhatsApp Business API |
| SMS alerts | Twilio |
| Email at scale | SendGrid |

---

## 20. Phase 3 — Autonomous AI Employee (Future)

```
MAX (Master Orchestrator)
  |-- RecruitmentAgent    job postings, candidates, interviews
  |-- HRAgent             leave, payroll queries, policies
  |-- SalesAgent          CRM, pipeline, leads
  |-- FinanceAgent        revenue reports, budget queries
  |-- MonitoringAgent     all 12 apps, alerts, predictions
  |-- EscalationAgent     auto-escalate based on rules
  |-- RollbackAgent       auto-rollback on health failure
  `-- ReportingAgent      weekly/monthly auto-reports

All agents share:
  - One Knowledge Base (pgVector)
  - One Audit Log
  - One Permission Model
  - One Escalation Chain
  - One Rollback System
```

---

## 21. Voice Command Reference (Complete)

### Monitoring (role-filtered)
- "Hey MAX, how is production right now?"
- "Why is SRJ slow today?"
- "What happened during last night's outage?"
- "Is the Notification Service back up?"
- "Show me PG memory trend for the last hour"
- "Restart the Auth Service"  -> L1 approval required

### Business (SUPER_ADMIN, EXECUTIVE)
- "Create a Java Developer job posting"
- "Find React candidates with 5 years experience"
- "Send interview invitations to the top 10"
- "Generate this week's hiring report"
- "What is our revenue target progress?"
- "How many offers did we accept this month?"

### Admin (SUPER_ADMIN only)
- "Add a new user"
- "Deactivate Ravi's account"
- "Show me today's audit log"
- "Change the memory threshold for PG to 90%"
- "Who approved the DB restart last night?"
- "Rollback the index change from this morning"

---

## 22. UI Prototype Reference

File: MAX-ui/venkatesh.html (single HTML file — open in any browser, no server needed)

Demo OTP: 123456 (same for all users in prototype)

| User | Role | What They See |
|---|---|---|
| Venkatesh N | SUPER_ADMIN | All 12 apps, confidence bars, escalation timer, Approve/Deny/Snooze/Rollback, full audit with device+IP |
| Ravi Kumar | DEVELOPER | Only SRJ + Auth Service, own incidents only |
| Sneha M | MARKETING | CRM data, campaigns, no infra |
| Arun S | OPERATIONS | Green/red status only, raise ticket button |
| Kiran | EXECUTIVE | Revenue %, candidates, interviews — no tech data |

---

## 23. The Two Pillars — What MAX Actually Does

MAX has two distinct AI jobs. Both raise voice alerts. Both require approval before action. Both write to audit log.

```
MAX
├── PILLAR 1 — Server Checkup AI
│     What it watches: CPU · Memory · Disk · Uptime · SSL expiry · Domain health
│     Where: Ionos VMs · Plesk hosts · any Linux/Windows server
│     Alerts on: server down · SSL expiring < 14 days · disk > 90% · uptime drop
│     Action: voice alert → owner approves → MAX acts (restart / scale / renew)
│
└── PILLAR 2 — Projects Checkup AI
      What it watches: HTTP response · error rate · DB health · Kafka lag · logs
      Where: SRJ · PG · CRM · Auth · API Gateway · Notification · Email GW ·
             File Storage · Report Engine · Interviews · Resume · Analytics (12 apps)
      Alerts on: app slow · error spike · app down · DB connection exhausted
      Action: voice alert + confidence score → owner approves → MAX acts + snapshots
```

Both pillars feed the same incident pipeline → same approval flow → same audit log → same rollback system.

---

## 24. Implementation Roadmap — By Pillar and Phase

> ⚠️ **Before writing a single line of code — read Section 24.0 first.**

### 24.0 — First Step for Every New Project / Every New Developer

**This is mandatory. Do this before opening any IDE.**

1. Copy `MAX-PROJECT-CONTEXT.md` into the root of the new project repository.
2. When starting any AI-assisted coding session (Copilot, ChatGPT, Claude, etc.), paste this at the top of the chat:
   ```
   I am building the MAX Enterprise AI Platform.
   Read MAX-PROJECT-CONTEXT.md first — it has the full vision, tech stack,
   two pillars, roles, phases, DB tables, and coding standards.
   Today's task is: [describe only the new issue here]
   ```
3. Update `MAX-PROJECT-CONTEXT.md` → Section 11 (Current Status) at the end of every sprint.
4. New developer joining the team? Hand them `MAX-PROJECT-CONTEXT.md` + `MAX-Enterprise-AI-Platform.md`. They read those two files and they know everything.

> **Why:** This is a POC now, but it will become a real production system. Every session, every developer, every AI tool must start from the same shared knowledge — not from memory or guessing.

---

### Phase 1 — Foundation (Sprint 1–2, Weeks 1–2)
**Exit criteria:** can login, one server metric flows in, one app metric flows in, incidents visible in dashboard.

| # | Task | Pillar |
|---|---|---|
| 1 | PostgreSQL schema + seed data | Both |
| 2 | ASP.NET Core setup: Serilog, health checks, CORS, rate limits, error contract | Both |
| 3 | Auth: validate email → OTP → JWT → role redirect | Both |
| 4 | Angular login screen (email input → OTP boxes → role page) | Both |
| 5 | **Pillar 1:** Server collector for first Ionos server → Kafka → `MAX_app_metrics` | Pillar 1 |
| 6 | **Pillar 1:** SSL expiry check + domain uptime check (first server) | Pillar 1 |
| 7 | **Pillar 2:** App collector for SRJ → Kafka → `MAX_app_metrics` | Pillar 2 |
| 8 | Basic incident creation from collector data | Both |
| 9 | Angular dashboard — incidents list (no AI yet, rule-based threshold) | Both |

### Phase 2 — AI Monitoring + Intelligence (Sprint 3–5, Weeks 3–6)
**Exit criteria:** both pillars fully AI-driven, voice works, approval + rollback works, all 12 apps + all servers monitored.

| # | Task | Pillar |
|---|---|---|
| 10 | **Pillar 1 full:** all servers onboarded — SSL · disk · uptime · CPU · memory alerts | Pillar 1 |
| 11 | **Pillar 2 full:** all 12 apps onboarded — response · errors · DB · logs | Pillar 2 |
| 12 | Python FastAPI AI: root cause analysis + confidence score engine | Both |
| 13 | Incident approve / deny / snooze / rollback flow | Both |
| 14 | Escalation chain + live countdown timer | Both |
| 15 | SignalR real-time push to Angular (incidents, alerts, status) | Both |
| 16 | Voice: "Hey MAX" wake word → STT → AI response → TTS | Both |
| 17 | Audio alerts (6 sound types: activation, critical, warning, success, escalation, notify) | Both |
| 18 | Alert threshold tuning (snooze 3× → offer threshold change) | Both |
| 19 | pgVector knowledge base — MAX learns from every resolved incident | Both |
| 20 | All 5 role screens (SUPER_ADMIN, DEVELOPER, MARKETING, OPERATIONS, EXECUTIVE) | Both |

### Phase 3 — Autonomous AI Employee (Sprint 6–8, Weeks 7–10)
**Exit criteria:** business AI working, Teams/Slack connected, mobile-ready, multi-server automation, MAX monitors itself.

| # | Task | Pillar |
|---|---|---|
| 21 | Business AI: job postings, candidate search, reports, hiring predictions | New |
| 22 | Microsoft Teams + Slack bot integration | Both |
| 23 | Mobile-optimised UX (approve on phone, voice on mobile) | Both |
| 24 | Multi-server collector automation (Ansible / containerised) | Pillar 1 |
| 25 | Secure secrets store (Vault / environment-based) | Both |
| 26 | Cross-app event correlation (App A failure predicts App B failure) | Pillar 2 |
| 27 | Self-monitoring: MAX raises a CRITICAL incident if its own service goes down | Both |

---

## 25. Professional Phase Summary and Presentation
The project is divided into three clear implementation phases so stakeholders and developers can understand what we are doing, why it is needed, and how the architecture supports each step.

### Phase 1 — Foundation
- Build the secure core platform: database schema, user/auth flow, basic monitoring pipeline, and first incident workflow.
- Start both pillars with one server and one app.
- Why it is needed: a stable, secure foundation is required before adding AI decisions or operational automation.

### Phase 2 — AI Monitoring + Intelligence
- Add intelligence to both pillars: root cause AI, confidence scoring, approvals, escalation, rollback, voice alerts.
- Onboard all servers (Pillar 1) and all 12 apps (Pillar 2).
- Why it is needed: this is where MAX starts delivering real value — explaining issues, proposing fixes, learning from history.

### Phase 3 — Autonomous AI Employee
- Extend MAX to business AI, Teams/Slack, mobile maturity, multi-server automation, and self-monitoring.
- Why it is needed: after both pillars work reliably, MAX can scale into a true AI employee with safe human gating.

## 26. Mobile Implementation and Phase Order
Mobile implementation is built into the roadmap, not left to the end as an afterthought. Each phase includes mobile readiness with a clear order so the project does not miss any key step.

### Mobile work by phase
- **Phase 1:** mobile-ready architecture and responsive UI foundation
  - build Angular components that scale from desktop to mobile
  - design APIs for compact mobile summaries and charts
  - prepare the voice/chat widget for mobile devices
- **Phase 2:** mobile UX polish and real-time mobile interactions
  - implement touch-friendly approval and alert flows
  - optimize incident cards for smaller screens
  - add lightweight mobile notification and sound support
- **Phase 3:** mobile maturation and native-ready expansion
  - optimize navigation and performance for phones/tablets
  - support future native mobile apps using the same backend APIs
  - include mobile-ready escalation, rollback, and business actions

### Keep the order
1. Phase 1 first: secure foundation, auth, API, and responsive UI.
2. Phase 2 second: AI monitoring on both pillars, approval flows, and mobile-friendly UX.
3. Phase 3 last: autonomous orchestration, mobile maturity, and broader scale.

This order ensures nothing is skipped and every stage is clearly defined.

### Presentation and Stakeholder Communication
Two slide decks are now available in the workspace:
- `MAX-AI-Platform-Phases.pptx` — simple stakeholder summary
- `MAX-AI-Platform-Phases-Professional-Final.pptx` — polished professional deck with mobile and architecture slides

The professional deck is designed to be easier for everyone to read and understand, with:
- bold headers and visual sections
- boxed key points for each phase

## 26. Developer-Friendly Handoff
This project is packaged so any developer can understand the requirements and start building immediately.

### Files to share
- `MAX-Enterprise-AI-Platform.md` — master spec, architecture, APIs, database schema, and phase plan.
- `MAX-AI-Platform-Phases.pptx` — quick stakeholder deck.
- `MAX-AI-Platform-Phases-Professional-Final.pptx` — polished professional deck with mobile and architecture slides.
- `generate_ppt.py` / `generate_ppt_professional.py` — scripts to regenerate the slide decks.
- `index.html` / `venkatesh.html` — UI prototypes for the dashboard look and feel.

### What developers should see first
1. Read the phase plan and implementation roadmap.
2. Review the architecture overview and API/DB sections.
3. Use the mobile implementation section to ensure no mobile step is missed.
4. Open the PPT for quick project context and stakeholder alignment.

### Why this is easy for developers
- clear phase order with no skipped steps
- explicit mobile work in every phase
- separate design, architecture, and execution sections
- a single source of truth for requirements
- visual slides for quick review and technical detail in the MD file

### Recommended share package
When sending the project to developers or stakeholders, include:
- `MAX-Enterprise-AI-Platform.md`
- `MAX-AI-Platform-Phases-Professional-Final.pptx`
- `generate_ppt_professional.py`
- `index.html` or `venkatesh.html`

This ensures everyone sees the requirement, the architecture, and the delivery plan in a clean, developer-friendly format.

## 27. New Project Onboarding into MAX Monitoring
Adding a new application into MAX is a clear, repeatable process. These steps ensure the new project is fully monitored, alerted, and controlled by the AI platform.

### New project integration steps
1. Register the new app in `MAX_apps` with its name, display name, team owner, and assigned roles.
2. Define the app's alert thresholds in `MAX_alert_thresholds` for CPU, memory, response time, and error rate.
3. Deploy a lightweight collector for the app that streams metrics, logs, errors, and health data into Kafka.
4. Validate collector data in the monitoring pipeline and confirm the app appears in the MAX dashboard.
5. Add app-specific business workflows or intents to the business agent if the app requires custom actions.
6. Confirm role-based access so only authorized users can view and approve actions for the new app.
7. Validate mobile rendering for the app's incident summary, approvals, and notification flows.

### Why this step is essential
This makes every new application part of MAX in a consistent way, preventing special-case engineering and ensuring monitoring, alerting, approvals, rollback, and mobile support all work correctly.

---

*Document version 2.0 — June 2026*
*Author: Venkatesh N*
*Covers: Vision, Architecture, Angular+PrimeNG frontend, ASP.NET Core API, Python FastAPI+LangGraph AI, PostgreSQL complete schema, OTP authentication, RBAC, Confidence Scores, Escalation Chain, Rollback, Audit Log, Alert Tuning, Voice+Sound, Implementation Roadmap*

---

## 28. Coding Standards — Developer Reference

> **Rule:** Every developer on MAX must follow these standards exactly. This makes every file easy to understand, every name predictable, and every log consistent.

---

### 28.1 Logging — Serilog (ASP.NET Core)

All backend logging uses **Serilog**. No `Console.WriteLine`. No `Debug.Print`. No raw `ILogger` string formatting.

**Install:**
```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Seq
```

**Setup in `Program.cs`:**
```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext} — {Message:lj}{NewLine}{Exception}")
    .WriteTo.File("logs/max-.log",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 14)
    .WriteTo.Seq("http://localhost:5341")   // optional central log server
    .CreateLogger();

builder.Host.UseSerilog();
```

**How to log in any class:**
```csharp
// Inject ILogger<T> — never create it manually
public class AuthController : ControllerBase
{
    private readonly ILogger<AuthController> _logger;

    public AuthController(ILogger<AuthController> logger) => _logger = logger;

    // ✅ Correct — structured log, no string concat
    _logger.LogInformation("OTP verified for {Email} from IP {IpAddress}", email, ipAddress);
    _logger.LogWarning("OTP attempt {Count}/3 failed for {Email}", count, email);
    _logger.LogError(ex, "OTP verification failed for {Email}", email);

    // ❌ Wrong — never do these
    // Console.WriteLine("OTP verified for " + email);
    // _logger.LogInformation($"OTP verified for {email}");   // no string interpolation
}
```

**Log levels guide:**
| Level | When to use | Example |
|---|---|---|
| `LogTrace` | Deep debug only, never production | Loop iteration values |
| `LogDebug` | Dev debugging, disable in prod | Parsed request body |
| `LogInformation` | Normal business events | User logged in, OTP sent |
| `LogWarning` | Something unexpected but not broken | OTP attempt 2/3 |
| `LogError` | Something failed, needs attention | DB write failed |
| `LogCritical` | System cannot continue | Kafka connection lost |

**Python (FastAPI) — use `structlog`:**
```python
import structlog

log = structlog.get_logger(__name__)

# ✅ Correct
log.info("otp_sent", email=email, expires_in_seconds=300)
log.warning("otp_failed", email=email, attempt=attempt_count)
log.error("db_write_failed", error=str(e), table="MAX_otp_sessions")
```

---

### 28.2 Naming Conventions — Database (PostgreSQL)

| Item | Convention | Example |
|---|---|---|
| Table name | `MAX_` prefix + `snake_case` | `MAX_users`, `MAX_incidents`, `MAX_audit_log` |
| Column name | `snake_case` | `full_name`, `created_at`, `is_active` |
| Primary key | always `id SERIAL PRIMARY KEY` | `id` |
| Foreign key | `referenced_table_singular_id` | `user_id`, `app_id`, `incident_id` |
| Boolean column | starts with `is_` or `has_` | `is_active`, `has_otp`, `is_resolved` |
| Timestamp column | ends with `_at` | `created_at`, `resolved_at`, `expires_at` |
| Enum column | `VARCHAR` with `CHECK` constraint | `CHECK (role IN ('SUPER_ADMIN','DEVELOPER'))` |
| Index | `idx_tablename_columnname` | `idx_MAX_users_email` |
| Junction/link table | both tables joined | `MAX_user_apps` |

**Example (correct):**
```sql
CREATE TABLE MAX_incidents (
    id              SERIAL PRIMARY KEY,
    app_id          INTEGER NOT NULL REFERENCES MAX_apps(id),
    reported_by     INTEGER REFERENCES MAX_users(id),
    severity        VARCHAR(20) NOT NULL CHECK (severity IN ('CRITICAL','HIGH','MEDIUM','LOW')),
    title           TEXT NOT NULL,
    description     TEXT,
    is_resolved     BOOLEAN DEFAULT FALSE,
    resolved_at     TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_MAX_incidents_app_id   ON MAX_incidents(app_id);
CREATE INDEX idx_MAX_incidents_severity ON MAX_incidents(severity);
```

---

### 28.3 Naming Conventions — ASP.NET Core Backend

| Item | Convention | Example |
|---|---|---|
| Controller | `PascalCase` + `Controller` suffix | `AuthController`, `IncidentController` |
| Service interface | `I` prefix + `PascalCase` + `Service` | `IAuthService`, `IOtpService` |
| Service class | `PascalCase` + `Service` | `AuthService`, `OtpService` |
| Repository interface | `I` prefix + entity + `Repository` | `IUserRepository` |
| Repository class | entity + `Repository` | `UserRepository` |
| Model / Entity | `PascalCase`, singular | `User`, `Incident`, `App` |
| DTO | entity + `Dto` or specific name | `LoginRequestDto`, `OtpVerifyDto`, `UserResponseDto` |
| Method name | `PascalCase` verb-first | `SendOtp()`, `VerifyOtp()`, `GetActiveUsers()` |
| Variable / parameter | `camelCase` | `userId`, `otpCode`, `incidentId` |
| Constant | `UPPER_SNAKE_CASE` | `OTP_EXPIRY_MINUTES`, `MAX_ATTEMPTS` |
| Config key | `PascalCase` sections | `Jwt:SecretKey`, `Smtp:Host` |

**Folder structure rule:**
```
MAX-api/
├── Controllers/
│   ├── AuthController.cs        ← one controller per domain
│   ├── IncidentController.cs
│   └── UserController.cs
├── Services/
│   ├── Interfaces/
│   │   ├── IAuthService.cs
│   │   └── IOtpService.cs
│   ├── AuthService.cs
│   └── OtpService.cs
├── Repositories/
│   ├── Interfaces/
│   │   └── IUserRepository.cs
│   └── UserRepository.cs
├── Models/
│   ├── Entities/               ← DB-mapped models
│   │   ├── User.cs
│   │   └── Incident.cs
│   └── Dtos/                   ← request/response shapes only
│       ├── OtpSendDto.cs
│       └── OtpVerifyDto.cs
├── Middleware/
│   ├── AuditMiddleware.cs
│   └── RoleAuthorizationMiddleware.cs
└── Program.cs
```

**API endpoint naming:**
```
POST   /api/auth/validate-email          ← verb in URL only when action is not CRUD
POST   /api/auth/otp/send
POST   /api/auth/otp/verify
GET    /api/users                        ← plural resource
GET    /api/users/{id}
POST   /api/incidents
GET    /api/incidents/{id}
PATCH  /api/incidents/{id}/resolve       ← sub-action
```

---

### 28.4 Naming Conventions — Angular Frontend

| Item | Convention | Example |
|---|---|---|
| Component class | `PascalCase` + `Component` | `LoginComponent`, `IncidentListComponent` |
| Component folder | `kebab-case` | `incident-list/`, `login/`, `role-guard/` |
| Component file | `kebab-case.component.ts` | `incident-list.component.ts` |
| Service class | `PascalCase` + `Service` | `AuthService`, `IncidentService`, `MaxApiService` |
| Service file | `kebab-case.service.ts` | `auth.service.ts`, `max-api.service.ts` |
| Guard | `PascalCase` + `Guard` | `AuthGuard`, `RoleGuard` |
| Pipe | `PascalCase` + `Pipe` | `ConfidenceLevelPipe`, `TimeAgoPipe` |
| Interface / model | `PascalCase` | `User`, `Incident`, `OtpVerifyRequest` |
| Model file | `kebab-case.model.ts` | `user.model.ts`, `incident.model.ts` |
| Observable variable | ends with `$` | `incidents$`, `currentUser$` |
| Environment const | `camelCase` | `apiBaseUrl`, `jwtSecret` |
| Template binding | `camelCase` | `(click)="onApprove()"`, `[disabled]="isLoading"` |

**Component structure rule (one file, clear sections):**
```typescript
// incident-list.component.ts
@Component({ selector: 'app-incident-list', templateUrl: './incident-list.component.html' })
export class IncidentListComponent implements OnInit, OnDestroy {

  // ── 1. Public properties (bound in template) ──
  incidents: Incident[] = [];
  isLoading = false;

  // ── 2. Private properties ──
  private destroy$ = new Subject<void>();

  // ── 3. Constructor — inject only, no logic ──
  constructor(private incidentService: IncidentService) {}

  // ── 4. Lifecycle hooks ──
  ngOnInit(): void { this.loadIncidents(); }
  ngOnDestroy(): void { this.destroy$.next(); this.destroy$.complete(); }

  // ── 5. Public methods (called from template) ──
  onApprove(id: number): void { /* ... */ }
  onDeny(id: number): void    { /* ... */ }

  // ── 6. Private methods (internal logic) ──
  private loadIncidents(): void { /* ... */ }
}
```

---

### 28.5 Naming Conventions — Python FastAPI (AI Layer)

| Item | Convention | Example |
|---|---|---|
| File name | `snake_case.py` | `monitoring_agent.py`, `root_cause_engine.py` |
| Class name | `PascalCase` | `MonitoringAgent`, `RootCauseEngine` |
| Function name | `snake_case` | `analyze_incident()`, `calculate_confidence()` |
| Variable name | `snake_case` | `incident_id`, `confidence_score` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_RETRY_COUNT`, `OTP_EXPIRY_SECONDS` |
| API route | `kebab-case` path | `/api/ai/analyze-incident` |
| Pydantic model | `PascalCase` | `IncidentAnalysisRequest`, `ConfidenceResponse` |

**Agent file structure rule:**
```
MAX-ai/
├── agents/
│   ├── monitoring_agent.py      ← one file per agent
│   ├── root_cause_agent.py
│   ├── escalation_agent.py
│   ├── rollback_agent.py
│   └── knowledge_agent.py
├── engine/
│   ├── confidence_engine.py     ← confidence score logic
│   ├── prediction_engine.py     ← ML trend prediction
│   └── correlation_engine.py   ← cross-app pattern detection
├── models/
│   ├── incident.py              ← Pydantic models
│   └── analysis_result.py
├── routers/
│   └── ai_router.py            ← FastAPI route definitions
├── services/
│   └── kafka_consumer.py
└── main.py
```

---

### 28.6 General Code Rules (All Languages)

| Rule | Detail |
|---|---|
| **One responsibility per file** | One controller, one service, one component per file — never mix two domains |
| **No magic numbers** | Use named constants. `OTP_EXPIRY_MINUTES = 5`, not `5` |
| **No plain text secrets** | All secrets in environment variables or Vault. Never in code or committed files |
| **No unused imports** | Remove all unused `using`, `import`, `from x import y` |
| **No commented-out code** | Delete dead code. Use git history to recover if needed |
| **Max function length** | 30 lines. If longer, extract a private method |
| **Max file length** | 300 lines. If longer, split into sub-files or sub-components |
| **Error handling** | Catch specific exceptions, log with context, return proper HTTP status codes |
| **No raw SQL strings in controllers** | All DB queries in Repository layer only |
| **Async all the way** | All I/O (DB, HTTP, Kafka) must be `async/await` — never `.Result` or `.Wait()` |

**HTTP status code rules (ASP.NET Core):**
```csharp
return Ok(result);              // 200 — success with data
return Created(location, dto);  // 201 — created
return NoContent();             // 204 — success no data (DELETE)
return BadRequest("message");   // 400 — bad input
return Unauthorized();          // 401 — not logged in
return Forbid();                // 403 — logged in but no permission
return NotFound("message");     // 404 — resource not found
return StatusCode(500, "msg");  // 500 — unexpected server error
```

---

### 28.7 Git Commit Message Standard

Format: `type(scope): short description`

| Type | When | Example |
|---|---|---|
| `feat` | New feature | `feat(auth): add email-based OTP login` |
| `fix` | Bug fix | `fix(incidents): prevent duplicate alert on retry` |
| `refactor` | Code improvement, no behaviour change | `refactor(auth): extract OtpService from AuthController` |
| `style` | CSS / formatting only | `style(login): update button colour to cyan` |
| `docs` | Documentation only | `docs(md): add coding standards section` |
| `chore` | Build, config, dependencies | `chore: upgrade Serilog to 3.1.1` |
| `test` | Add or update tests | `test(auth): add OTP expiry test` |

**Branch naming:**
```
feature/auth-email-otp
fix/incident-duplicate-alert
release/v1.0.0
hotfix/otp-lockout-bug
```

---

## 29. Pre-Development Review — Improvements Added Before Sprint 1

> This section records every gap found during the full document review and what was done. Read this before writing a single line of code.

---

### 29.1 Issues Found & Fixed in This Document

| # | Issue Found | Fix Applied |
|---|---|---|
| 1 | Auth flow said "user clicks their name from DB list" — old pattern | Updated to email-based login throughout (Section 17 + UI) |
| 2 | API endpoints still referenced `{ user_id }` for OTP | Updated to `{ email }` in Section 14 API list |
| 3 | Angular folder comment said "Step1: Name" | Updated to "Step1: Email" |
| 4 | Duplicate section numbers (two "Section 15", "Section 16") | Renumbered cleanly: 15a, 15b, 15c merged properly |
| 5 | No environment config strategy defined | Added Section 29.2 below |
| 6 | No error response contract defined | Added Section 29.3 below |
| 7 | No health check / readiness endpoint defined | Added Section 29.4 below |
| 8 | No data retention / archiving strategy | Added Section 29.5 below |
| 9 | No testing strategy defined | Added Section 29.6 below |
| 10 | No rate limiting / DDoS protection mentioned | Added Section 29.7 below |
| 11 | `venkatesh.html` still existed as a conflicting file | Marked DEPRECATED — `index.html` is the only UI file |
| 12 | Document version was 2.0 pre-review | Bumped to v3.0 — Pre-Dev Review Complete |

---

### 29.2 Environment Configuration Strategy — All Credentials in Config, Never in Code

> **Golden rule: zero credentials, zero connection strings, zero secrets in source code. Ever.**
> All values that change per environment live in `appsettings.{env}.json` or `.env` files.
> Files with real secrets are NEVER committed to git.

**Three environments — all layers must support all three:**
| Environment | Purpose | Who Uses It |
|---|---|---|
| `local` | Developer machine, local DB, no real email | Individual devs |
| `uat` | Test server, real data shape, test email | QA + client review |
| `prod` | Live server, real users, real email | Everyone |

---

**ASP.NET Core — appsettings file structure:**
```
MAX-api/
├── appsettings.json                  ← committed — structure only, no secrets
├── appsettings.Development.json      ← committed — local dev overrides (no secrets)
├── appsettings.UAT.json              ← on UAT server only — never in git
└── appsettings.Production.json       ← on prod server only — never in git
```

`appsettings.json` (committed — no secrets, just structure):
```json
{
  "Jwt": {
    "Issuer":        "MAX",
    "Audience":      "MAX-users",
    "SecretKey":     "",
    "ExpiryHours":   8
  },
  "Smtp": {
    "Host":          "",
    "Port":          587,
    "FromAddress":   "",
    "Username":      "",
    "Password":      ""
  },
  "ConnectionStrings": {
    "Default":       ""
  },
  "Kafka": {
    "BootstrapServers": "",
    "GroupId":          "max-api"
  },
  "Redis": {
    "ConnectionString": ""
  },
  "AiService": {
    "BaseUrl":       "http://localhost:8000"
  },
  "RateLimit": {
    "OtpSendPerEmail": 3,
    "OtpSendWindowMinutes": 15
  }
}
```

`appsettings.Development.json` (committed — safe local defaults):
```json
{
  "Jwt":    { "SecretKey": "local-dev-secret-min-32-chars-long!" },
  "Smtp":   { "Host": "localhost", "Port": 1025 },
  "ConnectionStrings": { "Default": "Host=localhost;Database=maxdb_dev;Username=postgres;Password=postgres" },
  "Kafka":  { "BootstrapServers": "localhost:9092" },
  "Redis":  { "ConnectionString": "localhost:6379" }
}
```

`appsettings.Production.json` (on server only — never in git, add to `.gitignore`):
```json
{
  "Jwt":    { "SecretKey": "<strong-prod-secret-min-32-chars>" },
  "Smtp":   { "Host": "smtp.sendgrid.net", "Port": 587, "Username": "apikey", "Password": "<sendgrid-key>", "FromAddress": "max@yourcompany.com" },
  "ConnectionStrings": { "Default": "Host=<prod-db-host>;Database=maxdb;Username=max_user;Password=<prod-password>" },
  "Kafka":  { "BootstrapServers": "<prod-kafka-host>:9092" },
  "Redis":  { "ConnectionString": "<prod-redis-host>:6379,password=<prod-redis-password>" }
}
```

Set environment on the server:
```bash
export ASPNETCORE_ENVIRONMENT=Production   # Linux
# or in systemd service file:
# Environment=ASPNETCORE_ENVIRONMENT=Production
```

`.gitignore` must include:
```
appsettings.UAT.json
appsettings.Production.json
```

---

**Angular — environment file structure:**
```
MAX-ui/src/environments/
├── environment.ts           ← local dev — committed
├── environment.uat.ts       ← UAT — committed (no secrets, just URLs)
└── environment.prod.ts      ← prod — committed (no secrets, just URLs)
```

`environment.ts` (local):
```typescript
export const environment = {
  production:  false,
  apiBaseUrl:  'http://localhost:5000/api',
  wsUrl:       'ws://localhost:5000/hubs/events',
  appName:     'MAX — Local',
};
```

`environment.uat.ts`:
```typescript
export const environment = {
  production:  false,
  apiBaseUrl:  'https://uat.max.yourcompany.com/api',
  wsUrl:       'wss://uat.max.yourcompany.com/hubs/events',
  appName:     'MAX — UAT',
};
```

`environment.prod.ts`:
```typescript
export const environment = {
  production:  true,
  apiBaseUrl:  'https://max.yourcompany.com/api',
  wsUrl:       'wss://max.yourcompany.com/hubs/events',
  appName:     'MAX',
};
```

Build commands:
```bash
ng build                    # uses environment.ts (local)
ng build --configuration uat   # uses environment.uat.ts
ng build --configuration production  # uses environment.prod.ts
```

---

**Python FastAPI — `.env` file structure:**
```
MAX-ai/
├── .env.example      ← committed — shows all keys, no real values
├── .env.development  ← committed — safe local defaults
├── .env.uat          ← on UAT server only — never in git
└── .env.production   ← on prod server only — never in git
```

`.env.example` (committed — template for all developers):
```
DATABASE_URL=
KAFKA_BOOTSTRAP=
REDIS_URL=
JWT_SECRET=
OLLAMA_BASE_URL=
OPENAI_API_KEY=
SMTP_HOST=
SMTP_PORT=
SMTP_USERNAME=
SMTP_PASSWORD=
ENVIRONMENT=local
```

`.env.development` (committed — safe local values):
```
DATABASE_URL=postgresql://postgres:postgres@localhost/maxdb_dev
KAFKA_BOOTSTRAP=localhost:9092
REDIS_URL=redis://localhost:6379
JWT_SECRET=local-dev-secret-min-32-chars-long!
OLLAMA_BASE_URL=http://localhost:11434
OPENAI_API_KEY=
ENVIRONMENT=local
```

`.gitignore` must include:
```
.env.uat
.env.production
.env          # safety net
```

Load in Python:
```python
from dotenv import load_dotenv
import os

env = os.getenv('ENVIRONMENT', 'local')
load_dotenv(f'.env.{env}')   # load environment-specific file

DATABASE_URL = os.getenv('DATABASE_URL')
JWT_SECRET   = os.getenv('JWT_SECRET')
```

---

### 29.3 Standard API Error Response Contract

Every error from the ASP.NET Core API must return this exact shape. Frontend and AI layer both expect this format.

```json
{
  "success": false,
  "errorCode": "OTP_EXPIRED",
  "message": "Your OTP has expired. Please request a new one.",
  "details": null,
  "timestamp": "2026-06-16T10:30:00Z"
}
```

**Error codes used in MAX:**
| Code | HTTP | When |
|---|---|---|
| `EMAIL_NOT_FOUND` | 404 | Email not in MAX_users or is_active = false |
| `OTP_INVALID` | 400 | Wrong OTP entered |
| `OTP_EXPIRED` | 400 | OTP older than 5 minutes |
| `OTP_LOCKED` | 429 | 3 failed attempts — 15 min lockout |
| `UNAUTHORIZED` | 401 | No JWT or expired JWT |
| `FORBIDDEN` | 403 | Role not allowed for this action |
| `INCIDENT_NOT_FOUND` | 404 | Incident ID does not exist |
| `APPROVAL_LEVEL_INSUFFICIENT` | 403 | User's level too low to approve |
| `SNAPSHOT_FAILED` | 500 | Pre-action snapshot could not be taken |
| `VALIDATION_ERROR` | 422 | Request body failed validation |

**C# helper:**
```csharp
public static class ApiResponse
{
    public static IActionResult Fail(ControllerBase c, int status, string code, string msg)
        => c.StatusCode(status, new { success = false, errorCode = code,
                                      message = msg, details = (object?)null,
                                      timestamp = DateTime.UtcNow });

    public static IActionResult Ok<T>(ControllerBase c, T data)
        => c.Ok(new { success = true, data, timestamp = DateTime.UtcNow });
}
```

---

### 29.4 Health Check & Readiness Endpoints

Every service must expose health endpoints so MAX can monitor itself and load balancers can route correctly.

**ASP.NET Core:**
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddNpgsql(connectionString, name: "postgres")
    .AddRedis(redisUrl, name: "redis")
    .AddKafka(kafkaConfig, name: "kafka");

app.MapHealthChecks("/health/live");    // is the process alive?
app.MapHealthChecks("/health/ready",   // is it ready to serve traffic?
    new HealthCheckOptions { Predicate = _ => true });
```

**Python FastAPI:**
```python
@app.get("/health/live")
def liveness():
    return {"status": "ok"}

@app.get("/health/ready")
def readiness():
    # check DB, Kafka, Redis connections
    return {"status": "ok", "checks": { "db": "ok", "kafka": "ok" }}
```

MAX monitors these endpoints for itself: if `/health/ready` returns non-200, MAX raises a CRITICAL self-incident.

---

### 29.5 Data Retention & Archiving

MAX generates a lot of data. Without a retention policy the database grows uncontrolled.

| Table | Retention | Strategy |
|---|---|---|
| `MAX_app_metrics` | 90 days hot, 1 year archive | Partition by month; auto-drop old partitions |
| `MAX_audit_log` | 2 years (compliance) | Archive to cold storage after 6 months |
| `MAX_otp_sessions` | 24 hours after use or expiry | Nightly cleanup job |
| `MAX_action_snapshots` | 24 hours (then auto-purge) | Already in schema — add a scheduled job |
| `MAX_incidents` | Permanent (resolved ones move to archive table) | Archive after 6 months |
| `MAX_knowledge_base` | Permanent + growing | Never delete — this is MAX's memory |

**Cleanup job (runs nightly via pg_cron or a background service):**
```sql
-- Remove expired OTP sessions
DELETE FROM MAX_otp_sessions WHERE expires_at < NOW() - INTERVAL '1 day';

-- Remove expired snapshots
DELETE FROM MAX_action_snapshots WHERE expires_at < NOW() AND rolled_back = false;

-- Archive old metrics (move to MAX_app_metrics_archive)
INSERT INTO MAX_app_metrics_archive SELECT * FROM MAX_app_metrics
  WHERE recorded_at < NOW() - INTERVAL '90 days';
DELETE FROM MAX_app_metrics WHERE recorded_at < NOW() - INTERVAL '90 days';
```

---

### 29.6 Testing Strategy

Every layer must have tests before going to UAT. No exceptions.

| Layer | Test Type | Tool | What to Test |
|---|---|---|---|
| ASP.NET Core API | Unit tests | xUnit + Moq | OTP hash, JWT claims, role checks, error codes |
| ASP.NET Core API | Integration tests | xUnit + TestServer | Full auth flow, incident approval flow |
| Angular UI | Unit tests | Jasmine + Karma | AuthService, role guards, OTP component |
| Angular UI | E2E tests | Cypress | Login flow, dashboard loads, approve incident |
| Python FastAPI | Unit tests | pytest | Confidence engine, root cause logic |
| Python FastAPI | API tests | pytest + httpx | All /ai/* endpoints |
| Database | Migration tests | Manual + script | Schema applies cleanly, seed data loads |

**Minimum coverage target before UAT: 70% on business logic.**

**Critical paths that MUST have tests (non-negotiable):**
1. Email validate → OTP send → OTP verify → JWT issued
2. Incident APPROVE → snapshot taken → action executed → health check → audit logged
3. Escalation timer fires → backup admin notified → audit logged
4. Rollback → snapshot restored → health check → audit logged
5. Role refusal — every role attempting a forbidden action returns 403

---

### 29.7 Rate Limiting & Security Hardening

**Rate limits (ASP.NET Core — `AspNetCoreRateLimit` package):**
| Endpoint | Limit | Window |
|---|---|---|
| `POST /api/auth/validate-email` | 10 requests | per IP per minute |
| `POST /api/auth/otp/send` | 3 requests | per email per 15 minutes |
| `POST /api/auth/otp/verify` | 3 attempts | per session (then lockout) |
| All other endpoints | 200 requests | per user per minute |

**Security headers (add in middleware):**
```csharp
app.Use(async (ctx, next) => {
    ctx.Response.Headers["X-Content-Type-Options"]    = "nosniff";
    ctx.Response.Headers["X-Frame-Options"]           = "DENY";
    ctx.Response.Headers["X-XSS-Protection"]          = "1; mode=block";
    ctx.Response.Headers["Referrer-Policy"]           = "no-referrer";
    ctx.Response.Headers["Permissions-Policy"]        = "microphone=(self)";
    await next();
});
```

**CORS (only allow known origins):**
```csharp
builder.Services.AddCors(o => o.AddPolicy("MAXPolicy", p => p
    .WithOrigins("https://max.yourcompany.com", "http://localhost:4200")
    .AllowAnyHeader()
    .AllowAnyMethod()
    .AllowCredentials()));
```

**Never do:**
- Never log OTP values, passwords, or JWT secrets
- Never return stack traces to the client in production
- Never store JWT in localStorage — memory / BehaviorSubject only
- Never allow `SELECT *` in production queries — always name columns

---

### 29.8 Sprint 1 — What to Build First (Exact Order)

Follow this order exactly. Do not start Sprint 2 until Sprint 1 is complete and tested.

**Sprint 1 — Foundation (Week 1–2):**
1. PostgreSQL schema + seed data (`MAX_users`, `MAX_apps`, `MAX_alert_thresholds`)
   > SQL scripts live in `database/scripts/` folder — run manually in order (see below)
2. ASP.NET Core project setup: Serilog, Swagger, health checks, CORS, rate limiting, error response contract
   > Set up all three `appsettings` files (Section 29.2) before writing any controller
3. `AuthController` — validate email → OTP send → OTP verify → JWT
4. Angular project setup: folder structure, all three `environment.ts` files, auth module, `AuthService`, `AuthGuard`
5. Angular login screen: email input → OTP boxes → role redirect (matches `index.html` prototype)
6. One working collector for SRJ app → Kafka → metrics land in `MAX_app_metrics`
7. Basic `IncidentController` — create, list, get by id (no AI yet)
8. Angular dashboard — shows incidents list (hardcoded data is fine for now)

**SQL scripts folder structure (run manually in this exact order):**
```
database/
└── scripts/
    ├── 01_extensions.sql          ← CREATE EXTENSION vector;
    ├── 02_users_auth.sql          ← MAX_users, MAX_otp_sessions
    ├── 03_apps_metrics.sql        ← MAX_apps, MAX_app_metrics
    ├── 04_incidents.sql           ← MAX_incidents
    ├── 05_snapshots.sql           ← MAX_action_snapshots
    ├── 06_audit_log.sql           ← MAX_audit_log
    ├── 07_thresholds.sql          ← MAX_alert_thresholds
    ├── 08_escalation.sql          ← MAX_escalation_rules
    ├── 09_knowledge_base.sql      ← MAX_knowledge_base + pgVector index
    ├── 10_business.sql            ← MAX_business_actions
    ├── 11_server_inventory.sql    ← MAX_server_inventory
    └── 99_seed_data.sql           ← INSERT sample users, apps, thresholds
```

Run on local:
```bash
psql -U postgres -d maxdb -f database/scripts/01_extensions.sql
psql -U postgres -d maxdb -f database/scripts/02_users_auth.sql
# ... continue in order up to 99
```

Run on UAT/Prod:
```bash
psql "<connection-string>" -f database/scripts/01_extensions.sql
# ... same order
```

> **Rule:** Never modify a script that has already been run on UAT or prod.
> For changes after initial deploy, add a new numbered file: `12_add_column_xxx.sql`

**Sprint 1 exit criteria (UAT sign-off checklist):**
- [ ] Can login with a registered email via OTP
- [ ] Wrong email shows correct error message
- [ ] Wrong OTP locks after 3 attempts
- [ ] JWT role is correct for each test user
- [ ] Each role is redirected to the right page
- [ ] SRJ metrics visible in `MAX_app_metrics` table
- [ ] Incidents list loads in the dashboard
- [ ] All Serilog logs are structured (no plain Console.WriteLine anywhere)
- [ ] `/health/live` and `/health/ready` return 200
- [ ] All unit tests pass

---

## 30. Production Readiness — Before Go-Live

---

### 30.1 Docker Compose — One Command Local Setup (B)

**What it is:** A single file that starts every service on a developer's laptop with one command. No manual installs.

**Why you need it:** Without this, every new developer wastes 1–3 days installing PostgreSQL, Kafka, Redis, and Ollama manually. With this, they run one command and everything is running in 2 minutes.

**How it works:**
```bash
# Start everything locally
docker compose up -d

# Stop everything
docker compose down

# Check what's running
docker compose ps
```

**File to create: `docker-compose.dev.yml` in project root:**
```yaml
version: '3.9'

services:
  postgres:
    image: pgvector/pgvector:pg15
    container_name: max-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: maxdb_dev
    ports:
      - "5432:5432"
    volumes:
      - max-pg-data:/var/lib/postgresql/data
      - ./database/scripts:/docker-entrypoint-initdb.d   # auto-runs scripts on first start

  redis:
    image: redis:7-alpine
    container_name: max-redis
    ports:
      - "6379:6379"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: max-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: max-kafka
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"

  ollama:
    image: ollama/ollama:latest
    container_name: max-ollama
    ports:
      - "11434:11434"
    volumes:
      - max-ollama-data:/root/.ollama

  mailhog:
    image: mailhog/mailhog
    container_name: max-mailhog
    ports:
      - "1025:1025"    # SMTP — used by appsettings.Development.json
      - "8025:8025"    # Web UI — open http://localhost:8025 to see all emails

volumes:
  max-pg-data:
  max-ollama-data:
```

> **Note:** `MAX-api` and `MAX-ai` are NOT in this file — developers run those locally from their IDE. This file only starts the infrastructure services.

**After starting, pull the Ollama model once:**
```bash
docker exec max-ollama ollama pull llama3
```

---

### 30.2 PostgreSQL Backup Strategy (C)

**What it is:** A daily automatic backup of the MAX database, saved to a safe location, with clear steps to restore if the server dies.

**Why you need it:** If the IONOS server dies tonight and there is no backup, all audit logs, incidents, knowledge base, and user data are gone permanently. This defines exactly how backups work.

**Backup schedule:**
| Backup type | Frequency | Kept for | Where |
|---|---|---|---|
| Full database dump | Daily at 2 AM | 30 days | `/backups/daily/` on same server |
| Full database dump | Weekly (Sunday 3 AM) | 3 months | Offsite (SFTP / S3 / Ionos Object Storage) |
| Pre-deployment snapshot | Before every deployment | 7 days | `/backups/pre-deploy/` |

**Backup script — save as `/opt/max/backup.sh` on the server:**
```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/daily"
DB_NAME="maxdb"
DB_USER="max_user"

mkdir -p $BACKUP_DIR

# Dump the database
pg_dump -U $DB_USER -Fc $DB_NAME > "$BACKUP_DIR/max_$DATE.dump"

# Keep only last 30 days
find $BACKUP_DIR -name "*.dump" -mtime +30 -delete

echo "Backup complete: max_$DATE.dump"
```

**Schedule it (add to crontab on the server):**
```bash
crontab -e
# Add this line:
0 2 * * * /opt/max/backup.sh >> /var/log/max-backup.log 2>&1
```

**Restore from backup (step-by-step):**
```bash
# 1. Stop the MAX API so no writes happen during restore
docker compose stop max-api max-ai

# 2. Drop and recreate the database
psql -U postgres -c "DROP DATABASE IF EXISTS maxdb;"
psql -U postgres -c "CREATE DATABASE maxdb OWNER max_user;"

# 3. Restore the dump
pg_restore -U max_user -d maxdb /backups/daily/max_20260616_020000.dump

# 4. Restart services
docker compose start max-api max-ai

# 5. Verify: check user count
psql -U max_user -d maxdb -c "SELECT COUNT(*) FROM MAX_users;"
```

> **Rule:** Test the restore procedure every month on a copy. A backup you have never tested is not a backup.

---

### 30.3 Secret Rotation Policy (F)

**What it is:** A schedule for changing all passwords and keys, and the steps to do it safely without breaking anything.

**Why you need it:** If a secret leaks (old employee, hacked laptop, accidental git commit), you need to know exactly how to change it and how quickly. Without a rotation policy, a leaked secret stays valid forever.

**Rotation schedule:**
| Secret | Rotate every | How to rotate |
|---|---|---|
| JWT secret key | 6 months or immediately if leaked | Steps below |
| PostgreSQL password (`max_user`) | 6 months | Steps below |
| SMTP password / SendGrid API key | 6 months | Steps below |
| Redis password | 12 months | Update `.env.production` + restart Redis |
| Kafka credentials | 12 months | Update `.env.production` + restart Kafka |

**How to rotate JWT secret (most common):**
```
1. Generate a new strong secret (min 32 chars):
   openssl rand -base64 32

2. Update appsettings.Production.json on the server:
   "Jwt": { "SecretKey": "<new-secret>" }

3. Restart MAX API:
   docker compose restart max-api

4. Result: all existing JWT tokens become invalid immediately.
   All users will be logged out and must log in again with OTP.
   This is expected and correct behaviour.

5. Log the rotation in MAX_audit_log manually:
   INSERT INTO MAX_audit_log (action_type, description, timestamp)
   VALUES ('THRESHOLD_CHANGED', 'JWT secret rotated by Venkatesh N', NOW());
```

**How to rotate PostgreSQL password:**
```bash
# 1. Change password in PostgreSQL
psql -U postgres -c "ALTER USER max_user PASSWORD '<new-password>';"

# 2. Update appsettings.Production.json connection string
# 3. Update .env.production DATABASE_URL
# 4. Restart both API and AI service
docker compose restart max-api max-ai
```

**If a secret is leaked urgently:**
- Rotate it immediately — do not wait for the schedule
- Check `MAX_audit_log` for any suspicious logins or actions during the exposure window
- If JWT was leaked: rotate secret + manually deactivate any suspicious users (`is_active = false`)

---

### 30.4 Disaster Recovery Plan (G)

**What it is:** The exact steps to get MAX fully working again after a complete server failure.

**Why you need it:** A dream project cannot afford to lose data or stay down for days. This gives a clear step-by-step playbook so recovery is fast and nothing is forgotten.

**Recovery time targets:**
| Scenario | Target recovery time |
|---|---|
| Single service crash (API, AI, Redis) | < 5 minutes (Docker auto-restart) |
| Full server reboot | < 15 minutes |
| Server hardware failure (full rebuild) | < 4 hours |
| Data corruption / accidental delete | < 2 hours (from backup) |

**Full server rebuild — step by step:**
```
Step 1 — Provision new server (Ionos or any Linux VPS)
  - Ubuntu 22.04 LTS, minimum 16 GB RAM, 200 GB disk

Step 2 — Install Docker + Docker Compose
  curl -fsSL https://get.docker.com | sh
  apt install docker-compose-plugin

Step 3 — Clone the MAX repository
  git clone https://github.com/yourorg/max-platform.git
  cd max-platform

Step 4 — Place config files (not in git — copy from secure storage)
  - appsettings.Production.json  → MAX-api/
  - .env.production              → MAX-ai/

Step 5 — Start infrastructure services
  docker compose -f docker-compose.prod.yml up -d postgres redis kafka zookeeper ollama

Step 6 — Restore database from latest backup
  pg_restore -U max_user -d maxdb /backups/max_latest.dump

Step 7 — Start application services
  docker compose -f docker-compose.prod.yml up -d max-api max-ai max-ui

Step 8 — Verify
  curl https://max.yourcompany.com/health/ready
  → should return {"status":"ok"}

Step 9 — Run smoke tests
  - Login with test email
  - Check dashboard loads
  - Check one incident visible

Step 10 — Notify team that MAX is restored
```

**What to keep in secure storage (NOT in git):**
- `appsettings.Production.json`
- `.env.production`
- Latest database backup copy
- Server SSH keys

---

### 30.5 Performance Targets — How Many Users, How Fast (H)

**What it is:** Clear numbers that define when MAX is ready for production. Without these numbers, there is no way to know if the system is fast enough or strong enough.

**Why you need it:** MAX is a real-time AI platform. If it is slow, people stop trusting it. If it crashes under load, it is worse than no monitoring at all.

**Response time targets:**
| Action | Target | Maximum acceptable |
|---|---|---|
| Login (validate email) | < 200 ms | 500 ms |
| OTP verify + JWT issue | < 300 ms | 700 ms |
| Dashboard load (incidents list) | < 500 ms | 1000 ms |
| Incident detail + AI analysis | < 2000 ms | 4000 ms |
| Voice response (STT → AI → TTS) | < 3000 ms | 5000 ms |
| Real-time push (SignalR event to browser) | < 500 ms | 1000 ms |
| pgVector knowledge base search | < 800 ms | 2000 ms |

**Concurrent user targets:**
| Environment | Max simultaneous users | Notes |
|---|---|---|
| Local dev | 1–5 | No performance concern |
| UAT | 10–20 | Full team testing |
| Production (initial) | 50 | Current IONOS server (32 GB RAM) |
| Production (scaled) | 200+ | After adding GPU + Kubernetes |

**Load test checklist (run before prod go-live):**
```
Tool: k6 (free, open source — https://k6.io)

Test 1 — Login flood
  50 users logging in simultaneously
  All must succeed in < 700 ms
  No OTP lockouts triggered by test users

Test 2 — Dashboard sustained load
  30 users with dashboard open for 10 minutes
  SignalR events delivered to all 30 in < 500 ms each
  No memory leaks in API process

Test 3 — AI analysis burst
  10 incidents created simultaneously
  All AI analyses complete in < 4 seconds each
  No incidents lost or stuck in OPEN state

Test 4 — Kafka throughput
  All 12 app collectors sending metrics every 5 seconds = ~144 events/minute
  All events processed with < 2 second lag

Pass criteria: all 4 tests pass with no errors for 10 minutes straight.
If any test fails: do NOT go to production. Fix and re-test.
```

**Simple k6 example — run this before prod:**
```javascript
// load-test/dashboard.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 30,        // 30 simultaneous users
  duration: '5m', // for 5 minutes
};

export default function () {
  const res = http.get('https://max.yourcompany.com/api/apps', {
    headers: { Authorization: `Bearer ${__ENV.TEST_JWT}` },
  });
  check(res, { 'status 200': (r) => r.status === 200 });
  check(res, { 'under 500ms': (r) => r.timings.duration < 500 });
  sleep(1);
}
```

```bash
# Run it
k6 run --env TEST_JWT=<test-jwt-token> load-test/dashboard.js
```


