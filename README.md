# Gmail Semantic Organizer — n8n + OpenAI

A three-workflow system that classifies Gmail inbox messages daily using GPT-4o-mini, applies labels automatically, and continuously improves classification accuracy through per-sender contextual memory.

Built and deployed as a personal productivity tool. Running in production.

## System overview

```
07:00  Email Organization        → Fetches, classifies, and labels emails
       Sub - Acciones Gmail      → Called by main: applies labels + updates memory  
00:00  Organization Mail Learning → Consolidates and prunes historical memory
```

The system gets more accurate over time: every manual label correction feeds back into the memory, and the learning workflow keeps that memory clean and relevant.

## Workflows

### 1. Email Organization (daily at 7am)

Main pipeline. Processes the last 50 unclassified emails:

1. Clears the pending queue from the previous run
2. Fetches the sender whitelist from Google Sheets
3. Fetches 50 unclassified Gmail messages
4. **Whitelist check** — whitelisted senders skip AI and get labeled `FAMILY/IMPORTANT` immediately
5. **Size filter** — emails over 500KB are marked `AMBIGUOUS` without AI processing
6. **HTML cleaning** — strips tags, encoded entities, and decodes Base64/QP headers
7. **Memory injection** — loads per-sender classification history and builds a contextual prompt
8. **GPT-4o-mini classification** — structured JSON output with category, confidence, and reasoning
9. **PDF pipeline** — if the model signals `necesita_pdf: true`, downloads the attachment, extracts text, rebuilds the prompt, and re-classifies
10. Calls `Sub - Acciones Gmail` to apply labels and update memory
11. Sends a **daily summary email** with per-category and per-sender breakdown
12. Sends an **alert email** if any emails were lost during processing

### 2. Sub - Acciones Gmail (called by main workflow)

Handles the action layer:

- Routes each email to the correct Gmail label based on category
- Checks whether a memory entry for this sender/category already exists
- Appends a new memory record or updates the existing one
- Flags entries that were manually corrected (`is_manual: true`) so they get priority in future classifications

### 3. Organization Mail Learning (daily at midnight)

Keeps the memory lean and accurate:

- Loads the full historical memory from Google Sheets
- Checks whether each entry still matches the current Gmail label (detects manual corrections)
- For each sender/category combination, keeps only the most recent relevant entries
- Deletes stale or redundant records to prevent memory bloat

## Classification categories

| Category | Description |
|----------|-------------|
| `EVENTO_DE_PAGO` | Invoices, receipts, bank notifications |
| `IA_Y_EMPRENDIMIENTO` | Technical/strategic AI and entrepreneurship content |
| `TRABAJO_REMOTO` | Recruiting, remote work culture |
| `INVERSIONES` | Finance, markets, investment news |
| `SALUD/NATURAL` | Health, natural living, interior design |
| `RUIDO_COMERCIAL` | Marketing, spam, low-signal newsletters |
| `AMBIGUO` | Default when model confidence is below threshold |

## Architecture

```
Gmail (50 msgs)
    │
    ├─ Whitelist → FAMILY/IMPORTANT (no AI)
    ├─ Size filter (>500KB) → AMBIGUOUS (no AI)
    │
    └─ AI Classification Pipeline
           │
           ├─ Load sender memory (Google Sheets)
           ├─ Clean HTML + decode headers
           ├─ Build prompt with historical context
           ├─ GPT-4o-mini → { categoria, confianza, necesita_pdf, razonamiento_tecnico }
           │
           ├─ [if necesita_pdf] → Download PDF → Extract text → Re-classify
           │
           └─ Sub-workflow: Apply Gmail label + Update memory

Midnight: Learning workflow prunes and consolidates memory
```

## Key design decisions

**Contextual memory per sender:** Classification history is stored in Google Sheets per sender. Manually corrected labels are flagged and given priority over automatic classification in future runs.

**PDF-aware classification:** Rather than always downloading attachments (slow, expensive), the model first attempts classification from subject and body. Only if it signals insufficient confidence does the workflow download and extract the PDF.

**Pending queue with second pass:** Emails queued for PDF processing are stored in a `Pendientes` sheet and retried in the next loop iteration, preventing Gmail API rate limit issues.

**Memory consolidation:** A nightly job removes stale entries and keeps the memory focused, avoiding prompt bloat and classification drift over time.

**Processing validation:** The workflow tracks emails entering vs. emails processed. Any discrepancy triggers an HTML-formatted alert email.

**Cost efficiency:** `gpt-4o-mini` with body truncation at 2,500 characters. Estimated cost: ~$0.80 USD per 1,000 emails.

## Stack

- **n8n** — workflow orchestration
- **OpenAI API** — GPT-4o-mini for semantic classification
- **Gmail API** — message fetching, label management
- **Google Sheets** — whitelist, historical memory, pending queue

## Google Sheets structure

Three sheets in one workbook:

| Sheet | Purpose |
|-------|---------|
| `WhiteList` | Sender addresses that bypass AI classification |
| `MemoriaHistorica` | Per-sender classification history with manual correction flag |
| `Pendientes` | Queue for emails requiring PDF reprocessing |

## Setup

1. Import the three workflow files into your n8n instance:
   - `Email_Organization.json`
   - `Sub_-_Acciones_Gmail.json`
   - `Organization_Mail_Learning.json`
2. Connect Gmail and Google Sheets credentials in n8n
3. Create the Google Sheets workbook with the 3 sheets above
4. Update `documentId` references in all Google Sheets nodes to your workbook ID
5. Update Gmail label IDs to match your account labels
6. Activate all three workflows

## Output

Each processed email gets one Gmail label applied automatically. A daily summary email reports total emails processed, breakdown by category, breakdown by sender, and alerts on any processing discrepancy.
