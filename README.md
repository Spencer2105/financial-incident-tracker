# Financial Incident Tracker

Google Apps Script web application for registering, tracking, and escalating financial incidents (penalties and losses) within a bank's Internal Control and Non-Financial Risk area.

Built during a Systems Engineering internship at BBVA Peru.

> The full source is not public — this repository shows the architecture and patterns used.

## What it does

- **Incident registration** — data entry with metadata: area, amount, category (penalty or loss), stage, responsible parties
- **Multi-stage escalation** — workflow across Finance → Legal → Management, with stage-specific required fields
- **Linked records** — purchase orders and contracts attached per incident
- **Dashboard** — real-time view of open incidents, pending amounts, and deadlines by area
- **Email notifications** — automated alerts on stage transitions using Gmail API

## Why Google Apps Script

The organization already ran its core processes on Google Workspace (Sheets, Drive, Gmail). Apps Script was the zero-infrastructure choice: no server to provision, auth is handled by Google OAuth, and data lives in Sheets with Row-Level access. The tradeoff is the V8 runtime limitations and the 6-minute execution quota.

## Architecture

The codebase follows a layered pattern adapted for the Apps Script runtime, where all files share the same global scope:

```
Layer 0 — Utilities and base classes
Layer 1 — Data models (plain objects with typed fields)
Layer 2 — External service wrappers (Drive, Gmail)
Layer 3 — Repository layer (read/write via SpreadsheetApp)
Layer 4 — Service layer (business logic, validation, orchestration)
Layer 5 — HTML pages (served via HtmlService)
Layer 6 — Reusable UI components (included via HtmlService.createHtmlOutputFromFile)
Layer 7 — Controllers (doGet routing, JSON API endpoints)
Layer 8 — Config and workflow state machines
```

Files are numbered by layer (`1_IncidenciaModel.js`, `4_ServicioIncidencia.js`, etc.) so the dependency order is visually obvious in the IDE.

## Data flow

```
User fills form (Layer 5/6)
    ↓ fetch() call
Controller handles route (Layer 7)
    ↓
Service validates + applies business rules (Layer 4)
    ↓
Repository reads/writes Sheets (Layer 3)
    ↓
External services fire if needed (Layer 2): Drive attachment, Gmail alert
```

## Key patterns

See [`docs/patterns.md`](docs/patterns.md) for annotated code samples of:
- How models are defined (plain objects, no classes — Apps Script V8 limitation workaround)
- Repository pattern over Google Sheets (treating a Sheet as a table)
- Service layer validation and multi-stage state machine
- HTML include pattern for reusable UI components
- Session management without cookies (PropertiesService)

## Stack

| Layer | Technology |
|---|---|
| Runtime | Google Apps Script (V8) |
| Data store | Google Sheets |
| File storage | Google Drive |
| Email | Gmail API (via GmailApp) |
| Frontend | HTML Service + vanilla JS |
| Auth | Google OAuth (built-in) |
