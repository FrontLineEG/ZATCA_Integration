# ZATCA Phase 1 QR fix

App: `FrontLineEG/ZATCA_Integration`, branch `fix/phase-1-qr-without-csid` (from `version-15`)

## Issue

With **Enable ZATCA E-Invoicing** on and **ZATCA Phase = Phase 1**, submitting a Sales Invoice fails with:

> Production CSID None not found

No invoice can be submitted, and no Phase 1 QR code is created.

**Root cause:** in `clearence_util.generate_einvoice`, the CSID lookup (`get_zatca_config`) and the XML signing run *before* the check that returns for non-Phase-2 companies. Phase 1 has no CSID, so it always crashes. The ordering changed in commit [`dcc58dc`](https://github.com/FrontLineEG/ZATCA_Integration/commit/dcc58dc3f022d7375f8b2e3763d74725775ec4b0) (2025-07-08, "Reafctor the submission and handling of invoices") and is still present on `version-15`, `version-16` and `develop`.

## Decision

Minimal fix inside the ZATCA app; no new app or setting.

1. **Code (`clearence_util.py` only):** move the country and phase checks before the CSID lookup and signing. The condition is unchanged, so Phase 2 behaves exactly as before. The existing Phase 1 QR code (`phase_one_utils.create_qr_code`) then runs on submit, unchanged.
2. **Setup (Desk, per company):**
   - Company → ZATCA Settings: **Enable ZATCA E-Invoicing** ✔, **ZATCA Phase = Phase 1**.
   - Create a **Zatca CSR Settings** record with **ZATCA Phase = Phase 1** and link it in **CSR Settings (Phase 1 Only)**. The QR takes the seller's Arabic name and VAT number from this record, so they must match the company. The record's address and CRN fields are mandatory; use real values, because they are reused for the Phase 2 CSR.
   - Print Format **ZATCA**: enable it and set it as the Sales Invoice default. It shows `ksa_einv_qr` for Phase 1 and the ZATCA-cleared QR for Phase 2.

## Known limitations (not fixed, kept out of scope)

In `phase_one_utils.create_qr_code`:

- Credit notes show negative amounts in the QR.
- Round amounts print as `100.0`, not `100.00`.
- For non-SAR invoices, the VAT amount is in invoice currency while the total is in SAR.
- A `hasattr(doc, "")` typo re-creates the QR custom field on every submit.

## Tested on a local copy of prod

- Invoices were submitted with Phase 1 enabled and a CSR Settings record linked: no error, and a QR was created.
- Nothing was sent to ZATCA.
