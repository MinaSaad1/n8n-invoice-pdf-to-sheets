# Architecture

## High-level

```
Google Drive Trigger (specific folder, file created)
        │
        ▼
Download PDF (Google Drive node, binary out)
        │
        ▼
Convert to Base64 (Code node, binary to base64 string + filename + file_id)
        │
        ▼
Claude Extract Invoice (HTTP Request to api.anthropic.com/v1/messages with PDF document)
        │
        ▼
Parse Invoice JSON (Code node, strips fences and parses, fails safe to placeholder)
        │
        ▼
Check for Duplicate (Google Sheets read on Invoice Number)
        │
        ▼
Is Duplicate? (IF node)
        │
   ┌────┴─────┐
   ▼          ▼
duplicate   new
   │          │
Log Duplicate Warning  Append Invoice Row
(Google Sheets append) (Google Sheets append)
```

The workflow is built around a single round trip: Drive in, Claude extracts, Sheets out. No sub-workflows, no message queues, no manual OCR pass.

## Components

### Drive New Invoice (`Google Drive Trigger`)

Watches one specific folder for new files. Polls every 1 to 5 minutes (n8n default for the Drive trigger). The folder ID is the only input you set, the file ID flows downstream as `$json.id`.

The trigger is intentionally folder-scoped, not Drive-wide. That keeps the OAuth scope conversation simple ("read files in this one folder") and prevents the workflow from accidentally reacting to unrelated uploads elsewhere in your Drive.

### Download PDF (`Google Drive` node)

Pulls the file content as a binary attachment. The trigger gives you metadata only, this node fetches the actual bytes. Generic operation, no flavor-specific tuning needed.

### Convert to Base64 (`Code` node)

n8n binary data is base64-encoded internally already. This node pulls the encoded string out of the binary slot, attaches the filename and the original Drive file ID, and forwards them as a JSON payload. From here on out the workflow handles structured JSON only, no binaries.

The `file_id` stays attached so the Sheets row can link back to the source PDF in Drive. Useful for audits.

### Claude Extract Invoice (`HTTP Request` node)

Posts to `https://api.anthropic.com/v1/messages` with a `document` content block containing the base64 PDF and a single `text` block with the extraction prompt. The model is `claude-sonnet-4-6` by default, swap to `claude-sonnet-4-6` for cost-sensitive deployments.

The prompt asks for a strict JSON object with vendor, invoice number, dates, totals, currency, and line items. It also tells the model to treat any instructions inside the PDF text as data, not commands. That last line is a soft defense against prompt injection, see `SECURITY.md` for why it's not enough on its own.

### Parse Invoice JSON (`Code` node)

Strips `\`\`\`json` fences if Claude added them, parses the JSON, and falls back to a placeholder (`vendor_name: 'Parse Error', invoice_number: 'UNKNOWN', total: 0`) on parse failure. The placeholder is deliberate: a parse error still flows downstream so the duplicate check sees `UNKNOWN` and doesn't accidentally match a real invoice. You'll see the error rows pile up in the sheet and know to investigate.

It also stamps `processed_at` with the current ISO timestamp and re-attaches `file_name` and `file_id` from upstream.

### Check for Duplicate (`Google Sheets` read)

Reads the `Invoices` tab and filters by `Invoice Number == $json.invoice_number`. If a row matches, the IF node downstream routes to the duplicate logger. If no row matches, it routes to the appender.

This is the cheapest possible dedup, one read per invoice. It scales fine up to a few thousand rows per tab. Beyond that, switch to a database or move dedup into a Code node that hashes vendor+number+date.

### Is Duplicate? (`IF` node)

Single condition: `Invoice Number is not empty` on the upstream read. If the read returned a hit, the field is non-empty and the duplicate branch fires. Clean and explicit.

### Append Invoice Row and Log Duplicate Warning (`Google Sheets` append)

Twin nodes pointed at two different tabs of the same spreadsheet. The append maps every parsed field onto a column. The duplicate logger writes a smaller summary row into a `Duplicates` tab so AP staff have a place to look when the workflow blocks something.

## Design decisions worth calling out

### Why Claude Vision instead of a dedicated OCR service

A single API call replaces a multi-stage pipeline (OCR -> field detection -> normalization). For machine-generated PDFs it lands close to perfect accuracy. For scanned PDFs it sits around 85%, which matches or beats most general-purpose OCR services and skips the per-vendor template tuning those services need. The cost (0.01 to 0.05 USD per invoice) is comparable to dedicated OCR APIs, and the workflow is one node simpler.

### Why the trigger is folder-watch, not Gmail

Email triggers are tempting but messy: marketing emails, replies, and "your invoice is attached as a link" notifications all hit the same inbox. A Drive folder is an explicit hand-off ("this is an invoice, please process it") and the only way a non-invoice ends up in the pipeline is if a human dropped it there.

If you need email ingestion, run a small upstream workflow that watches Gmail, downloads PDF attachments matching invoice keywords, and saves them to the Drive folder. Keep that classification step out of this template.

### Why the duplicate branch logs to a Sheet, not Slack

A logged row is durable and searchable. A Slack message is ephemeral and gets buried by the next 50 channel messages. The Sheet log is also a perfect audit trail when an accountant asks "did we ever see this invoice number before".

If you want both, add a Slack node downstream of `Log Duplicate Warning`. The default omits it to keep the credential set tight.

### Why a Code node parses JSON instead of using the model's tool-use feature

The HTTP Request node is universal. A Code-node parser handles markdown fences, occasional trailing commas, and other model output quirks without changing nodes. Tool-use mode would buy stricter schema enforcement at the cost of wiring a different node and locking the workflow to a specific Anthropic SDK shape. For a 6-field extraction, the Code parser is enough.

## Performance notes

| Step | Latency expectation |
|---|---|
| Drive trigger detection | 1 to 5 minutes (Drive polling interval) |
| Download PDF | 1 to 3 seconds for typical invoices (under 1 MB) |
| Convert to Base64 | under 100 ms |
| Claude Vision call | 5 to 15 seconds, depends on PDF size and model |
| Parse JSON | under 100 ms |
| Sheets read for dedup | 1 to 2 seconds |
| Sheets append | 1 to 2 seconds |

End to end: about 60 seconds from "PDF lands in Drive" to "row in Sheet", dominated by Drive polling and the Claude call.

## Observability

- **n8n Executions panel** is the primary debugging surface. Each invoice run shows up there.
- **The `Duplicates` tab** doubles as your incident log: parse errors with `UNKNOWN` numbers and real duplicates both end up there.
- **Anthropic console usage page** tracks token spend per workflow run. Watch it for the first month, especially if you start uploading scanned PDFs (token count for image pages is much higher than text pages).
- Consider adding a final `Append` step that writes one row per execution to a `RunLog` tab, capturing timestamp, file_id, status, and total tokens used. Useful when failures cluster.

## See also

- [SECURITY.md](SECURITY.md) - prompt injection through invoice text, OAuth scoping, OCR accuracy as a financial integrity issue
- [SETUP.md](SETUP.md) - Drive folder setup, sheet schema, OCR caveats, dedup strategy
- [Catalog architecture principles](https://github.com/MinaSaad1/n8n-ai-agents/blob/main/docs/architecture-principles.md) - patterns shared across every template in the collection
