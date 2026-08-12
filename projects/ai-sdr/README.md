# AI SDR

Production-ready AI system for automatic lead intake, qualification and routing for a marketing agency.

Built as part of the **AI Automation Career Roadmap**.

## Overview

AI SDR automatically processes incoming leads without manual workflow execution.

The system:

1. Receives a lead through an HTTP POST webhook.
2. Validates required fields.
3. Normalizes contact information.
4. Checks for duplicate leads.
5. Saves the lead to Supabase.
6. Qualifies the lead using OpenAI.
7. Assigns a qualification category and score.
8. Routes the lead according to its category.
9. Updates the lead status in Supabase.
10. Sends Telegram notifications when human action is required.
11. Returns a predictable JSON response to the source system.
12. Uses a separate error workflow for production failures.

## Tech Stack

* n8n
* OpenAI API
* Supabase / PostgreSQL
* Telegram Bot API
* HTTP Webhooks
* JSON

## Architecture

```text
HTTP POST
    ↓
Webhook
    ↓
Validate Required Fields
    ↓
Normalize Contacts
    ↓
Find Duplicate Lead
    ↓
Duplicate?
   ↙   ↘
 Yes    No
  ↓      ↓
409    Create Lead
         ↓
     OpenAI Qualification
         ↓
     Update Lead
         ↓
     Route by Category
       ↙ ↓ ↓ ↓ ↘
    Hot Warm Cold Insufficient Data Fallback
```

A separate error workflow handles production execution failures and sends technical alerts.

## AI Qualification

The AI evaluates the lead using a 0–100 scoring system based on:

* requested service;
* company or business context;
* budget;
* task/problem description;
* timeframe or urgency.

The model returns structured data:

```text
score
category
summary
reason
next_action
missing_data
```

The current implementation uses `gpt-4o-mini` with structured JSON Schema output.

## Lead Categories

### Hot

Score: **75–100**

Action:

* priority processing;
* lead status updated;
* urgent Telegram notification sent to sales.

### Warm

Score: **45–74**

Action:

* lead added to the standard sales queue;
* Telegram notification sent.

### Cold

Score: **20–44**

Action:

* lead saved for nurturing;
* no Telegram alert is generated to avoid unnecessary sales noise.

### Insufficient Data

Score: **0–19**

Action:

* status changed to `needs_information`;
* Telegram notification requests additional information.

### Manual Review

Fallback route for an unexpected AI category.

Action:

* status changed to `manual_review`;
* Telegram technical notification sent;
* lead preserved for human review.

## Data Storage

Supabase stores:

* original lead information;
* normalized email;
* normalized phone;
* source;
* budget;
* AI score;
* AI category;
* AI summary;
* AI reasoning;
* recommended next action;
* missing data;
* AI model;
* qualification timestamp;
* workflow status.

Lead data is stored before AI qualification so that a lead is not lost if a later processing stage fails.

## Duplicate Protection

Before creating a new lead, the workflow normalizes email and phone information and searches Supabase for an existing record.

Duplicate requests return HTTP `409`.

## API

### Method

```text
POST
```

### Content-Type

```text
application/json
```

The production webhook URL is intentionally not included in the public repository.

### Example Request

```json
{
  "name": "Maria Volkova",
  "company": "Retail Group",
  "email": "maria@example.com",
  "phone": "+79997654321",
  "service": "AI lead automation",
  "budget": 500000,
  "message": "Budget approved. Ready to start next week.",
  "source": "website"
}
```

### Example Successful Response

```json
{
  "success": true,
  "lead_id": "lead-id",
  "status": "qualified_hot",
  "category": "hot"
}
```

## Error Handling

Production errors are handled by a separate n8n Error Workflow.

When a production execution fails, the system sends a technical Telegram alert containing:

* workflow name;
* error message;
* last executed node;
* execution ID.

Failed production executions are retained for diagnostics.

## Testing

The following scenarios were tested successfully:

| Test                        | Result              |
| --------------------------- | ------------------- |
| Warm lead                   | Passed              |
| Hot lead                    | Passed              |
| Insufficient data           | Passed              |
| Production Webhook          | Passed              |
| Technical error             | Passed              |
| Recovery after forced error | Passed              |
| Fallback / manual review    | Not manually tested |

The fallback route is implemented as a defensive mechanism, but its controlled test was intentionally skipped.

## Production Status

**Status: Production Ready**

The workflow:

* is active;
* is published;
* works through the Production Webhook;
* does not require manual execution;
* stores and qualifies leads;
* routes working lead categories;
* sends required Telegram notifications;
* handles production errors.

A website is not required for the current implementation. Any future website or form can be connected by sending the required JSON payload to the Production Webhook.

## Repository Structure

```text
ai-sdr/
├── README.md
├── workflow/
│   └── AI-SDR.json
└── docs/
    └── AI-SDR-Production-Documentation.docx
```

## Security

The repository does not contain:

* production webhook URLs;
* API keys;
* OpenAI credentials;
* Supabase credentials;
* Telegram bot tokens.

Credentials remain stored inside n8n.

## Current Limitations

* No website or frontend is connected.
* The fallback/manual-review route has not undergone a controlled test.
* Cold leads intentionally do not trigger Telegram notifications.
* Production URLs and credentials are excluded from public documentation.

## Project Result

Built a production-ready AI SDR in n8n for automatic inbound lead processing, including webhook intake, validation, duplicate detection, Supabase storage, AI qualification, five routing scenarios, Telegram sales notifications and a dedicated production error workflow.

The system operates autonomously and can be connected to a website, form or another lead source through HTTP POST.
