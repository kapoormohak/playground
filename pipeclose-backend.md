# Pipeclose Backend Documentation & Architecture

Pipeclose (*Mini CRM*) is an extensible, enterprise-ready Customer Relationship Management (CRM) backend built with Node.js, TypeScript, Express, Socket.IO, and SQLite. It provides pipeline sales tracking, two-way email synchronization, in-browser VoIP calling via Twilio, calendar scheduling, and an AI sales copilot.

---

## Table of Contents
1. [High-Level System Architecture](#1-high-level-system-architecture)
2. [Data Model & Entity Relationships](#2-data-model--entity-relationships)
3. [Authentication & Authorization Flow](#3-authentication--authorization-flow)
4. [Sales Pipeline & Deal Management Flow](#4-sales-pipeline--deal-management-flow)
5. [Two-Way Email Sync & Instant Notification Flow](#5-two-way-email-sync--instant-notification-flow)
6. [Telephony & WebRTC Calling Flow (Twilio)](#6-telephony--webrtc-calling-flow-twilio)
7. [Calendar & Reminder Processing Flow](#7-calendar--reminder-processing-flow)
8. [AI Sales Agent & Copilot Flow](#8-ai-sales-agent--copilot-flow)
9. [Bulk Data Import Flow](#9-bulk-data-import-flow)
10. [API Route Reference](#10-api-route-reference)

---

## 1. High-Level System Architecture

The application adopts a **Modular Monolith** architecture with layered separation of concerns (Route -> Controller -> Service -> Model) and an embedded SQLite database running with Write-Ahead Logging (WAL) mode.

```mermaid
flowchart TD
    Client["Frontend Client (Web / Mobile)"]
    
    subgraph Gateway ["Express & Network Layer"]
        CORS["CORS & JSON Body Parser"]
        AuthMid["JWT Auth Middleware"]
        SocketServer["Socket.IO Server"]
    end

    subgraph CoreModules ["Core CRM Domain Modules"]
        AuthMod["Auth Module"]
        PipelineMod["Pipelines & Deals Module"]
        ContactMod["Persons & Organizations Module"]
        EmailMod["Email & Tracking Module"]
        CallMod["Twilio Calls Module"]
        CalendarMod["Calendar & Activities Module"]
        AIMod["AI Sales Copilot Module"]
        ImportMod["CSV Import Module"]
    end

    subgraph BackgroundJobs ["Background Cron & Queues"]
        TokenCron["OAuth Token Refresh Cron (Every 6h)"]
        ReminderCron["Calendar Reminder Cron (Every 1m)"]
        TrashCron["Trash Cleanup Cron (30 Days)"]
        ImapIdle["IMAP IDLE Worker"]
        GmailPush["Gmail Pub/Sub Push Watcher"]
    end

    subgraph DataStore ["Persistence & External Services"]
        SQLite[("SQLite Embedded Database (data.db - WAL Mode)")]
        GoogleAPI["Google APIs (Gmail & OAuth)"]
        MSGraph["Microsoft Graph (Outlook 365)"]
        TwilioAPI["Twilio Voice & TwiML Gateway"]
        GroqAPI["Groq AI & RunPod Serverless"]
    end

    Client -->|HTTP REST Requests| CORS
    Client <-->|WebSocket Real-time Events| SocketServer
    
    CORS --> AuthMid
    AuthMid --> CoreModules
    
    CoreModules --> SQLite
    BackgroundJobs --> SQLite

    EmailMod <--> GoogleAPI
    EmailMod <--> MSGraph
    CallMod <--> TwilioAPI
    AIMod <--> GroqAPI
    
    SocketServer <--> BackgroundJobs
    SocketServer <--> CoreModules
```

---

## 2. Data Model & Entity Relationships

The relational model ties deals, contacts, companies, communications, and activities together:

```mermaid
flowchart TD
    User["users (Sales Rep / Admin)"]
    Org["organisations (Company / Account)"]
    Person["persons (Contact Person)"]
    Pipeline["pipelines (Sales Process)"]
    Stage["pipeline_stages (Ordered Steps)"]
    Deal["deals (Sales Opportunity)"]
    DealActivity["deal_activities / activities (Tasks, Calls, Notes)"]
    EmailAccount["email_accounts (IMAP / Gmail OAuth)"]
    Email["emails (Synced Messages)"]
    Call["calls (Twilio Voice Call Logs)"]
    Calendar["calendar_events (Meetings)"]

    User -->|Owns| Org
    User -->|Owns| Person
    User -->|Manages| Pipeline
    User -->|Assigned / Owns| Deal
    User -->|Connects| EmailAccount

    Org -->|Has many| Person
    Org -->|Associated with| Deal
    Person -->|Associated with| Deal

    Pipeline -->|Contains in sequence| Stage
    Stage -->|Current step for| Deal

    Deal -->|Tracks| DealActivity
    Deal -->|Linked to| Email
    Deal -->|Linked to| Call

    User -->|Schedules| Calendar
    Person -->|Invited to| Calendar
    Deal -->|Associated with| Calendar
```

---

## 3. Authentication & Authorization Flow

Pipeclose uses bcrypt password hashing and JWT authentication, backed by email-based One-Time Password (OTP) verification for registration and secure operations.

```mermaid
flowchart TD
    User([User / Browser])
    AuthCtrl["AuthController"]
    AuthSvc["AuthService"]
    EmailSvc["EmailService"]
    UserDB[("SQLite: users / otps")]
    ClientStorage[("Client LocalStorage")]

    %% Registration Flow
    User -->|"1. POST /api/auth/register"| AuthCtrl
    AuthCtrl -->|"registerUser(payload)"| AuthSvc
    AuthSvc -->|"bcrypt.hash(password, 10)"| AuthSvc
    AuthSvc -->|"INSERT INTO users (isVerified = 0)"| UserDB
    AuthSvc -->|"Generate 6-digit OTP"| AuthSvc
    AuthSvc -->|"INSERT INTO otps (expires in 10m)"| UserDB
    AuthSvc -->|"sendVerificationEmail(email, otp)"| EmailSvc
    EmailSvc -->|"Deliver OTP email"| User
    AuthCtrl -->|"201 Created (Pending Verification)"| User

    %% Verification Flow
    User -->|"2. POST /api/auth/verify-otp"| AuthCtrl
    AuthCtrl -->|"verifyOtp(email, otp)"| AuthSvc
    AuthSvc -->|"SELECT * FROM otps WHERE email"| UserDB
    UserDB -->|"Return OTP record"| AuthSvc
    
    CheckOtp{"Is OTP valid & not expired?"}
    AuthSvc --> CheckOtp
    
    CheckOtp -- Yes --> MarkVerified["UPDATE users SET isVerified = 1"]
    MarkVerified --> GenJWT["Generate JWT Token (userId, role)"]
    GenJWT --> SaveToken["Return 200 OK + JWT"]
    SaveToken -->|"Store Bearer Token"| ClientStorage
    
    CheckOtp -- No --> ReturnErr["Return 400 Bad Request (Invalid/Expired OTP)"]
    ReturnErr --> User

    %% Login Flow
    User -->|"3. POST /api/auth/login"| AuthCtrl
    AuthCtrl -->|"login(email, password)"| AuthSvc
    AuthSvc -->|"SELECT * FROM users WHERE email"| UserDB
    UserDB -->|"Return user record"| AuthSvc
    
    CheckPass{"bcrypt.compare(password, hash)?"}
    AuthSvc --> CheckPass
    CheckPass -- Password Matches --> GenLoginJWT["Generate JWT Token"]
    GenLoginJWT -->|"200 OK + Token"| User
    CheckPass -- Password Mismatch --> Unauthorized["401 Unauthorized"]
    Unauthorized --> User
```

---

## 4. Sales Pipeline & Deal Management Flow

Pipelines model customizable sales processes. Deals move across stages with automatic win probability calculations, stage change audits, and rotten deal detection.

```mermaid
flowchart TD
    Client([Sales Representative])
    DealCtrl["DealController"]
    DealSvc["DealService"]
    PipelineModel["Pipeline & Stage Models"]
    DealModel["DealModel"]
    HistoryModel["DealHistoryModel"]
    DB[("SQLite: deals, deal_history")]

    %% Create Deal
    Client -->|"POST /api/deals"| DealCtrl
    DealCtrl -->|"createDeal(data, userId)"| DealSvc
    DealSvc -->|"Validate stageId belongs to pipeline"| PipelineModel
    DealSvc -->|"INSERT INTO deals (status = 'OPEN')"| DealModel
    DealModel --> DB
    DealSvc -->|"INSERT INTO deal_history (CREATED event)"| HistoryModel
    HistoryModel --> DB
    DealCtrl -->|"201 Created (Deal Object)"| Client

    %% Move Stage / Drag and Drop
    Client -->|"PUT /api/deals/:id/stage"| DealCtrl
    DealCtrl -->|"changeStage(dealId, newStageId)"| DealSvc
    DealSvc -->|"Fetch stage attributes"| PipelineModel
    PipelineModel -->|"Stage config (isWon, isLost, probability)"| DealSvc

    CheckStage{"Stage Type?"}
    DealSvc --> CheckStage

    CheckStage -- isWon == 1 --> MarkWon["SET status='WON', actualCloseDate=NOW()"]
    CheckStage -- isLost == 1 --> MarkLost["SET status='LOST', lostReason=..."]
    CheckStage -- In Progress --> SetProg["SET stageId=newStageId, probability=stage.probability"]

    MarkWon --> RecordHistory["INSERT INTO deal_history (oldStageId, newStageId)"]
    MarkLost --> RecordHistory
    SetProg --> RecordHistory

    RecordHistory --> ResetActivity["UPDATE deals SET lastActivityAt=NOW(), isRotten=0"]
    ResetActivity --> DB
    DealCtrl -->|"200 OK (Updated Deal)"| Client

    %% Rotting Subgraph
    subgraph RottingLogic ["Deal Rotting Evaluation"]
        CalcTime["Calculate Inactivity: NOW - lastActivityAt"]
        CheckRotten{"Days Inactive >= stage.rottenDays?"}
        MarkRotten["UPDATE deals SET isRotten = 1"]
        KeepFresh["Keep isRotten = 0"]

        CalcTime --> CheckRotten
        CheckRotten -- Yes --> MarkRotten
        CheckRotten -- No --> KeepFresh
    end
```

---

## 5. Two-Way Email Sync & Instant Notification Flow

The email module synchronizes mail via IMAP or Google/Microsoft OAuth, listens for incoming emails in real-time, extracts deal associations, and pushes alerts through WebSockets.

```mermaid
flowchart TD
    subgraph EmailSources ["External Mail Sources"]
        IMAPServer["IMAP Mail Server"]
        Gmail["Google Gmail API / PubSub"]
    end

    subgraph SyncServices ["Sync Services & Parsers"]
        ImapIdle["IMAP IDLE Service (TCP Socket)"]
        GmailPush["Gmail Webhook (/api/webhooks/email)"]
        EmailConnector["EmailConnectorService"]
        Parser["mailparser (MIME Parsing)"]
    end

    subgraph Storage ["SQLite Storage"]
        EmailDB[("emails, email_accounts")]
        DealDB[("deals, deal_activities")]
    end

    subgraph RealtimeAndAI ["Real-Time & AI Features"]
        SocketServer["Socket.IO Server"]
        Summarizer["Groq / RunPod AI Summarizer"]
        Dashboard["Frontend User Dashboard"]
    end

    IMAPServer -->|"New message alert (IMAP IDLE)"| ImapIdle
    Gmail -->|"Push webhook notification"| GmailPush

    ImapIdle -->|"fetchNewMessages()"| EmailConnector
    GmailPush -->|"syncGmailAccount()"| EmailConnector

    EmailConnector -->|"Parse raw RFC822 message"| Parser
    Parser -->|"Parsed email & attachments"| EmailConnector

    EmailConnector -->|"INSERT message (dedup by messageId)"| EmailDB

    EmailConnector -->|"Lookup deal by sender/recipient email"| DealDB
    DealDB -->|"Link email to deal & refresh lastActivityAt"| DealDB

    EmailConnector -->|"Trigger background thread summary"| Summarizer
    Summarizer -->|"UPDATE email thread summary"| EmailDB

    EmailConnector -->|"emit 'new_email' (dealId, message)"| SocketServer
    SocketServer -->|"WebSocket notification"| Dashboard
```

---

## 6. Telephony & WebRTC Calling Flow (Twilio)

The calls module enables browser-based outbound calling via Twilio Voice SDK, handles live call webhooks, stores dual-channel audio recordings, and pushes status updates.

```mermaid
flowchart TD
    BrowserUser([Sales Rep in Browser])
    TwilioSDK["Twilio Client SDK (WebRTC)"]
    CallCtrl["CallController"]
    WebhookCtrl["WebhookController"]
    CallSvc["CallService"]
    TwilioAPI["Twilio Voice Cloud"]
    CustomerPhone([Customer Phone / PSTN])
    SocketServer["Socket.IO Server"]
    CallDB[("SQLite: calls")]

    %% Token Flow
    BrowserUser -->|"1. GET /api/calls/token"| CallCtrl
    CallCtrl -->|"generateCapabilityToken(userId)"| CallSvc
    CallSvc -->|"Return Twilio Voice JWT"| BrowserUser

    %% Dial Flow
    BrowserUser -->|"2. Initiate Call (WebRTC)"| TwilioSDK
    TwilioSDK -->|"Voice Connection Request"| TwilioAPI
    TwilioAPI -->|"3. POST /api/webhooks/twilio/voice"| WebhookCtrl
    WebhookCtrl -->|"handleOutboundVoiceWebhook(From, To, CallSid)"| CallSvc
    CallSvc -->|"INSERT call (status='initiated', direction='outbound')"| CallDB
    CallSvc -->|"Return TwiML with Dial & Recording config"| TwilioAPI

    %% Outbound Leg
    TwilioAPI -->|"PSTN Dial Call"| CustomerPhone
    TwilioAPI -->|"4. POST /api/webhooks/twilio/status"| WebhookCtrl
    WebhookCtrl -->|"updateCallStatus(CallSid, status)"| CallSvc
    CallSvc -->|"UPDATE status='ringing' / 'in-progress'"| CallDB
    CallSvc -->|"emit 'call_status'"| SocketServer
    SocketServer -->|"Push live status"| BrowserUser

    %% Call Finish & Recording
    CustomerPhone -->|"Call Ended"| TwilioAPI
    TwilioAPI -->|"5. POST /api/webhooks/twilio/status (completed)"| WebhookCtrl
    WebhookCtrl -->|"finalizeCall(CallSid, duration)"| CallSvc
    TwilioAPI -->|"6. POST /api/webhooks/twilio/recording (recordingUrl)"| WebhookCtrl
    WebhookCtrl -->|"attachRecordingUrl(CallSid, url)"| CallSvc
    CallSvc -->|"UPDATE calls SET status='completed', recordingUrl=url"| CallDB
    CallSvc -->|"emit 'call_completed'"| SocketServer
    SocketServer -->|"Call completed & recording available"| BrowserUser
```

---

## 7. Calendar & Reminder Processing Flow

The calendar module allows reps to organize meetings and tasks. A background cron worker checks every 60 seconds for due reminders and dispatches notifications via WebSockets or email.

```mermaid
flowchart TD
    User([User / Browser])
    CalCtrl["CalendarController"]
    CalSvc["CalendarService"]
    CalDB[("SQLite: calendar_events, reminders, notifications")]
    CronWorker["Cron: reminderProcessor (Runs every 1m)"]
    Dispatcher["NotificationDispatcherService"]
    SocketServer["Socket.IO Server"]
    EmailSvc["EmailService"]

    %% Scheduling
    User -->|"POST /api/calendar/events"| CalCtrl
    CalCtrl -->|"createEvent(payload, userId)"| CalSvc
    CalSvc -->|"INSERT INTO calendar_events"| CalDB
    CalSvc -->|"Compute triggerTime (startTime - offset)"| CalSvc
    CalSvc -->|"INSERT INTO event_reminders (status='PENDING')"| CalDB
    CalCtrl -->|"201 Created"| User

    %% Cron Processing
    CronWorker -->|"SELECT * FROM event_reminders WHERE triggerTime <= NOW()"| CalDB
    CalDB -->|"Due reminders list"| CronWorker
    CronWorker -->|"dispatchReminder(reminder)"| Dispatcher
    Dispatcher -->|"Fetch event & user details"| CalDB

    CheckChannel{"Notification Channel?"}
    Dispatcher --> CheckChannel

    CheckChannel -- In-App Push --> PushAlert["emit 'calendar_reminder'"]
    PushAlert --> SocketServer
    SocketServer -->|"Instant popup alert"| User

    CheckChannel -- Email Reminder --> MailAlert["sendReminderEmail()"]
    MailAlert --> EmailSvc
    EmailSvc -->|"Deliver notification email"| User

    PushAlert --> UpdateReminder["UPDATE event_reminders SET status='SENT', sentAt=NOW()"]
    MailAlert --> UpdateReminder
    UpdateReminder -->|"INSERT INTO event_notifications"| CalDB
```

---

## 8. AI Sales Agent & Copilot Flow

The AI Agent acts as an in-context sales copilot. It synthesizes client history, brand guidelines, and product pricing models to recommend the next best action or draft replies using Groq LLMs.

```mermaid
flowchart TD
    Rep([Sales Representative])
    AICtrl["SuggestionController"]
    Orchestrator["SuggestionOrchestratorService"]

    subgraph ContextEngine ["Context Extraction Engine"]
        ClientProfileModel["ClientProfileModel"]
        BrandModel["BrandGuidelinesModel"]
        PricingModel["PricingModel"]
        HistoryModel["Email & Activity History"]
    end

    GroqService["GroqApiService (LLM Inference)"]
    QA["QualityAssuranceService"]
    AIDB[("SQLite: ai_suggestions")]

    Rep -->|"POST /api/ai/suggest (dealId)"| AICtrl
    AICtrl -->|"generateNextBestAction(dealId)"| Orchestrator

    Orchestrator -->|"Fetch client pain points & profile"| ClientProfileModel
    Orchestrator -->|"Fetch company tone & communication rules"| BrandModel
    Orchestrator -->|"Fetch applicable tiers & discount limits"| PricingModel
    Orchestrator -->|"Fetch last 5 emails & call notes"| HistoryModel

    ClientProfileModel --> Orchestrator
    BrandModel --> Orchestrator
    PricingModel --> Orchestrator
    HistoryModel --> Orchestrator

    Orchestrator -->|"Synthesize prompt with context & system prompt"| Orchestrator
    Orchestrator -->|"Chat Completion request (LLaMA-3 / Mixtral)"| GroqService
    GroqService -->|"Generated suggestion & draft response"| Orchestrator

    Orchestrator -->|"Validate output against safety & pricing rules"| QA
    QA -->|"Approved suggestion payload"| Orchestrator

    Orchestrator -->|"INSERT INTO ai_suggestions (status='PENDING')"| AIDB
    AICtrl -->|"200 OK (Action Suggestion + Email Draft)"| Rep
```

---

## 9. Bulk Data Import Flow

The data import module supports importing contacts, organizations, and leads via CSV or spreadsheet files with field mapping and transactional batch inserts.

```mermaid
flowchart TD
    Admin([User / Admin])
    ImportCtrl["ImportController"]
    ImportSvc["ImportService"]
    Processors["Person & Organization Processors"]
    ImportDB[("SQLite: imports, persons, organisations")]

    %% Upload
    Admin -->|"1. POST /api/import/upload (CSV / Excel)"| ImportCtrl
    ImportCtrl -->|"parseFileHeadersAndPreview(file)"| ImportSvc
    ImportSvc -->|"Read header row & sample 5 rows"| ImportSvc
    ImportCtrl -->|"Return detected columns & preview data"| Admin

    %% Execute
    Admin -->|"2. POST /api/import/execute (importId, fieldMappings)"| ImportCtrl
    ImportCtrl -->|"processImport(importId, fieldMappings)"| ImportSvc
    
    ImportSvc -->|"Begin SQLite Transaction"| ImportDB

    ImportSvc -->|"Iterate rows with field mapping"| Processors
    Processors -->|"Validate required fields & formats"| Processors

    CheckRow{"Valid Record?"}
    Processors --> CheckRow

    CheckRow -- Valid --> InsertEntity["INSERT INTO persons / organisations"]
    InsertEntity --> ImportDB
    InsertEntity --> IncSuccess["incrementSuccessCount()"]

    CheckRow -- Invalid --> LogError["Record row error details"]
    LogError --> IncError["incrementErrorCount()"]

    IncSuccess --> CheckMore{"More rows?"}
    IncError --> CheckMore

    CheckMore -- Yes --> Processors
    CheckMore -- No --> CommitTx["Commit SQLite Transaction"]
    CommitTx --> Finalize["UPDATE imports SET status='COMPLETED'"]
    Finalize --> ImportDB
    ImportCtrl -->|"200 OK (Import stats & error report)"| Admin
```

---

## 10. API Route Reference

Below is a consolidated reference of the primary REST endpoints exposed by the server:

| Group | Route Prefix | Key Endpoints | Description |
| :--- | :--- | :--- | :--- |
| **Auth** | `/api/auth` | `POST /register`, `POST /login`, `POST /verify-otp`, `GET /me` | User registration, authentication, and session handling |
| **Pipelines** | `/api/pipelines` | `GET /`, `POST /`, `PUT /:id`, `DELETE /:id` | Sales pipelines configuration and ordering |
| **Deals** | `/api/deals` | `GET /`, `POST /`, `PUT /:id`, `PUT /:id/stage`, `DELETE /:id` | Sales opportunities, stage progression, and status updates |
| **Contacts** | `/api/persons` | `GET /`, `POST /`, `PUT /:id`, `DELETE /:id` | Individual customer contacts management |
| **Companies**| `/api/organisations`| `GET /`, `POST /`, `PUT /:id`, `DELETE /:id` | Organization profiles and linked company data |
| **Emails** | `/api/emails` | `GET /`, `POST /send`, `GET /threads`, `GET /accounts` | Multi-account email sync, composition, and threads |
| **Drafts** | `/api/email/drafts` | `GET /`, `POST /`, `PUT /:id`, `DELETE /:id`, `POST /:id/send` | Email composer drafts management |
| **Calls** | `/api/calls` | `GET /token`, `GET /logs`, `POST /call` | Twilio WebRTC token generation and call log access |
| **Webhooks** | `/api/webhooks` | `POST /twilio/voice`, `POST /twilio/status`, `POST /email` | External event ingestion from Twilio and Gmail Pub/Sub |
| **Calendar** | `/api/calendar` | `GET /events`, `POST /events`, `PUT /events/:id`, `DELETE /events/:id` | Event scheduling, reminders, and calendar sharing |
| **AI Copilot**| `/api/ai` | `POST /suggest`, `GET /config`, `PUT /config` | AI-assisted sales suggestions, brand guidelines, and pricing |
| **Import** | `/api/import` | `POST /upload`, `POST /execute`, `GET /status/:id` | Bulk CSV/Excel contact and deal ingestion |
| **Health** | `/api/health` | `GET /` | System health check and uptime status |
