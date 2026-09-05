# Agent 03 — Exceptions

## Objective

Read each structured invoice and its matching output, then populate **only** the `exceptions` section with flags for duplicates, mismatches, and missing data.

## Inputs

- `data/structured/<invoice_id>.json` — the structured invoice files (with `matching` populated by the matching agent).
- `data/invoices.csv` — source invoice data, used to detect duplicates across the whole set.
- `docs/schema.md` — the shared schema.

## Output

Updates each `data/structured/<invoice_id>.json` file in place, filling the `exceptions` section.

## Responsibilities

Examine all structured invoices and build the full exception list for each one:

1. **Missing data**
   - `missing_po` — `structured_invoice.po_number` is null.
   - `missing_amount` — amount is null, zero, or non-numeric.
   - `missing_receipt` — `matching.receipt_match.found` is false.
2. **Duplicates**
   - `duplicate_invoice` — another invoice shares the same key fields (PO number, vendor, amount, invoice date). The note should reference the duplicate invoice id.
3. **Mismatches**
   - `amount_mismatch` — `matching.po_match.amount_match` is false.
   - `quantity_mismatch` — receipt found but `quantity_match` is false.

For each exception item populate `type`, `severity`, `details`, and `suggested_action`:

| Type | Severity | Suggested action |
|------|----------|------------------|
| `duplicate_invoice` | `warning` | "Void duplicate invoice, keep the original" |
| `amount_mismatch` | `warning` | "Request vendor credit or corrected invoice" |
| `missing_po` | `critical` | "Obtain PO number from vendor or purchasing team" |
| `missing_amount` | `critical` | "Request corrected invoice with amount" |
| `missing_receipt` | `warning` | "Confirm goods were received and obtain receipt" |
| `quantity_mismatch` | `warning` | "Verify received quantity with warehouse" |

`has_exceptions` is `false` (and `items` is `[]`) when no exceptions apply.

## Boundaries — What This Agent Must NOT Do

- Do **not** touch, edit, or overwrite `raw_inputs`, `structured_invoice`, `matching`, or `report`.
- Do **not** modify the matching results or the structured fields.
- Do **not** produce any summary text or recommended action — that's the report agent's job.

## Schema Compliance

- Fill only the `exceptions` section per `docs/schema.md`.
- Leave `report` as `null`.
- Always set both `has_exceptions` and `items`.

## Example Output Snippet (for `INV-2024-006`, amount mismatch)

```json
{
  "exceptions": {
    "has_exceptions": true,
    "items": [
      {
        "type": "amount_mismatch",
        "severity": "warning",
        "details": "Invoice amount 8600.00 does not match PO-1004 amount 7900.00 (delta 700.00)",
        "suggested_action": "Request vendor credit or corrected invoice"
      }
    ]
  }
}
```

## Definition of Done

- Every structured invoice file has `exceptions` populated.
- Every required exception type (duplicates, mismatches, missing data) is flagged on exactly the invoices where it applies, and nowhere else.
- `raw_inputs`, `structured_invoice`, and `matching` are unchanged; `report` is still `null`.