# n8n Cloud: Technical Reference — Memory, Loops, Wait Node, and PDF/Gmail Workflows

This document answers eight specific engineering questions about n8n Cloud's runtime behavior, based on the n8n official documentation, GitHub source code and issues, and verified community forum threads. Where a behavior is undocumented or contested, this is stated explicitly. The n8n Cloud version context is the 1.x / early 2.x series (the most recent relevant change in 2025–2026 being the Wait node fix in sub-workflows introduced in v2.0).

---

## 1. RAM limits by plan and how execution data is stored in memory

### RAM caps by plan

n8n Cloud enforces fixed per-instance RAM limits. These are not listed on a single pricing page; they are published in the Kubernetes deployment reference documentation and confirmed by the pricing FAQ and n8n community team and mentors:

- **Starter / free trial**: ~320 MiB RAM, 10 millicores CPU with burst, 5 concurrent executions, 2,500 executions/month
- **Pro (10k executions)**: ~640 MiB RAM, 20 millicores CPU with burst
- **Pro (50k executions)**: ~1,280 MiB RAM, 80 millicores CPU with burst
- A community moderator responding to an OOM issue on Starter expressed them as "Starter ~512 MB, Pro ~1 GB, Business higher", but the numbers in the official n8n documentation (320/640/1,280 MiB) are the authoritative reference values.
- n8n itself (the Node.js process, the editor, etc.) consumes on average ~180 MiB of RAM before your workflow adds anything, leaving a very small effective margin on Starter.
- Pro plan users have asked whether they can pay to upgrade RAM independently; this is currently not available in the product — the only path on Cloud is to upgrade the plan tier.

### How execution data is kept in memory

n8n's execution engine loads into RAM all items that have passed through the workflow for the entire duration of the execution. The official documentation states clearly that the instance "loads the complete data set into RAM during execution" and that the factors increasing memory usage are: amount of JSON, size of binary data, number of nodes, and whether the execution is manual (manual executions create an additional copy of the data for the UI).

Binary data in Cloud is kept in RAM by default — the `N8N_DEFAULT_BINARY_DATA_MODE=filesystem` / `s3` environment variables are only available on self-hosted instances, and external binary storage is an Enterprise feature.

### Does n8n retain ALL previous node outputs for the entire execution?

Yes. n8n keeps the input and output of every executed node available for expression resolution (`$('Node Name').item`, `$('Node Name').all()`, etc.) and for the editor's Input/Output panels, for the entire lifetime of a single execution. This is why adding more processing stages, more items, or running manually (which duplicates the in-memory snapshot for the UI) are all direct contributors to OOM errors.

When a manual execution is running, the browser UI also adds pressure — n8n documentation explicitly warns that "interacting with the workflow UI while it performs heavy executions could also push the memory capacity over the limit".

### Cloud storage limits (not RAM — database)

- Starter: 2,500 saved executions, 7-day retention
- Pro: 25,000 / 30 days
- Enterprise: 50,000 / unlimited

When disk usage reaches 85%, n8n Cloud automatically backs up and restores the instance without execution data.

---

## 2. Wait node + binary data in n8n Cloud

### Two modes, two very different memory profiles

The Wait node has an undocumented hard threshold of approximately **65 seconds** that determines its behavior:

- **Waits under ~65 seconds**: the execution **stays in memory** and the process simply sleeps. Data is NOT flushed to the database. n8n's Wait node documentation states clearly: "For wait times shorter than 65 seconds, the workflow does not offload execution data to the database. Instead, the process continues running and the execution resumes after the specified interval elapses."
- **Waits over ~65 seconds, webhook waits, or form waits**: n8n **serializes the complete execution state to the database**, terminates the in-process execution, and resumes from the DB when the condition is met. The Wait node documentation states: "When the workflow pauses, it offloads execution data to the database. When the resume condition is met, the workflow reloads the data and execution continues."

### Behavior with large binary data (Gmail attachments in base64, etc.)

There is no separate path for binary data: the Wait node serializes **whatever is in the execution payload** — both JSON items and binaries — to the DB, and then fully reloads them on resume. In n8n Cloud, binary data is kept in memory by default (filesystem/S3 modes are self-hosted only, and external S3 storage is Enterprise only).

