# Setup notes

Specifics that don't fit cleanly in the README's Quickstart. Read this before activating in production.

## Google Drive folder setup

The workflow watches a single folder for new PDF uploads. Create one dedicated to invoices, do not reuse a general "Documents" folder.

1. In Drive, create a folder named something like `Invoice Inbox`. The exact name doesn't matter, but be specific so future-you doesn't dump random files into it.
2. Open the folder. Copy the part of the URL after `/folders/`, that's the folder ID. Example: `https://drive.google.com/drive/folders/1abc_DEF234ghi567JKL` gives you `1abc_DEF234ghi567JKL`.
3. Paste that ID into the `Drive New Invoice` node, **Folder to Watch** field.
4. (Optional but recommended) Set up a Drive sharing rule so only people authorized to handle AP can drop files there. Or share a folder-specific upload link with vendors and route their submissions in directly.

### Scoping OAuth to the folder, not all of Drive

The default Google Drive OAuth flow asks for full Drive access. That's more than the workflow needs. Two options:

**Option A: Use the read-only scope at the OAuth client level.** When configuring the OAuth client in Google Cloud Console, request `https://www.googleapis.com/auth/drive.readonly` instead of the default `https://www.googleapis.com/auth/drive`. The trigger and download node both work with read-only.

**Option B: Use `drive.file` scope.** This restricts access to files the user explicitly opens via a Google Picker or files the app itself created. Useful if you want the absolute minimum, less convenient because the user has to grant per-file access through a picker UI.

For most teams, Option A is the right balance. Set it once at the OAuth client level, then any credential created from that client inherits the restricted scope.

If you've already authorized n8n with the broad scope, you'll need to revoke and re-authorize. Visit https://myaccount.google.com/permissions, remove n8n's grant, then reconnect from inside n8n with the OAuth client configured to the smaller scope.

## Google Sheet schema

The workflow expects a spreadsheet with two tabs. Create them before activating, the workflow doesn't auto-create.

### Tab 1: `Invoices`

Column headers (row 1, exact spelling, case matters):

| Column | Type | Notes |
|---|---|---|
| Processed At | ISO 8601 timestamp | When the workflow ran |
| Vendor | string | From `vendor_name` |
| Invoice Number | string | From `invoice_number`, primary dedup key |
| Invoice Date | YYYY-MM-DD | From `invoice_date` |
| Due Date | YYYY-MM-DD or empty | From `due_date`, may be null |
| Subtotal | number | Pre-tax amount |
| Tax | number | Tax line, 0 if absent |
| Total | number | Final amount including tax |
| Currency | 3-letter code | ISO 4217: USD, EUR, EGP, etc. |
| Line Items JSON | JSON string | Stringified array of line item objects, useful for audit, optional for accounting |
| Source PDF ID | Drive file ID | Link back to the source PDF |
| File Name | string | Original PDF filename |

### Tab 2: `Duplicates`

| Column | Type | Notes |
|---|---|---|
| Detected At | ISO 8601 timestamp | When the workflow saw the duplicate |
| Invoice Number | string | The number that already exists |
| Vendor | string | Vendor on the new (rejected) PDF |
| File Name | string | Original PDF filename of the rejected file |

### (Optional) Tab 3: `NeedsReview`

If you wire in a math-check or scanned-PDF guard from `SECURITY.md`, route flagged rows here instead of into `Invoices`. Same columns as `Invoices` plus a `Reason` column.

## OCR caveats and accuracy expectations

Claude Vision reads PDFs in two modes depending on the file:

| PDF type | Accuracy expectation |
|---|---|
| Machine-generated invoice PDFs (most modern invoicing software outputs) | Near perfect, > 99% on all fields |
| High-resolution scanned PDFs (300+ dpi, clear text) | About 90-95%, occasional digit errors |
| Low-resolution scanned PDFs (faxed, photographed) | About 80-85%, digit errors and field misclassification more common |
| Handwritten invoices | Unreliable, do not depend on the extracted numbers without manual review |

