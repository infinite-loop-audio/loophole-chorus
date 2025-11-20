# IPC Envelope `ts` Field Normalisation

**Date:** 2025-11-20  
**Task:** Align IPC documentation so that `ts` field is always required, never null, and always an ISO 8601 string

## Files Modified

- `docs/specs/ipc/envelope.md` — Updated field description for `ts` to explicitly state it is required and non-null

## Changes Summary

### Field Description Update

- Updated the `ts` field description in section 3.1 (Fields) to:
  - Explicitly mark it as **required**
  - State that it must never be null or omitted
  - Clarify that if an implementation auto-fills this field, it must still be an ISO 8601 string, never null
  - Maintain existing notes about usage (ordering diagnostics, drift detection, not for sample-accurate timing)

### Examples Verification

- All envelope examples in the document already contained `ts` as an ISO 8601 string
- No examples showed `ts: null` or omitted the field
- All examples remain valid and consistent with the updated requirement

### Other Files Checked

- Searched all IPC specification files under `docs/specs/ipc/` for references to `ts`
- No other files required updates; domain-specific specs focus on payloads rather than envelope structure
- The envelope specification is the single source of truth for envelope field requirements

## Result

The `ts` field is now consistently documented as:
- **Required** — must be present in all envelopes
- **Non-null** — must never be null
- **ISO 8601 string** — must be a valid ISO 8601 timestamp string format

This ensures clear expectations for all implementations and removes any ambiguity about whether `ts` can be optional or nullable.

