# AI Invoice Automation — n8n + Gemini + Airtable

Production-ready, end-to-end invoice processing pipeline. Emails with PDF
invoices are ingested, parsed with Google Gemini, validated, deduplicated,
stored in Airtable, routed through a manual review + multi-department
approval workflow, and every step is written to an immutable audit trail.

> **Note on OCR:** per project constraints, this solution does **not** use any
> OCR engine and does **not** call the Claude API. Text is extracted directly
> from the PDF's text layer. If a PDF has no extractable text (i.e. it is a
> pure image/scan), the workflow does not attempt OCR — it fails safe,
> flags the invoice as `Manual Data Entry Required`, and notifies AP staff
> so a human can key it in. This is documented behavior, not a bug.

---

## 1. Architecture

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                WORKFLOW 1 — INTAKE                       │
                    └─────────────────────────────────────────────────────────┘
IMAP Trigger → Filter (PDF only) → Extract Metadata → Extract PDF Text
   → [Text present?] ──No──→ Flag "Manual Data Entry Required" → Airtable + Audit + Email AP team
        │Yes
        ▼
   Gemini 1.5/2.0 Flash (structured JSON extraction)
        ▼
   Parse & Clean JSON (Code node) → Validate Schema (Code node)
        ▼
   Duplicate Check (Airtable lookup: Vendor + Invoice Number + Date + Amount)
        ▼
   Cost Center Prediction (Code node, rule engine)
        ▼
   Vendor Master Validation (Airtable lookup)
        ▼
   Upload PDF to Google Drive (get shareable link) → Create Invoice record in Airtable
   (Status = Draft, Send for Approval = false)
        ▼
   Write Audit Trail entry
        ▼
   [Error at any step] → Retry (exponential backoff, max 3) → Dead-letter → Email AP team

                    ┌─────────────────────────────────────────────────────────┐
                    │              WORKFLOW 2 — APPROVAL (Cron 15 min)          │
                    └─────────────────────────────────────────────────────────┘
Cron → Airtable Search (Send for Approval=true AND Status=Draft)
     → Resolve department approver(s) → Send approval email(s) with signed
       Approve/Reject links (Webhook)
     → Webhook (Approve) ──→ Update Status=Fully Approved ──→ Audit + notify submitter
     → Webhook (Reject)  ──→ Update Status=Rejected        ──→ Audit + notify submitter