Observed consequences:
- If the wait threshold is not carefully configured, an execution with a large payload can get stuck "waiting" and, on restart, attempt to rehydrate; multiple GitHub issues describe waiting executions that fail to resume or become deadlocked.
- On version upgrades, executions with a `Wait` node that had been offloaded with expressions referencing `$json` have failed on resume because the serialized context did not deserialize as expected — a recurring class of reported regressions.
- n8n 2.0 explicitly fixed a long-standing bug with sub-workflows + Wait: parent workflows now wait when a child sub-workflow pauses, instead of returning the pre-Wait input data.

**Practical implication**: placing a Wait node after a Gmail node with `downloadAttachments: true` and a large base64 PDF payload means the complete base64 blob (a) stays in memory for the first 65 seconds, (b) gets serialized to the DB on longer waits, and (c) gets reloaded back into RAM on resume. On a 320 MiB Starter or 640 MiB Pro, this is a well-documented failure mode.

---

## 3. SplitInBatches referencing external nodes (e.g. Google Sheets "Get History Memorial")

### How SplitInBatches (Loop Over Items) actually processes data

The Loop Over Items node "stores the original input data, and with each iteration, returns a predefined number of items through the loop output. When the node finishes executing, it combines all processed data and returns it through the done output."

In plain terms: the node keeps the **full original input array** anchored in memory for the entire life of the loop (so it can deliver the next batch), and also **accumulates processed items across iterations** so it can emit them on the Done branch when it finishes.

### What happens with nodes inside the loop (including the Google Sheets "Get History Memorial")

Every node inside the loop that runs once per iteration stores its **output from that run** in n8n's execution data structure. That structure is indexed by run index, meaning that when iteration 2 runs, the output of iteration 1 of "Get History Memorial" **is still in memory** — referenced as `run[0]` for that node. This is what makes expressions like `$('Get History Memorial').first()`, `.all()`, `.context[...]`, etc. work during and after the loop.

The n8n community has documented the symptom firsthand: "I have a workflow that produces a lot of information on each split in batches iteration. It visibly slows down (each iteration is slower than the previous); I trust it is saving all the accumulated data." — to which an n8n expert responded recommending **sub-workflows** (Execute Workflow) to free memory after each batch.

### Does n8n accumulate ALL data from previous iterations?

Yes. Every iteration of every node inside the loop remains resident in the execution data memory until the execution finishes (or fails). With 1,300 rows being read from Google Sheets on each iteration, that payload multiplies by the number of iterations.

---

## 4. Extract from File (PDF) with the `keepSource: json` option

### Official documentation of the option

The Extract From File node, in its PDF operation, exposes a **"Options → Keep Source"** parameter that controls what happens to the incoming item after extraction. The three possible values are **`json`**, **`binary`**, and **`both`** — corresponding respectively to: (a) retain only the original JSON and add the extracted text, (b) retain only the original binary and add the extracted text, (c) retain both.

### Exact semantics of `keepSource: json`

When `keepSource` is set to `json`:
- The fields extracted from the PDF (`text`, `numpages`, `info`, `metadata`, etc.) are added to the output item's JSON.
- The **original JSON fields** of the incoming item are **preserved**.
- The **original binary attachment is dropped** from the output item (not forwarded). This is the memory-saving option.

In contrast, `keepSource: binary` or `both` preserves the binary PDF in the output item, meaning the base64-encoded PDF travels with every subsequent node.

### Recommended pattern to avoid memory bloat

1. As soon as you have the extracted text, **use `keepSource: json`** (or the Extract From File default, which does not re-emit the binary unless explicitly requested).
2. If you need to keep the PDF further downstream, store it externally first (Google Drive / S3) and pass only a URL/ID in JSON rather than the binary.
3. In n8n Cloud you cannot change the binary mode to `filesystem` or `s3`, so your only lever is *not carrying the binary* beyond the extraction step.

---

## 5. Gmail `get message` with `simple: false` and `downloadAttachments: true`

### What the node actually stores

The Gmail node on the "Get Message" path (non-simple mode) uses `mailparser`'s `simpleParser` to decode the raw base64 MIME message and then — **for each attachment, if `downloadAttachments` is true** — calls `this.helpers.prepareBinaryData(attachment.content, filename, contentType)` and stores the result under binary keys `attachment_0`, `attachment_1`, etc. in the item. The resulting item has:
- **json**: headers, labels, IDs, size estimate, the parsed text/HTML body.
- **binary**: one entry per attachment (base64-encoded content).

### Memory footprint of a 500 KB PDF attachment

