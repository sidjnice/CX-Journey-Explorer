---
id: SCN-013
title: IVR Press Path Data → Data Streams
category: data-observability
status: partial
source: ORC-34959 · ORC-53330 (spike) · Jira 2026-07-24
last_verified: 2026-07-24
---

## Summary
VC generates IVR log data (DTMF key-press / press path) for every call and currently writes it to a Kinesis stream (IVR LOG). This data is NOT accessible to customers through Data Share or the data lake in a usable form. Customers cannot build IVR press path reports. ORC-34959 adds the missing fields (ACTION_LABEL, InteractionID) to the existing Kinesis stream so downstream analytics and the data warehouse decommission can proceed.

## Trigger
Agent hangs up or patron navigates IVR — MENU action captures DTMF; VC writes an IVR LOG event to Kinesis. Currently missing fields prevent data lake joins.

---

## Products Involved

| Product / Component | Role |
|---|---|
| VCSvc IVR Script Engine | Generates IVR log events on each MENU action |
| Kinesis (IVR LOG stream) | Current delivery mechanism — not Kafka |
| Data Streams / Data Share | Customer-facing access point — IVR press path NOT yet available |
| Data Lake / Snowflake | Downstream consumer — cannot join without InteractionID + ACTION_LABEL |
| SuiteData | Async logging (separate from IVR LOG stream) |

---

## Journey — State Machine

- Patron navigates IVR (`MENU` action)
  - DTMF captured → script branches
    - VC writes IVR LOG event to Kinesis (exists today)
      - `→ [GAP]` Fields missing from current IVR LOG schema:
        - `ACTION_LABEL` — confirmed missing (Ganesh checked, not found)
        - `ACTION_NAME` — unclear if present under a different name
        - `InteractionID` (GUID) — needed for data lake joins
          - Current schema uses `bus_no + contactid` for joins
          - Data lake switched to regional GUIDs — keys don't align
          - `→ [UNKNOWN]` `division_no` field: from campaign? skill? agent? Not yet answered
      - Phase 1 (ORC-34959): Add missing fields to existing Kinesis stream (DBandStream mode)
        - Schema change is CU-worthy (adding fields only, no serialization impact)
        - Phased BU rollout: enable per deploy, not tenant-by-tenant
        - `→ [UNKNOWN]` Volume spike check needed — ORC-53330 spike (1-day work)
      - Phase 2: Enable for all BUs via post-deploy script
      - Phase 3: Clean up legacy DB mode, default to stream

---

## Events

### Emitted by VC → Kinesis IVR LOG stream

| Event Field | Current Status | After ORC-34959 |
|---|---|---|
| `ACTION_GUID` | Present | Present |
| `ACTION_LABEL` | Missing | Added |
| `ACTION_NAME` | Unclear (may exist under different name) | Confirmed |
| `InteractionID` (GUID) | Missing | Added |
| `bus_no + contactid` | Present | Present (retained for legacy) |
| `division_no` | Present (source unclear) | Clarify which entity |

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| Studio Script Action | `MENU` | Captures DTMF — generates IVR LOG event |
| Kinesis | IVR LOG stream | Current delivery — preferred over Kafka (scope decision 2026-05-27) |
| Data Streams | Customer-facing | Not yet accessible for press path |
| Data Lake | Snowflake DAT-19786 | Data governance tracked separately |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| Customer IVR script | `Code/IvrScripts/` | Contains MENU action — press path defined here |

---

## Open Questions / Gaps

- `[UNKNOWN]` `division_no`: which entity is this tied to — campaign, skill, or agent? Asked in grooming, not yet answered
- `[UNKNOWN]` Are there other downstream consumers of the IVR LOG Kinesis stream beyond Schwab + Anupam's team? Check KCL
- `[UNKNOWN]` Volume spike: how much additional Kinesis capacity is needed? ORC-53330 is the spike ticket (1-day estimate)
- `[GUESSING]` `ACTION_NAME` may exist in current schema under a different field name — needs Ganesh to confirm
- `[CONFIRMED]` `ACTION_LABEL` is confirmed missing. The VC `Label` field appears to be the same thing — needs final confirmation
- `[CONFIRMED]` InteractionID addition is CU-worthy (schema addition only, no serialization change)
- `[OPEN]` Digital channel parity: IVR LOG stream is voice-only — does digital need equivalent press-path data? Not in scope for ORC-34959

---

## Related Scenarios
- SCN-002: Inbound call (IVR phase generates this data)
- CAP-06: Spawn path enforcement (same Kinesis infrastructure)
- SCN-014: Consult system script (once consult script is surfaced, its press path data also needs streaming)