The workflow does not auto-detect PDF type. If your workload mixes types, consider routing PDFs without an embedded text layer through a `NeedsReview` tab by default. n8n doesn't expose a "has text layer" flag natively, the cheapest detection is to count the bytes returned by Drive: scanned PDFs are typically 5x to 20x larger than text-only PDFs of the same page count.

**Always review before paying.** This is non-negotiable, not a soft suggestion. The Sheet is a queue of extracted data, not an authorization to send money.

## Claude Vision pricing

Cost per invoice runs roughly:

| Invoice type | Approx tokens | Approx cost |
|---|---|---|
| 1-page text invoice (pre-paid PDF) | 1,500 to 2,500 input tokens, 400 output | 0.01 to 0.02 USD |
| 1-page scanned invoice | 4,000 to 8,000 input tokens, 400 output | 0.02 to 0.04 USD |
| Multi-page invoice (3 to 5 pages) | 8,000 to 20,000 input tokens, 600 output | 0.04 to 0.10 USD |

> **Note**: cost and latency figures below were measured on Claude Haiku 4.5. The workflow now ships with Claude Sonnet 4.6 as the default, which is more capable and more expensive. Select a Haiku model in the node's model dropdown to get back to the numbers quoted here.


Pricing is for `claude-sonnet-4-6` at mid-2026 rates. `claude-sonnet-4-6` cuts the cost roughly in half with a slight accuracy trade-off on scanned PDFs.

At 200 invoices a month on machine-generated PDFs, expect about 4 USD. At 500 invoices a month with a mix of scanned files, expect 15 to 25 USD. Set the spend cap in the Anthropic console at roughly 2x your expected monthly bill so a runaway loop has a ceiling.

## Duplicate detection and the composite key trap

The default dedup key is `invoice_number`. That works for vendors who use globally-unique invoice numbers across their lifetime. It breaks for:

- Small vendors who restart numbering each year (`INV-001` every January 1st)
- Vendors who use sequential numbers per customer (your `INV-001` is also their next customer's `INV-001`)
- Vendors whose numbering scheme collides across business units they own

Two fixes:

### Fix 1: Composite dedup key (recommended)

In `Parse Invoice JSON`, build a composite key:

```javascript
parsed.dedup_key = [
  (parsed.vendor_name || '').toLowerCase().replace(/[^a-z0-9]/g, ''),
  (parsed.invoice_number || '').toLowerCase().replace(/[^a-z0-9]/g, ''),
  parsed.invoice_date ? parsed.invoice_date.slice(0, 4) : ''
].join('|');
```

Add a `Dedup Key` column to the `Invoices` tab. Update `Check for Duplicate` to filter on `Dedup Key` instead of `Invoice Number`. Now `INV-001` from Vendor A in 2026 and `INV-001` from Vendor B in 2026 don't collide, and `INV-001` from Vendor A in 2025 vs 2026 don't collide either.

### Fix 2: Vendor-scoped dedup

Keep `invoice_number` as the primary key but add a vendor-name filter. Edit `Check for Duplicate` to filter on both `Invoice Number == X` AND `Vendor == Y`. Slightly weaker than the composite key (still vulnerable to vendors who restart numbering yearly), but a smaller code change.

For most users, the composite key is the safer default. Recommend rolling it out before you have enough rows in the sheet to need a real backfill.

## Always-review-before-paying disclaimer

This is not a "good practice" reminder, it's a hard operating rule for any workflow that automates the data layer of accounts payable.

The workflow extracts numbers from PDFs and writes them to a sheet. It does not:

- Verify that the vendor is one you actually do business with
- Check that the line items match a purchase order
- Confirm that the bill-to address is yours
- Detect a fraudulent invoice that's structurally identical to a legitimate one

A human approver does these checks. Wire your AP process so payment authorization happens after a human reviews the row, never before. The Sheet is a fast input queue, not an automated payment system.

If you eventually want to automate small-invoice payments (under 50 USD, recurring vendors only), do that as a separate workflow that reads from a `Approved` tab someone explicitly moves rows into. Don't shortcut this template into doing both jobs.