- A PDF travels inside the Gmail API response as base64. Base64 encoding expands the binary by ~4/3 (~33%). A 500 KB PDF is therefore ~667 KB of base64 text during parsing.
- n8n's `prepareBinaryData` method converts the attachment back into a **Buffer** (binary) for in-memory storage — in n8n Cloud that Buffer lives in RAM (default in-memory binary mode).
- **Effective RAM footprint per item**: approximately **1.1–1.5 MB** is a reasonable estimate for a 500 KB PDF after parsing — the Buffer itself (~500 KB) + the transient base64 string during decoding (~667 KB) + overhead from the parsed email JSON representation (headers, bodies) + any internal duplicate the engine maintains between execution data + the frontend copy if the workflow runs manually (duplicated).
- That ~1 MB is per item and stays for the entire execution. Running this 25 times inside a SplitInBatches loop before a Wait, and you're at ≥25 MB just from attachments, plus all other data — which together with the rest of the workflow memory easily exceeds the 320 MiB Starter limit.

Also important: `downloadAttachments: true` with `simplify: true` historically returns only the attachments and drops the content/body; to get both you need `simple: false` and read the binary keys directly.

---

## 6. Execution timeout on n8n Cloud and "Error in 42ms"

### Timeout behavior

- Globally (self-hosted), `EXECUTIONS_TIMEOUT` defaults to `-1` (unlimited) and can be overridden per workflow. In Cloud the environment variable is not user-configurable; instead "n8n enforces a maximum timeout available for each plan" which is set per workflow in the Workflow Settings dialog.
- n8n does not publish an explicit "maximum timeout per plan" number on a single table, but community threads with n8n staff historically confirm a conservative limit on Cloud. A widely cited 2026 discussion shows the Anthropic node in n8n Cloud v1.115.3 hitting a hard timeout of **300,000 ms (5 minutes)** at the HTTP layer, not the global execution layer — that is, per call, not per workflow.
- Code nodes running in Cloud use the "task runner" architecture and have a **separate 60-second task timeout**.
- The global workflow timeout is a **soft timeout**: if the workflow runs in the main process it triggers after the current node finishes; if it has its own process, n8n waits one fifth of the timeout and then kills the process.

### Can a workflow fail with "Error in 42ms"?

