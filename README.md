# Keisi8.github.io
Personal Learning Portfolio

# AI-Powered Support Ticket Triage & Routing

A portfolio project demonstrating an end-to-end automation workflow that uses a large language model to assist customer service agents in triaging and responding to HR software support tickets, while proactively alerting CX operations managers to recurring issues on a product module level. 

**Transparency note:** This project was built with AI assistance (Claude) for guidance, code suggestions, and troubleshooting. All workflow design decisions, problem definition, and conclusions are my own.

---

## Problem Statement

Customer service agents handling high volumes of support tickets spend significant time on manual triage by reading, categorising, and routing each ticket before any resolution begins. This creates mental overhead, slows response times, and makes it difficult to spot systemic issues early.

This project automates the triage layer and adds a pattern detection layer on top, shifting CX operations from reactive to proactive.

---

## Use Case Within a Company

This workflow sits between the ITSM system (e.g., Zendesk, Jira) and the customer service agent. When a ticket arrives:

- The **customer service agent (L1)** receives an AI-generated summary, troubleshooting steps, and a suggested customer reply before they have fully read the ticket.
- The **L2 engineering team** receives routed tickets with category tags, reducing back-and-forth between support and engineering.
- The **CX operations manager** receives automated alerts when a specific product module accumulates an unusual number of tickets within a short window, signalling a potential systemic issue that may require cross-functional discussion rather than a one-off fix.

---

## Assumptions

- The HR platform has four core modules: **Payroll**, **Absence Management**, **Onboarding**, and **Integrations**.
- Tickets are ingested via webhook, simulating a real ITSM integration.
- The LLM is given module context directly in the system prompt — in a production environment, this would be replaced by a retrieval layer or a pre-configured knowledge base.
- Google Sheets is used as a lightweight data store for demonstration purposes. A production setup would use a proper database.
- The spike detector threshold is set to 3 tickets from the same module within 60 minutes.

---

## Tools Used

| Tool | Purpose | Why |
|---|---|---|
| **n8n** | Workflow automation | Open-source, low-code, intuitive to use and fast to prototype with. Supports webhooks, HTTP requests, conditional logic, and scheduling out of the box |
| **Claude API** | LLM classification and response generation | Strong instruction-following and structured JSON output. Cost-effective for low-volume prototyping |
| **Google Sheets** | Ticket log and data store | Zero setup, shareable, and sufficient for demonstration purposes |
| **Gmail** | Alert delivery | Native n8n integration, quick to configure |
| **Hoppscotch** | Webhook testing | Browser-based API client, no installation required |

---

## Data Protection & Security Considerations

### Credential Management
API keys and OAuth tokens are stored as encrypted credentials within n8n's credential manager rather than hardcoded into workflow parameters. This ensures sensitive values are never exposed in the workflow JSON or version history. Google OAuth access was granted with the minimum required permissions, allowing access to specific file access rather than broad Drive access to following the principle of least privilege.

For a production environment, in a company context, the respective product and compliance guidelines, and restrictions would be used. 

### GDPR & Data Protection
This prototype was built using simulated ticket data only — no real customer information was processed at any point.

In a production deployment, the storage layer would be replaced by an enterprise database with its own security controls and compliance certifications. However, data protection responsibilities extend beyond storage infrastructure. Regardless of which database is used, the implementing organisation remains responsible for: defining a valid legal basis for automated processing, storing only data that is necessary, setting clear retention and deletion policies for ticket log data, and conducting a Data Protection Impact Assessment (DPIA) before going live.

Particular attention would be required around the LLM processing step. Ticket content, which may include employee names, salary details, or other personal data, is sent to an external API provider. This requires a Data Processing Agreement with the LLM provider and a clear assessment of whether any data is retained beyond the scope of the individual request. This consideration applies independently of what database is used. 

---

## Workflow 1 — AI Support Ticket Assistant

This workflow runs continuously in the background, triggered by incoming tickets.

**Step 1 — Webhook trigger**
Receives an incoming ticket as a JSON payload via HTTP POST. Fields: `ticket_id`, `subject`, `body`.

**Step 2 — Claude API call (HTTP Request node)**
Sends the ticket content to the Claude API with a system prompt providing HR platform module context. The prompt requests a structured JSON response containing:
- `category` — one of: payroll, absence, onboarding, integrations
- `urgency` — one of: high, medium, low
- `summary` — one sentence describing the issue
- `troubleshooting_steps` — array of 3–5 actionable steps for the agent
- `suggested_reply` — a professional, empathetic draft response to send to the customer

Authentication is handled via a stored n8n credential rather than an exposed API key in the node parameters.

**Step 3 — Code node (JavaScript)**
Strips markdown code fences from the Claude response and parses the output into clean JSON. Also retrieves the `ticket_id` from the original webhook payload to ensure it is carried through the pipeline.

**Step 4 — Google Sheets node**
Appends a new row to the ticket log with all fields, including a generated ISO timestamp. The timestamp is critical for the spike detector in Workflow 2.

---

## Workflow 2 — CX Spike Detector

This workflow runs on a schedule every 15 minutes and monitors the ticket log for module-level activity spikes.

**Step 1 — Schedule trigger**
Fires automatically every 15 minutes. 

**Step 2 — Google Sheets node**
Reads all rows from the ticket log, retrieving up to 100 rows.

**Step 3 — Code node (JavaScript)**
Filters tickets logged in the last 60 minutes, groups them by category, and identifies the module with the highest ticket concentration. Outputs:
- `shouldAlert` — boolean
- `count` — total recent tickets
- `topCategory` — module with the most tickets
- `topCategoryCount` — number of tickets in that module
- `topCategoryTickets` — comma-separated ticket IDs

**Step 4 — IF node**
Checks whether `shouldAlert` is true. Routes to Gmail on the true branch, does nothing on the false branch.

**Step 5 — Gmail node**
Sends a structured HTML alert email to the CX operations manager summarising the spike: which module is affected, how many tickets, and which ticket IDs are involved. The email includes a recommendation to assess whether the pattern indicates a process gap, documentation issue, or a product-level change requiring cross-functional discussion.

---

## Results

Both workflows were tested end-to-end with 13 simulated tickets across all four modules. The Claude API correctly classified category and urgency in all cases and generated Personio-specific troubleshooting steps and suggested replies. The spike detector successfully identified a payroll module spike (4 tickets within 60 minutes) and triggered the alert email.

---

## Reflections & Learnings

**What surprised me:** How intuitive and fast n8n is once you get the hang of it. The visual node structure makes it easy to reason about data flow, and most steps came together quickly once the core logic was clear.

**What I would do differently in a real company environment:** I would not instruct the LLM on platform modules directly in the system prompt. Instead, I would connect it to a retrieval layer or a pre-existing internal knowledge base — many companies are already building these as part of their customer service automation stack. This would make the suggestions more accurate and reduce maintenance overhead when the product changes.

**What this workflow solves:** It goes hand in hand with an automated customer service agent. If one does not already exist, it cuts manual triage work and reduces the mental load on customer service agents to diagnose and route tickets. The spike detector adds a strategic layer for CX operations managers to track recurring issues that may need deeper resolution beyond a one-off fix.

**Next iteration:** Further development of the alerts layer to extract richer insights — for example, trending issue summaries per module, week-on-week comparisons, or escalation recommendations. Connecting to a real ITSM system via its native API would be the first production step.

---

*Built with n8n · Claude API · Google Sheets · Gmail*
