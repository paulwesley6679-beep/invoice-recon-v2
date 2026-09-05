# Agent 04 — Report

## Objective

Read each fully-processed structured invoice and populate **only** the `report` section with a summary and a recommended action.

## Inputs

- `data/structured/<invoice_id>.json` — the structured invoice files, with `structured_invoice`, `matching`, and `exceptions` already populated.
- `docs/schema.md` — the shared schema.

## Output

Updates each `data/structured/<invoice_id>.json` file in place, filling the `report` section.

## Responsibilities

For each invoice, derive a human-readable report from the exceptions and matching data:

1. **Write a `summary`** — a one- to two-sentence description of the invoice outcome, referencing the invoice id, PO, and receipt where applicable.
2. **Set a `status`**:
   - `"reconciled"` — no exceptions and matching status is `"matched"`.
   - `"needs_review"` — at least one exception, none critical.
   - `"blocked"` — any critical exception (e.g. `missing_po`, `missing_amount`).
3. **Pick a `recommended_action`** based on the worst-severity exception:

   | Condition | Recommended action |
   |-----------|--------------------|
   | No exceptions / reconciled | "Approve for payment" |
   | `amount_mismatch` present | "Request vendor credit or corrected invoice" |
   | `duplicate_invoice` present | "Void duplicate invoice, keep the original" |
   | `missing_receipt` present | "Confirm goods received before payment" |
   | `missing_po` or `missing_amount` present | "Hold payment until data is resolved" |

   The report agent chooses the single most appropriate action; if several apply, prefer the most severe one.
4. **Add notes** — brief free-text when helpful (e.g. a duplicate pair reference).

## Boundaries — What This Agent Must NOT Do

- Do **not** touch, edit, or overwrite `raw_inputs`, `structured_invoice`, `matching`, or `exceptions`.
- Do **not** re-derive matching or exception data. Read, summarize, act, but never change.

## Schema Compliance

- Fill only the `report` section per `docs/schema.md`.
- Non-empty `report` for every invoice (by this stage all sections must be populated).

## Example Output Snippet (for `INV-2024-005`, missing PO)

```json
{
  "report": {
    "summary": "Invoice INV-2024-005 is missing a PO number and cannot be matched.",
    "status": "blocked",
    "recommended_action": "Hold payment until data is resolved",
    "notes": ["No PO reference found in invoices.csv"]
  }
}
```

## Definition of Done

- Every structured invoice file has `report` populated with `summary`, `status`, `recommended_action`, and `notes`.
- Status and recommended action are consistent with the `matching` and `exceptions` sections.
- `raw_inputs`, `structured_invoice`, `matching`, and `exceptions` are unchanged.