# Agent 02 — Matching

## Objective

For each structured invoice in `data/structured/<invoice_id>.json`, perform 3-way matching (invoice vs purchase order vs receipt) and populate **only** the `matching` section.

## Inputs

- `data/structured/<invoice_id>.json` — the structured invoice files produced by the ingestion agent.
- `data/purchase_orders.csv` — PO records.
- `data/receipts.csv` — receipt records.
- `docs/schema.md` — the shared schema.

## Output

Updates each `data/structured/<invoice_id>.json` file in place, filling the `matching` section.

## Responsibilities

1. **Load** each structured invoice.
2. **Match invoice → PO** using `structured_invoice.po_number`:
   - `po_match.found`: whether a matching PO exists in `purchase_orders.csv`.
   - `po_match.amount_match`: whether the invoice amount equals the PO total within tolerance.
   - Record `amount_invoice`, `amount_po`, and `delta` (invoice minus PO).
3. **Match invoice → receipt** using the PO number:
   - `receipt_match.found`: whether a receipt exists for the PO.
   - `receipt_match.quantity_match`: whether quantities line up (set `null`/skip when the invoice carries no quantity).
   - Record `receipt_id`, `quantity_invoice`, `quantity_receipt`.
4. **Derive overall status**:
   - `"matched"` — PO found, amounts match, and receipt found.
   - `"partial"` — PO found but amount mismatch, or receipt missing.
   - `"unmatched"` — no PO found.
5. If an invoice cannot be matched because the PO number is missing/null, record `po_match.found: false` and a note, and continue.

## Boundaries — What This Agent Must NOT Do

- Do **not** touch, edit, or overwrite `raw_inputs`, `structured_invoice`, `exceptions`, or `report`.
- Do **not** flag or classify exceptions — that is the exceptions agent's job. Record objective matching facts and notes only.
- Do **not** compute or emit any report summary.

## Schema Compliance

- Fill only the `matching` section per `docs/schema.md`.
- Leave `exceptions` and `report` as `null`.
- `matching.notes` may hold short factual notes about special cases (e.g. "PO number missing, skipped Po lookup").

## Example Output Snippet (for `INV-2024-006`, amount mismatch)

```json
{
  "matching": {
    "po_match": {
      "po_number": "PO-1004",
      "found": true,
      "amount_match": false,
      "amount_invoice": 8600.00,
      "amount_po": 7900.00,
      "delta": 700.00
    },
    "receipt_match": {
      "receipt_id": "RCPT-2024-004",
      "found": true,
      "quantity_match": true,
      "quantity_invoice": null,
      "quantity_receipt": 7
    },
    "overall_status": "partial",
    "notes": []
  }
}
```

## Definition of Done

- Every structured invoice file has a valid, populated `matching` section.
- `raw_inputs`, `structured_invoice` are unchanged; `exceptions` and `report` are still `null`.
- No exception flags or report text anywhere in the output.