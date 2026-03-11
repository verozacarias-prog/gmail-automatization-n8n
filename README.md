# Gmail Semantic Organizer — n8n + OpenAI

Automated Gmail inbox classification system that runs daily, classifies incoming emails using GPT-4o-mini, and applies Gmail labels automatically — with contextual memory, PDF handling, and processing validation.

## What it does

Every morning at 7am, this workflow:

1. Fetches the last 50 unclassified Gmail messages
2. Applies a **sender whitelist** (Google Sheets) — whitelisted senders are immediately tagged as `FAMILY/IMPORTANT`
3. **Filters oversized emails** (>500KB) and marks them as `AMBIGUOUS` without processing
4. **Cleans HTML** from email bodies and strips encoded headers
5. **Loads historical memory** (per sender) from Google Sheets to improve classification consistency
6. Sends each email to **GPT-4o-mini** for semantic classification
7. If the model needs a PDF to classify confidently, **downloads and extracts the attachment** and re-runs classification
8. Queues emails that require PDF processing for a second pass
9. **Applies Gmail labels** via a sub-workflow
10. Sends a **summary report** by email — or an **alert** if processing discrepancies are detected

## Classification categories

| Category | Description |
|----------|-------------|
| `EVENTO_DE_PAGO` | Invoices, receipts, bank notifications |
| `IA_Y_EMPRENDIMIENTO` | Technical/strategic AI and entrepreneurship content |
| `TRABAJO_REMOTO` | Recruiting, remote work culture |
| `INVERSIONES` | Finance, markets, investment news |
| `SALUD/NATURAL` | Health, natural living, interior design |
| `RUIDO_COMERCIAL` | Marketing, spam, low-signal newsletters |
| `AMBIGUO` | Default when confidence < threshold |

## Architecture

```
Gmail (50 msgs)
    │
    ├─ Whitelist check (Google Sheets) → Label: FAMILY/IMPORTANT
    │
    ├─ Size filter (>500KB) → Mark AMBIGUOUS, skip AI
    │
    └─ AI classification pipeline
           │
           ├─ Load sender history (Google Sheets: MemoriaHistorica)
           ├─ Clean HTML body + decode headers
           ├─ Build contextual prompt with history
           ├─ GPT-4o-mini → Structured JSON output
           │     { categoria, confianza, necesita_pdf, razonamiento_tecnico }
           │
           ├─ If needs PDF → Download attachment → Extract text → Re-classify
           │
           ├─ Apply Gmail labels (sub-workflow)
           └─ Update historical memory

Daily report → Success email or Discrepancy alert
```

## Key design decisions

**Contextual memory per sender:** Classification history is stored in Google Sheets and injected into each prompt. If a sender was manually corrected before, that correction takes priority over automatic classification.

**PDF-aware pipeline:** The model signals `necesita_pdf: true` when the email body is insufficient to classify with confidence. The workflow then fetches the full attachment, extracts text, and re-runs classification — avoiding misclassification of invoices or contracts.

**Pending queue:** Emails requiring PDF processing are stored in a `Pendientes` sheet and processed in batches of 3 in a second loop, preventing Gmail API rate limit issues.

**Processing validation:** The workflow tracks total emails entering vs. total processed. Any discrepancy triggers an HTML-formatted alert email with per-category and per-sender breakdown.

**Cost efficiency:** Using `gpt-4o-mini` with body truncation at 2,500 characters. Estimated cost: ~$0.80 USD per 1,000 emails.

## Stack

- **n8n** — workflow orchestration
- **OpenAI API** — GPT-4o-mini for semantic classification
- **Gmail API** — message fetching and label management
- **Google Sheets** — whitelist, historical memory, pending queue

## Configuration (Google Sheets)

The workflow reads from a Google Sheets workbook with 3 sheets:

| Sheet | Purpose |
|-------|---------|
| `WhiteList` | Sender email addresses that bypass AI classification |
| `MemoriaHistorica` | Per-sender classification history with manual correction flag |
| `Pendientes` | Queue for emails pending PDF processing |

## How to use

1. Import `Email_Organization.json` into your n8n instance
2. Connect your Gmail and Google Sheets credentials
3. Create the Google Sheets workbook with the 3 sheets above
4. Update the `documentId` references in the Google Sheets nodes to your workbook ID
5. Update Gmail label IDs to match your Gmail account
6. Activate the workflow — it will run daily at 7am

## Output

Each processed email gets one Gmail label applied automatically. A daily summary email is sent with:
- Total emails processed vs. received
- Breakdown by category
- Breakdown by sender
- Alert if any emails were lost during processing
