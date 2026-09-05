# Invoice Recon

A hackathon project for invoice reconciliation. Invoice Recon ingests invoices, purchase orders, and receipts, performs 3-way matching, flags exceptions, and produces a reconciliation report with recommended actions.

## Goal

Automatically reconcile vendor invoices against their corresponding purchase orders (POs) and goods receipts, and flag any exceptions that require human review. An exception can be a duplicate invoice, an amount mismatch, or missing data (such as an absent PO number or amount).

The pipeline is broken into four sequential agent steps:

1. **Ingestion** — parses the raw CSVs into structured per-invoice JSON.
2. **Matching** — performs 3-way matching of invoice vs PO vs receipt.
3. **Exceptions** — flags duplicates, mismatches, and missing data.
4. **Report** — produces a summary with a recommended action for each invoice.

Each agent populates only its own section of the shared JSON schema, so the output of one step is clean input for the next.

## Project Structure

```
├── README.md                 # This file
├── data/
│   ├── invoices.csv          # ~6 sample invoices (incl. edge cases)
│   ├── purchase_orders.csv   # Sample purchase orders
│   ├── receipts.csv          # Sample goods receipts
│   └── structured/           # Output of the ingestion agent (per invoice)
├── docs/
│   └── schema.md             # Shared JSON schema definition
└── agents/
    ├── 01-ingestion.md       # Ingestion agent task spec
    ├── 02-matching.md        # 3-way matching agent task spec
    ├── 03-exceptions.md      # Exceptions agent task spec
    └── 04-report.md          # Report agent task spec
```

## Checklist of What's Built

- [x] `README.md` — project overview, goal, and this checklist
- [x] `data/invoices.csv` — 6 sample invoices, including:
  - [x] At least one duplicate invoice (`INV-2024-004` duplicates `INV-2024-001`)
  - [x] At least one missing PO number (`INV-2024-005`)
  - [x] At least one amount mismatch (`INV-2024-006` vs `PO-1004`)
- [x] `data/purchase_orders.csv` — sample purchase orders
- [x] `data/receipts.csv` — sample goods receipts
- [x] `docs/schema.md` — shared JSON schema with `raw_inputs`, `structured_invoice`, `matching`, `exceptions`, and `report` sections
- [x] `agents/01-ingestion.md` — task spec for the ingestion agent
- [x] `agents/02-matching.md` — task spec for the matching agent
- [x] `agents/03-exceptions.md` — task spec for the exceptions agent
- [x] `agents/04-report.md` — task spec for the report agent

## How to Use

Run the agent steps in order, pointing each agent at the shared schema in `docs/schema.md`:

1. Have the **ingestion** agent read the CSVs from `data/` and write structured JSON to `data/structured/<invoice_id>.json`.
2. Have the **matching** agent fill the `matching` section of each file.
3. Have the **exceptions** agent fill the `exceptions` section.
4. Have the **report** agent fill the `report` section.

Each agent must only touch the section it owns, leaving the others untouched.
