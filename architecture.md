# Architecture Write-Up

## Overview

This solution is a minimal end-to-end AI intake and triage workflow for ArcVault built manually in `n8n`. The workflow uses a `Schedule Trigger`, a single Groq LLM call through an `HTTP Request` node, deterministic routing and escalation logic in an `n8n Code` node, and a final `outputs.json` artifact generated through `Convert to File`.

This write-up matches the real workflow that should be built in n8n. Codex generated the prompts, code snippets, and documentation, but it does not create or sync the n8n workflow automatically.

## Actual n8n Workflow

The workflow is:

1. `Schedule Trigger`
2. `Code: Load Sample Inputs`
3. `HTTP Request: LLM Classifier`
4. `Code: Parse LLM JSON`
5. `Code: Route And Escalate`
6. `Aggregate`
7. `Code: Build Outputs JSON`
8. `Convert to File`

There is no `Split Out` node in the final design because the `Code: Load Sample Inputs` node already emits one n8n item per synthetic request.
There is also no disk-write node in the final solution because `n8n` cloud would not write the file to local disk. The JSON artifact is produced in `Convert to File` and downloaded from the workflow UI.

## System Design

The workflow has three layers:

1. Ingestion
   - `Schedule Trigger` starts the workflow automatically.
   - `Code: Load Sample Inputs` emits the five synthetic customer requests as separate items.

2. AI classification and enrichment
   - `HTTP Request: LLM Classifier` sends each request to Groq using an OpenAI-compatible chat completions endpoint.
   - The model returns strict JSON only.
   - `Code: Parse LLM JSON` parses that JSON and combines it with the original request metadata.

3. Deterministic decisioning and persistence
   - `Code: Route And Escalate` applies queue routing and escalation logic using plain business rules.
   - `Aggregate` collects all final records into one array.
   - `Code: Build Outputs JSON` prepares the final JSON string.
   - `Convert to File` creates the downloadable `outputs.json` artifact.

State is kept in:

- transient workflow execution data inside n8n
- the downloaded `outputs.json` artifact used as the final deliverable

## Routing Logic

Routing is deterministic and intentionally lives outside the LLM:

- `Bug Report` -> `Engineering`
- `Incident/Outage` -> `Engineering`
- `Feature Request` -> `Product`
- `Billing Issue` -> `Billing`
- `Technical Question` -> `IT/Security` if the message contains identity, authentication, or security terms such as `sso`, `okta`, `auth`, `authentication`, `saml`, `oauth`, or `security`
- other `Technical Question` messages -> `Engineering`

This keeps the workflow easier to test and easier to justify during review. The model interprets the message; the workflow owns the operational routing decision.

## Escalation Logic

A record is escalated to `Human Review / Escalation` if any of the following is true:

- `confidence < 70`
- the message suggests outage or broad impact, such as `outage`, `multiple users affected`, `all users`, `system down`, `service down`, or `dashboard stopped loading`
- the billing discrepancy exceeds `$500`

This means a high-confidence model result can still be escalated when the business impact is high. That is intentional and closer to how a production support workflow should behave.

## Why This Design Is Minimal But Complete

I intentionally avoided:

- multiple LLM calls
- a database
- a message queue
- a separate validation service
- live external downstream systems

Those would add complexity without improving the score of a 3 to 5 hour take-home. This version is enough to show workflow functionality, prompt quality, system design thinking, and structured output quality.

## Known Limitations / Tradeoffs

- The workflow uses a synthetic input node rather than a live inbound integration.
- In `n8n` cloud, the output is downloaded manually from `Convert to File` rather than written automatically to local disk.
- The workflow uses a single LLM call and does not include a second-pass validation model.
- The design favors clarity and explainability over operational robustness.

## What I Would Improve In Phase 2

If I were extending this beyond the assessment, I would add:

- a real webhook, mailbox, or form trigger instead of scheduled synthetic input
- retry handling and dead-letter paths for failed model calls
- JSON schema validation before routing
- persistence to Google Sheets, Airtable, or a database for team visibility
- notifications for escalated items
- analytics on confidence, routing, and escalation trends

## Assumptions

- A scheduled batch over the five provided messages satisfies the assignment’s automatic trigger requirement.
- A downloadable JSON artifact is an acceptable structured output deliverable.
- The LLM output is advisory for classification and enrichment only; queue routing and escalation remain deterministic by design.
