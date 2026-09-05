# Shared JSON Schema

Every structured invoice produced by the pipeline conforms to a single shared JSON schema, stored at `data/structured/<invoice_id>.json`. The schema has five top-level sections:

| Section              | Owned By           | Purpose                                                        |
|----------------------|--------------------|----------------------------------------------------------------|
| `raw_inputs`         | Ingestion          | Original values as read from the CSVs                          |
| `structured_invoice` | Ingestion          | Normalized invoice fields                                      |
| `matching`           | Matching agent     | Result of 3-way matching (invoice vs PO vs receipt)            |
| `exceptions`         | Exceptions agent   | Flags for duplicates, mismatches, and missing data             |
| `report`             | Report agent       | Human-readable summary and recommended action                  |

Each agent populates **only** its own section and leaves the rest `null` or untouched. The final output has every section populated.

## Top-Level Shape

```json
{
  "schema_version": "1.0",
  "invoice_id": "INV-2024-001",
  "raw_inputs": { },
  "structured_invoice": { },
  "matching": null,
  "exceptions": null,
  "report": null
}
```

- `schema_version`: semver string identifying the version of this schema.
- `invoice_id`: stable identifier for the invoice (used as the JSON filename).
- Section values that have not yet been populated by their owning agent must be `null`, **not** empty objects or missing keys.

## Section: `raw_inputs`

Original, unmodified values copied directly from the source CSV row. This preserves the source of truth for auditing even after normalization.

```json
{
  "raw_inputs": {
    "source": "data/invoices.csv",
    "row": {
      "invoice_id": "INV-2024-001",
      "po_number": "PO-1001",
      "vendor": "Acme Supplies",
      "invoice_date": "2024-01-10",
      "due_date": "2024-02-09",
      "amount": "12500.00",
      "currency": "USD"
    }
  }
}
```

- Every key present in the source CSV row is preserved verbatim (values remain strings).

## Section: `structured_invoice`

Normalized, typed representation of the invoice, produced by the ingestion agent.

```json
{
  "structured_invoice": {
    "invoice_id": "INV-2024-001",
    "po_number": "PO-1001",
    "vendor": "Acme Supplies",
    "invoice_date": "2024-01-10",
    "due_date": "2024-02-09",
    "amount": 12500.00,
    "currency": "USD",
    "normalization": {
      "amount_currency": "USD",
      "notes": []
    }
  }
}
```

- Amounts are normalized to a numeric value, with currency tracked separately.
- `po_number` is `null` when the source row has no PO number.
- `normalization.notes` holds free-text notes from ingest (e.g. "PO number missing", "amount flagged as needs_review").
- Non-applicable values are `null`; there are **no** empty-string placeholders.

## Section: `matching`

Populated only by the matching agent. Holds the outcome of 3-way matching between the invoice, its PO, and its receipt.

```json
{
  "matching": {
    "po_match": {
      "po_number": "PO-1001",
      "found": true,
      "amount_match": true,
      "amount_invoice": 12500.00,
      "amount_po": 12500.00,
      "delta": 0.00
    },
    "receipt_match": {
      "receipt_id": "RCPT-2024-001",
      "found": true,
      "quantity_match": true,
      "quantity_invoice": null,
      "quantity_receipt": 10
    },
    "overall_status": "matched",
    "notes": []
  }
}
```

- `po_match.found`: whether a PO referenced by the invoice exists.
- `po_match.amount_match`: whether invoice amount equals PO amount within tolerance.
- `receipt_match.found`: whether a receipt for the PO exists.
- `receipt_match.quantity_match`: whether quantities line up (skipped when invoice has no quantity).
- `overall_status`: `"matched"`, `"partial"`, or `"unmatched"`.

## Section: `exceptions`

Populated only by the exceptions agent. An array of exceptions detected for the invoice.

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

- `has_exceptions` is `false` when no exceptions exist; `items` is `[]` in that case.
- `type` is one of: `duplicate_invoice`, `amount_mismatch`, `missing_po`, `missing_amount`, `missing_receipt`, `quantity_mismatch`.
- `severity` is `"info"`, `"warning"`, or `"critical"`.

## Section: `report`

Populated only by the report agent. A summary and recommended action for the invoice.

```json
{
  "report": {
    "summary": "Invoice INV-2024-001 matches its PO PO-1001 and receipt RCPT-2024-001.",
    "status": "reconciled",
    "recommended_action": "Approve for payment",
    "notes": []
  }
}
```

- `status` is `"reconciled"`, `"needs_review"`, or `"blocked"`.
- `recommended_action` is a short human-readable instruction.

## Example: Fully Populated Document

```json
{
  "schema_version": "1.0",
  "invoice_id": "INV-2024-001",
  "raw_inputs": {
    "source": "data/invoices.csv",
    "row": {
      "invoice_id": "INV-2024-001",
      "po_number": "PO-1001",
      "vendor": "Acme Supplies",
      "invoice_date": "2024-01-10",
      "due_date": "2024-02-09",
      "amount": "12500.00",
      "currency": "USD"
    }
  },
  "structured_invoice": {
    "invoice_id": "INV-2024-001",
    "po_number": "PO-1001",
    "vendor": "Acme Supplies",
    "invoice_date": "2024-01-10",
    "due_date": "2024-02-09",
    "amount": 12500.00,
    "currency": "USD",
    "normalization": { "amount_currency": "USD", "notes": [] }
  },
  "matching": {
    "po_match": { "po_number": "PO-1001", "found": true, "amount_match": true, "amount_invoice": 12500.00, "amount_po": 12500.00, "delta": 0.00 },
    "receipt_match": { "receipt_id": "RCPT-2024-001", "found": true, "quantity_match": true, "quantity_invoice": null, "quantity_receipt": 10 },
    "overall_status": "matched",
    "notes": []
  },
  "exceptions": {
    "has_exceptions": false,
    "items": []
  },
  "report": {
    "summary": "Invoice INV-2024-001 matches its PO PO-1001 and receipt RCPT-2024-001.",
    "status": "reconciled",
    "recommended_action": "Approve for payment",
    "notes": []
  }
}
```

## Validation Rules

- All five top-level keys must be present at all times.
- Sections not yet owned/populated must be `null`.
- An agent must never empty, overwrite, or delete another agent's section.
- Unknown keys are permitted for forward-compatibility but should be documented.