```

## 2. Tech Stack

| Layer          | Choice                                    |
|----------------|--------------------------------------------|
| Orchestration  | n8n (self-hosted or cloud)                 |
| LLM            | Google Gemini (`gemini-1.5-flash` / `gemini-2.0-flash`) via REST |
| PDF text       | n8n native `Extract From File` (PDF, text layer only — no OCR) |
| Database       | Airtable (Invoices, Vendors, Departments, AuditTrail) |
| File storage   | Google Drive (shareable link referenced from Airtable) |
| Email          | IMAP (inbound) + SMTP (outbound)           |
| Approvals      | Webhook (signed HMAC links)                |

## 3. Folder Structure

```
/README.md
/.env.example
/workflows/invoice_processing.json
/workflows/approval_workflow.json
/database/airtable_schema.md
/src/prompts/invoice_extraction_prompt.txt
/src/functions/geminiClient.js
/src/functions/cleanAndValidate.js
/src/functions/duplicateDetection.js
/src/functions/costCenterPredictor.js
/src/functions/vendorValidator.js
/src/functions/auditLogger.js
/src/functions/retryWithBackoff.js
/src/functions/approvalRouting.js
/sample/sample_invoice.txt
/tests/functions.test.js
/docs/API.md
```

## 4. Installation

1. Install n8n ≥ 1.60 (`npm install -g n8n` or Docker image `n8nio/n8n`).
2. Create an Airtable base using `database/airtable_schema.md`. Copy the
   Base ID and generate a Personal Access Token with `data.records:read`,
   `data.records:write`, `schema.bases:read` scopes.
3. Get a Gemini API key from Google AI Studio (https://aistudio.google.com/apikey).
4. Set up a mailbox for IMAP (dedicated inbox, e.g. `invoices@company.com`)
   and SMTP credentials for outbound notifications.
5. Create a Google Drive service account (or OAuth2 credential in n8n) and a
   dedicated folder for invoice PDFs.
6. Copy `.env.example` to `.env`, fill in every value, and load it into your
   n8n instance's environment (or n8n's credential store — do **not** hardcode
   secrets inside workflow JSON; every credential in the exported workflows
   references a *named credential*, not an inline key).
7. Import both files in `/workflows` via n8n UI → **Import from File**.
8. Open each Airtable / Gmail / SMTP / HTTP Request node and attach the
   matching n8n credential (created from your `.env` values).
9. Activate both workflows.

## 5. Environment Variables

See `.env.example` for the full list. Key ones:

| Variable | Purpose |
|---|---|
| `GEMINI_API_KEY` | Google Gemini API key |
| `GEMINI_MODEL` | `gemini-2.0-flash` (default) |
| `AIRTABLE_API_KEY` | Airtable PAT |
| `AIRTABLE_BASE_ID` | Airtable base ID |
| `IMAP_HOST/PORT/USER/PASSWORD` | Inbound mailbox |
| `SMTP_HOST/PORT/USER/PASSWORD` | Outbound mailbox |
| `APPROVAL_LINK_SECRET` | HMAC secret used to sign approve/reject links |
| `N8N_WEBHOOK_BASE_URL` | Public base URL n8n is reachable at |
| `GOOGLE_DRIVE_FOLDER_ID` | Destination folder for stored PDFs |

## 6. Running

- Send a test email with `sample/sample_invoice.txt` converted to PDF (or
  any PDF invoice) as an attachment to the monitored inbox.
- Watch the `invoice_processing` workflow execution in n8n's **Executions**
  tab.
- A new `Draft` record appears in the Airtable `Invoices` table.
- Open the record, verify/edit fields, tick a department, tick
  `Send for Approval`.
- Within 15 minutes (or trigger the approval workflow manually) an approval
  email is sent to the department approver. Clicking Approve/Reject updates
  the status and writes an audit entry.

## 7. Testing

`tests/functions.test.js` contains Jest unit tests for every pure JS
function used inside the n8n Code nodes (validation, cleaning, duplicate
detection, cost-center prediction, confidence scoring, audit logging,
retry backoff). Run with:

```bash
npm install --save-dev jest
npx jest tests/functions.test.js
```

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Workflow never triggers | IMAP credentials wrong / mailbox not IDLE-capable | Verify IMAP creds; some providers need "app passwords" |
| `Manual Data Entry Required` on every invoice | PDF is scanned/image-only | Expected — no OCR is used by design. Ask vendor to resend as digital PDF, or key data manually |
| Gemini returns non-JSON | Model added prose around the JSON | `cleanAndValidate.js` strips code fences/prose and re-parses; if it still fails the record is dead-lettered and flagged `AI Extraction Failed` |
| Duplicate not detected | Airtable formula field mismatch on Invoice Number formatting | Normalize invoice numbers (function does `.trim().toUpperCase()`) before comparison |
| Approval email never arrives | SMTP creds / department has no mapped approver | Check `Departments` table has an `Approver Email` set |
| Webhook approve link expired/invalid | HMAC secret mismatch or link reused | Links are single-use and signed with `APPROVAL_LINK_SECRET`; regenerate by re-triggering approval workflow |

## 9. Assumptions

- One inbox receives invoices from all vendors; sender is not pre-authenticated
  (any PDF attachment is processed and flagged for human review either way).
- "Store the original PDF" is implemented as: upload to Google Drive, store
  the shareable link as an Airtable Attachment field (Airtable attachments
  require a public URL rather than raw binary upload via API).
- Multi-department routing sends one approval email per selected department;
  the invoice is `Fully Approved` only once **all** selected departments
  approve, and `Rejected` if **any** department rejects (configurable in
  `approvalRouting.js`).
- Currency/amount fields are parsed as strings from the AI and normalized to
  numbers server-side; if parsing fails the field is kept null and an
  anomaly is recorded rather than guessed.
- "No OCR" is treated as a hard constraint — scanned invoices are routed to
  manual entry, not silently dropped.

## 10. Future Improvements

- Add Slack/Teams approval buttons as an alternative to email links.
- Move file storage to S3 with signed URLs and lifecycle policies.
- Add a per-vendor confidence-threshold override for auto-approval of
  trusted, low-risk vendors.
- Multi-currency conversion at approval time using a live FX rate API.
- Move retry/dead-letter bookkeeping into a dedicated Airtable `Errors`
  table with alerting to PagerDuty/Opsgenie.