Yes. The node duration shown in the execution list is the time between its start and the error it throws. If a node throws synchronously (wrong parameters, missing binary field, invalid credential, bad expression, a throw inside the node body's `execute()`) it returns in tens of milliseconds — this is normal. Two well-documented classes of this:

- An uncaught `NodeApiError`/`NodeOperationError` from a custom or badly-built node can crash the entire n8n instance in under a hundred milliseconds, which appears externally as an almost-instant failure on a specific node.
- A misconfigured `Stop and Error` node with an empty `Error Message` parameter reliably fails in tens of ms and has historically crashed the Cloud container.

In other words, an "Error in 42ms" on a single node is perfectly normal when the node fails before contacting any external service — it is not a timeout artifact, it is an immediate node-level throw.

---

## 7. Best practices for loading large Google Sheets data inside a loop

The consensus from the official documentation, n8n team responses, and community is:

### Load reference data ONCE, before the loop

The `Google Sheets` node in "Get Rows" mode is expensive for large sheets because it loads the full header row (and, on append/update paths, often the entire sheet) to align columns. If you call Get Rows once per loop iteration with 1,300 rows, you are (a) paying the API call cost on every iteration (Google rate limits), and (b) duplicating that 1,300-row payload in execution data memory on every iteration, because n8n retains all outputs per execution.

### Recommended pattern

1. **Fetch the reference data once** (Get Rows) *before* the Loop Over Items node.
2. **Hold the reference set** using a Set/Edit Fields node, or — for truly large tables — write it to a lightweight lookup store (Supabase, Postgres, key-value) so each iteration can query by key instead of loading the whole table.
3. Inside each iteration, **reference** the pre-loaded data with `$('Get History Memorial').first().json` or a `Merge` node joined by key — n8n explicitly supports this "data from outside the loop" pattern (community posts show `$('Node Name').item`, `.first()`, `.all()` usage).
4. If the reference set is large enough to stress a single call, do the work inside an **Execute Workflow sub-workflow** so the reference load happens once per batch and is freed from memory when the sub-workflow returns a small result.

---

## 8. Does SplitInBatches accumulate memory across iterations?

### Empirical and documented answer: YES

- The Loop Over Items node "stores the original input data" and, **on the `done` branch, emits the combined processed data from all iterations**. To be able to emit the combined set on `done`, it must retain the output of every iteration.
- Every node placed inside the loop body stores its output by run index so that expressions like `$('Node Name').all()` work during and after the loop. This is a core engine design decision; it is why a visible per-iteration slowdown is observed and why the n8n team routinely directs users toward sub-workflows to free memory.
- Third-party technical posts describe the same behavior: "It keeps all execution data from every iteration in memory simultaneously, including the full node history from each step. In a loop processing 50,000 CRM records, that execution history grows exponentially until n8n runs out of RAM or the execution times out."

### What SplitInBatches does NOT do

- It does **not** delete data from previous iterations at the end of each iteration. There is no native purge/cleanup option. Community feature requests for exactly this ("SplitInBatches — Clear/Purge Cache at end of each Iteration") have been answered with the sub-workflow workaround, not a native configuration.

### The official memory-efficient pattern

From n8n's own documentation on memory errors:

> "Split the workflow into sub-workflows and make sure each sub-workflow returns a limited amount of data to its parent workflow. … the sub-workflow only keeps the current batch's data in memory, after which that memory is freed."

For a Gmail + PDF + Google Sheets + Wait workload on Cloud Starter/Pro, the memory-safe architecture is:

1. **Trigger** → **Loop Over Items (small batch size)** → **Execute Workflow** (sub-workflow that does the Gmail fetch, PDF extraction with `keepSource: json`, Google Sheets reference lookup, and writes the result) → **return only a small summary to the parent**.
2. Place any long Wait node *inside the sub-workflow so it only freezes the small per-batch payload*, or, where possible, replace long waits with a Cron/queue pattern that splits the execution entirely.
3. Pre-load reference data outside the loop; never re-read 1,300 rows per iteration.
4. Do not carry binary PDFs past the `Extract from File` step; use `keepSource: json` or re-upload to a storage bucket and reference by URL.

---

## Quick Reference Table

| # | Question | Short answer |
|---|---|---|
| 1 | Cloud RAM per plan? | ~320 MiB Starter / ~640 MiB Pro-10k / ~1,280 MiB Pro-50k; n8n itself uses ~180 MiB; full node history kept in RAM for entire execution |
| 2 | Wait node + binaries? | Waits <65s stay in memory; ≥65s/webhook/form serialize the full execution (including base64 binaries) to DB and reload on resume |
| 3 | SplitInBatches + Google Sheets inside loop? | Yes, outputs from every iteration remain in memory; visible per-iteration slowdown and eventual OOM on Cloud |
| 4 | Extract from File `keepSource: json`? | Keeps original JSON fields, adds extracted text, drops the binary PDF from the output item — the memory-saving option |
| 5 | Gmail `simple:false` + `downloadAttachments:true`? | Attachments stored as binary keys (`attachment_0`, …); a 500 KB PDF ≈ ~1 MB effective RAM per item |
| 6 | Cloud timeout and 42ms errors? | Per-plan cap (configurable in Workflow Settings), ~300s per external HTTP call, 60s for Code node task runner; instant errors (~42ms) are normal for node-level throws, not timeouts |
| 7 | Google Sheets reference data (1,300 rows) in a loop? | Load once before the loop and reference with `$('Node').first()` / Merge; for large sets, use a sub-workflow or external lookup (Postgres/Supabase) to avoid per-iteration re-fetch |
| 8 | SplitInBatches accumulation? | Confirmed: yes, processed data from all batches is retained for the `done` branch; the memory-safe pattern is Loop → Execute Sub-workflow returning a small summary |

---

## Conclusion for this specific symptom profile

A workflow combining **Gmail with `downloadAttachments: true`**, an **Extract-From-PDF** step that carries the binary forward, a **SplitInBatches loop** that also calls a **Google Sheets "Get Rows"** of ~1,300 reference rows **inside** the loop, and a **Wait node** on n8n Cloud Starter (~320 MiB) is hitting all three worst memory multipliers simultaneously:

1. Each iteration retains the Gmail + PDF + Sheets outputs from all previous iterations.
2. Each iteration reloads the 1,300-row reference set into per-execution memory.
3. The Wait node serializes all of that to disk and reloads it, and the Cloud instance cannot offload binaries to filesystem/S3.

The fix is **architectural, not configurational**, and aligns with n8n's own documented guidance: move the heavy work to an **Execute Workflow sub-workflow** (so memory is freed per batch), **pre-load reference data once** before the loop, configure **`keepSource: json`** on the Extract From File node, and (where possible) **avoid carrying the Gmail attachment binary past the point where it was extracted** — store a URL or discard the binary as soon as its text has been captured.
