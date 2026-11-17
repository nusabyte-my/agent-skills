---
name: myinvois-einvoice-readiness
description: Audit a Malaysian business ledger for LHDN MyInvois e-invoice readiness and build UBL 2.1 draft submissions. Use when asked about e-invoicing, MyInvois, LHDN, TIN validation, MSIC classification, or SST tax codes on invoices.
license: MIT
---

# MyInvois e-invoice readiness

Malaysia mandates e-invoicing through LHDN's MyInvois portal on a phased schedule.
This skill audits an existing invoice ledger against the submission requirements and
builds UBL 2.1 draft payloads — without ever submitting, and without inventing data
to make a check pass.

## When to use

- "Are we ready for e-invoicing?" / "check our MyInvois readiness"
- An invoice needs a TIN, an MSIC code, or a tax classification it doesn't have
- Building UBL 2.1 JSON/XML for MyInvois
- A readiness report keeps failing on the same field

**Not this skill:** generating the PDF a human reads (that's the ledger renderer),
or filing actual submissions — this skill is draft-only by design.

## Hard invariants

1. **Draft-only.** Nothing is submitted to LHDN, nothing is emailed, nothing is
   marked sent, nothing is deleted. Every tool exits non-zero when a gate blocks,
   so a scheduler can branch on the exit code.
2. **Never invent compliance data.** Do not fabricate a TIN, an MSIC code, or a tax
   code to turn a check green. A red gate with the missing input named is the
   correct output. Inventing a TIN produces a filing that LHDN rejects weeks later
   and an audit trail that says you made it up.
3. **Classify only what the work supports.** MSIC codes may be inferred from the
   described service (that is a judgement call about a real deliverable). Tax
   identifiers may not be inferred at all.
4. **Snapshot files are gitignored.** A fetched ledger snapshot contains client PII.

## The readiness gate

Run these checks over every non-void document, and report per-document, not as one
aggregate pass/fail — the point is which specific doc blocks submission.

| Check | Fails when | Fixable by the agent? |
|---|---|---|
| Seller TIN | supplier `taxId` empty on the business profile | No — owner supplies it |
| Buyer TIN | client `taxId` empty on the document | No — owner requests from client |
| Per-item MSIC | any `items[].industryCode` missing | Yes — infer from the description |
| Currency | any doc not in the filing currency | No — needs an FX rate on the issue date |
| Zero or negative totals | `total <= 0` | No — usually a test doc to delete |
| Totals math | `subtotal + tax - discount != total` | Yes — recompute |
| Duplicate issue | same supplier TIN + invoice number twice | No — owner decides which is real |

Report shape per document: `number`, `client`, `total`, and the list of failed
checks with the exact field path that is empty. An agent that reports "3 of 12
docs blocked: 2 missing buyer TIN, 1 missing MSIC on item 4" has done the job; one
that reports "not ready" has not.

## Building a UBL 2.1 draft

MyInvois accepts UBL 2.1. The mapping that bites:

- `invoice number` → `ID`, and it must be unique per supplier TIN forever, not per year
- `issueDate`/`dueDate` → `IssueDate`/`DueDate` as `YYYY-MM-DD`, no timezone suffix
- supplier and buyer each need a `PartyTaxScheme` with `CompanyID` = TIN, plus
  `PartyLegalEntity/RegistrationName` and a `PostalAddress` with `CountryID = MY`
- every `InvoiceLine` needs `InvoicedQuantity` **with its `unitCode`** (UN/ECE Rec 20:
  `C62` each, `HUR` hour, `MON` month) — a bare number is rejected
- `ItemClassificationCode` carries the 5-digit MSIC with `listID="CLASS"`
- `TaxCategory` needs `ID` (`01` sales tax, `02` service tax, `E` exempt) and
  `Percent`; the `TaxScheme` must name `OTH` with the Malaysian scheme id
- `LegalMonetaryTotal` needs `LineExtensionAmount`, `TaxExclusiveAmount`,
  `TaxInclusiveAmount`, `PayableAmount` — all four, even when equal
- `InvoiceTypeCode` is `01` for invoice, `02` credit note, `03` debit note,
  `04` refund note

Write the draft as JSON next to the document, never inside the production ledger.

## Validating before submission

- The signature is a UBL `UBLExtensions` document signing block — build it last,
  after every amount is final, because signing freezes the payload.
- Validate against the published MyInvois SDK schema before hand-off; a payload
  that parses as JSON is not the same as one LHDN accepts.
- TIN format check: `C` or `D`/`F`/`E` prefix + 8-9 digits + branch code
  (e.g. `C2584563200`). Validate the shape, never the registration — there is no
  offline registry.

## Pitfalls

- **Phase thresholds move.** The rollout has been amended repeatedly (the RM1M
  band was cancelled at one point). Treat the gate as a signal, never as a
  deadline you assert to the user — and never let it block accounts-receivable
  work, which has zero TIN dependency.
- **`taxId` and `industryCode` exist in most schemas already.** The usual blocker
  is empty data, not a missing column — check the data before proposing a migration.
- **One invoice re-keyed twice reads as inflated exposure.** Dedupe on supplier TIN
  + invoice number; same client plus same total across two docs is a signal, not a
  coincidence.
- **Zero-total test docs poison every report.** Flag them for the owner to delete.
  Silently filtering them hides the fact that the ledger has junk in it.
- **Software-generated item numbering.** If the PDF renderer numbers lines, never
  put "1. " in the description — you get "5. 5.".
- **Never send the signed payload anywhere except the sandbox** without an explicit
  instruction naming the environment.
