# Agent 01 — Ingestion

## Objective

Parse the raw CSVs in `data/` and produce one structured JSON document per invoice at `data/structured/<invoice_id>.json`, conforming to the shared schema in `docs/schema.md`.

## Inputs

- `data/invoices.csv` — invoice records (invoice number, PO number, vendor, dates, amount, currency)
- `data/purchase_orders.csv` — purchase order records (read-only reference; no PO parsing required for this step)
- `data/receipts.csv` — goods receipt records (read-only reference; no receipt parsing required for this step)

## Output

One file per invoice: `data/structured/<invoice_id>.json`.

## Responsibilities

1. **Read** each CSV and pick up every row of `invoices.csv`.
2. **Copy** the original row verbatim into `raw_inputs.row` and set `raw_inputs.source` to `"data/invoices.csv"`.
3. **Normalize** the invoice fields into `structured_invoice`:
   - `invoice_id`, `vendor`, `invoice_date`, `due_date`, `currency` as-is.
   - `amount` converted from string to a number.
   - `po_number` set to the value, or `null` when the source cell is empty.
   - `normalization.amount_currency` tracks the currency the amount is normalized to.
4. **Flag** data-quality issues for human review, per the rules below.
5. **Leave untouched**: set `matching`, `exceptions`, and `report` to `null`. You must NOT populate, guess, or pre-fill any of these sections.

## Flagging Rules

Flag the following cases by adding a note to `structured_invoice.normalization.notes` (message text is your choice, but must mention the issue):

- **Missing PO number** — `po_number` is null or blank.
- **Missing amount** — amount is blank, zero, or non-numeric.
- Both flags may appear together on the same invoice.

There is no matching, exception, or report logic in this agent — anything in those sections stays `null`.

## Schema Compliance

- Follow `docs/schema.md` exactly.
- Output filenames must be `<invoice_id>.json` (e.g. `data/structured/INV-2024-001.json`).
- Do not invent fields outside the schema unless documented in `docs/schema.md`.

## Example Output Snippet (for `INV-2024-005`, missing PO)

```json
{
  "schema_version": "1.0",
  "invoice_id": "INV-2024-005",
  "raw_inputs": {
    "source": "data/invoices.csv",
    "row": {
      "invoice_id": "INV-2024-005",
      "po_number": "",
      "vendor": "Umbrella Co",
      "invoice_date": "2024-01-18",
      "due_date": "2024-02-17",
      "amount": "9750.00",
      "currency": "USD"
    }
  },
  "structured_invoice": {
    "invoice_id": "INV-2024-005",
    "po_number": null,
    "vendor": "Umbrella Co",
    "invoice_date": "2024-01-18",
    "due_date": "2024-02-17",
    "amount": 9750.00,
    "currency": "USD",
    "normalization": {
      "amount_currency": "USD",
      "notes": ["Missing PO number - flagged as needs_review"]
    }
  },
  "matching": null,
  "exceptions": null,
  "report": null
}
```

## Definition of Done

- One JSON file exists in `data/structured/` per invoice.
- Every file has `matching`, `exceptions`, and `report` set to `null`.
- All amounts are numbers, and `normalization` is populated.
- Flags for missing PO and/or missing amount appear where applicable.