# Facility Maintenance Triage & Alerting

An AI-driven workflow automation that ingests facility maintenance reports, classifies and prioritizes them using an LLM grounded in equipment-specific history (RAG), and routes urgent issues to real-time alerts while maintaining a complete, auditable log of every ticket.

Built in [n8n](https://n8n.io) with [Claude](https://www.anthropic.com/claude) (Haiku) as the reasoning engine.

**[Live dashboard →]([./dashboard/index.html](https://frogman263.github.io/facility-maintenance-triage/dashboard/))**

---

## What it does

1. A technician submits a maintenance issue through a simple intake form (equipment ID, reporter, issue description).
2. The system looks up that equipment's known history — spec notes, prior failure patterns, last service date — from a maintained knowledge base.
3. Claude classifies the ticket (category, urgency, one-line summary) and generates a recommended next action, grounded in that retrieved equipment history rather than the issue description alone.
4. A parsing check verifies the AI's response is well-formed before it's trusted. Malformed output is routed to a human-review queue instead of silently corrupting the record.
5. Every ticket — regardless of urgency — is logged to a permanent audit trail.
6. Critical and High urgency tickets trigger a Slack alert. Critical alerts page the channel (`@here`) and are tracked until acknowledged; High alerts are informational only.
7. When someone acknowledges a Critical alert (via a Slack emoji reaction), a webhook listener updates the ticket's status and records how long it took — no polling, no manual dashboard checks required.
8. A live dashboard pulls directly from the underlying data to show open items, severity distribution, and acknowledgment metrics in real time.

## Architecture

```
Google Form (intake)
      │
      ▼
Google Sheets Trigger
      │
      ▼
Retrieve Equipment Context  ──►  Equipment Knowledge Base (lookup by equipment ID)
      │
      ▼
Claude (Haiku) — classify + recommend action, grounded in retrieved context
      │
      ▼
Parse Check ──── malformed? ──► Needs Review log + Slack alert (human review queue)
      │ (well-formed)
      ▼
Maintenance Log (every ticket, permanent record)
      │
      ▼
Switch: Urgency
      ├── Critical ──► Slack (@here) ──► Escalation Tracker (status: Pending)
      ├── High ─────► Slack (informational)
      └── Low/Medium ─► (logged, no alert)

Separate workflow — Escalation Listener:
Slack reaction event ──► Webhook ──► match ticket by message timestamp ──► update status + acknowledgment time
```

## Key design decisions

- **Every ticket is logged before urgency is evaluated**, not just the urgent ones. The audit trail should be complete regardless of what gets escalated.
- **Failure handling is explicit, not assumed.** The AI's output is validated before being trusted — a malformed or incomplete response is routed to a human-review path rather than silently producing a blank or corrupted record.
- **Acknowledgment is event-driven, not polled.** Rather than a scheduled job repeatedly checking "has anyone responded yet," the system reacts immediately to a Slack event via a webhook — a materially different (and more scalable) architecture than timer-based polling.
- **RAG grounds the AI in real specifics.** Classification and recommended actions reference actual equipment history (rated thresholds, prior service dates, known failure patterns) rather than reasoning generically about the issue text alone.
- **Severity tiers get different treatment, not just different labels.** Critical alerts actively page the channel and are tracked to resolution; High alerts inform without demanding immediate action. Alert fatigue is a real operational cost.

## Stack

- **n8n** — workflow orchestration (two workflows: main triage pipeline + escalation listener)
- **Claude Haiku** (Anthropic API) — classification and recommendation reasoning
- **Google Sheets** — intake, equipment knowledge base, audit logs (chosen for this prototype's rapid iteration; see Limitations)
- **Slack** — alerting, and the acknowledgment signal via emoji reaction + Events API webhook
- **Vanilla HTML/CSS/JS** — the live dashboard, reading sheet data directly client-side, no backend

## Known limitations

This is a working prototype demonstrating the architecture, not a production deployment. Specifically:

- **Intake is a form, not real telemetry.** A production version would ingest from actual sensor/SCADA data or a CMMS, not manual form entry.
- **No retry/rate-limit handling** on the AI API call itself, and no monitoring of the workflow's own uptime.
- **Acknowledgment is manual** (a person reacting in Slack), not tied to an actual work-order or resolution system.
- **Google Sheets as a data layer** works well for a fast prototype but wouldn't be the right long-term choice at real scale or with concurrent write volume — a proper database would replace it in a production version.
- **Equipment knowledge base is manually maintained** rather than synced from a real asset management system.

## Repo contents

- `workflow.json` — exported n8n workflow (main triage pipeline)
- `escalation-listener.json` — exported n8n workflow (Slack acknowledgment listener)
- `dashboard/index.html` — the live dashboard (self-contained, no build step)

---

*Built to demonstrate hands-on fluency with agent/workflow orchestration tools, prompt engineering, and RAG concepts — the core toolset for AI Builder-type roles connecting AI to real business data and processes.*
