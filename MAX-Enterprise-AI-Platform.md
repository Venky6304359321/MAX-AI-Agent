# MAX Enterprise AI Platform
## Complete Technical Blueprint — Build-Ready Design Document
**Version:** 2.0 — June 2026 | **Author:** Venkatesh N (Super Admin / Tech Lead)
**Status:** Design Complete — Ready for Implementation

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
|   |-- login/                   <- 3-step OTP screen (Step1: Name, Step2: OTP, Step3: Verified)
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
GET    /api/users/active                      -> name list for login screen
POST   /api/auth/otp/send    { user_id }      -> generate + email OTP
POST   /api/auth/otp/verify  { user_id, otp } -> verify -> JWT returned
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

## 15. AI Service — Python FastAPI + LangGraph

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

## 17. OTP Authentication — Complete Flow

```
Admin registers employee once in MAX_users table
      |
MAX opens on any machine (no machine-specific setup)
      |
GET /api/users/active -> names shown from DB with masked email
      |
Employee clicks their name
      |
POST /api/auth/otp/send { user_id }
  -> 6-digit OTP generated
  -> SHA-256 hashed + stored in MAX_otp_sessions (expires 5 min)
  -> Email sent via SMTP: "Your MAX OTP is 847291 — valid 5 minutes"
      |
Employee enters 6 boxes (auto-advance on type, backspace navigates back)
      |
POST /api/auth/otp/verify { user_id, otp }
  -> Hash input -> compare to DB hash
  -> Check expiry (5 min) + attempt count (max 3 -> 15 min lockout)
  -> On success: JWT returned with role + approval_level claims
      |
JWT stored in Angular BehaviorSubject (memory only — not localStorage)
All API calls include Bearer JWT header
Role enforced server-side on every request
Every action auto-logged to MAX_audit_log with device + browser + IP
```

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

## 23. Implementation Roadmap

| Phase | What to Build | When |
|---|---|---|
| 1 | PostgreSQL schema + seed data | Immediate |
| 2 | ASP.NET Core — Auth + OTP + JWT | Week 1 |
| 3 | Angular login — 3-step OTP flow | Week 1 |
| 4 | Kafka + Collector for SRJ (first app) | Week 1 |
| 5 | Python FastAPI — basic monitoring AI | Week 1 |
| 6 | Angular admin dashboard — incidents + approve | Week 2 |
| 7 | SignalR real-time WebSocket events | Week 2 |
| 8 | Voice — wake word + STT + TTS | Week 3 |
| 9 | Audio alerts (6 sound types) | Week 3 |
| 10 | Confidence score engine | Week 3 |
| 11 | Escalation chain + live timer | Week 4 |
| 12 | Rollback snapshot system | Week 4 |
| 13 | Alert threshold tuning | Week 5 |
| 14 | All 5 role screens | Week 5 |
| 15 | Business AI (job postings + candidates) | Week 6 |
| 16 | pgVector knowledge base | Week 7 |
| 17 | Mobile responsive layout | Week 8 |

---

*Document version 2.0 — June 2026*
*Author: Venkatesh N*
*Covers: Vision, Architecture, Angular+PrimeNG frontend, ASP.NET Core API, Python FastAPI+LangGraph AI, PostgreSQL complete schema, OTP authentication, RBAC, Confidence Scores, Escalation Chain, Rollback, Audit Log, Alert Tuning, Voice+Sound, Implementation Roadmap*

