---
id: SCN-014
title: Surfacing Consult System Script for Customization
category: conference-consult
status: unsupported
source: ORC-53014 · ORC-51894 · Jira 2026-07-24
last_verified: 2026-07-24
---

## Summary
Today, when Agent A consults Agent B, no custom Studio script runs on Agent B's consult leg. There is no `ONASSIGNMENT`-equivalent for consult calls. This blocks: Indicate actions, post-contact surveys, real-time transcription, AutoSummary, AI Feedback Management — all require a script context to fire. ORC-53014 proposes assigning a configurable custom script to consult legs, giving script writers control over what happens when a consult call goes active. Scope question unresolved: voice only, or digital as well?

## Trigger
Agent A dials Agent B (consult) → Agent B accepts → consult leg goes Active. Currently: no script assigned. With ORC-53014: a configurable custom script starts from `BEGIN` on Agent B's leg.

---

## Products Involved

| Product / Component | Role |
|---|---|
| VCSvc IVR Script Engine | Runs the custom consult script once Agent B accepts |
| Studio | Script authoring — customers write the consult script here |
| Agent Workspace | Indicate button requires UI team work to surface on consult leg |
| Feedback Manager / WFA | Post-contact survey delivery — unblocked by this work |
| AI Transcription (Real-Time) | Transcription on consult leg — unblocked once script context exists |
| AutoSummary | Summary generation for consult/conference — unblocked |
| Agent Assist | In-call AI assistance on consult leg — unblocked |

---

## Journey — State Machine (today vs after ORC-53014)

### Today (no consult script)
- Agent A dials Agent B
  - Agent B accepts consult → consult leg Active
    - `CallContactEvent` Status:**Active** · CallType:**Consult**
    - `AgentState` Agent B → **InboundConsult (5)**
    - `→ [GAP]` No script runs — Indicate, survey, transcription, AutoSummary: all inactive
    - `→ [GAP]` Indicate button not present on Agent B's consult UI
    - `→ [GAP]` Survey qualification check cannot run on consult leg
    - `→ [GAP]` Real-time transcription not activated on consult leg

### After ORC-53014 (proposed)
- Agent A dials Agent B
  - Agent B accepts consult → consult leg Active
    - Custom consult script assigned (at BU / skill / agent level — TBD)
      - Script starts from `BEGIN` action
        - `→ [OPEN]` Does `BEGIN` fire immediately on accept, or after an `OnAnswer`-like trigger?
          - Grooming note: "confusion between OnAnswer and BEGIN" — not resolved
        - Indicate action available to Agent A (target: Agent B)
          - Agent can click Indicate at any point after BEGIN
        - Survey qualification check runs on consult leg
          - Direct survey to Agent B if Agent A releases
        - Real-time transcription activated via script
        - AutoSummary context established for consult leg
  - `→ [OPEN]` Conference scenario (3rd agent added):
    - Agent A calls Agent C into the conference
    - Agent C accepts → automatically joins conference (no separate ONASSIGNMENT)
    - `→ [UNKNOWN]` Does Agent C's consult script trigger again on join, or only on initial consult accept?
    - Grooming note: "no OnConference() action exists" — needs new action or OnAnswer modification

---

## Events

### Today — consult leg has NO script events
| Event Type | Status |
|---|---|
| `CallContactEvent` Status:Active CallType:Consult | Emitted |
| Any script-driven event (PopURL, Indicator, Indicate) | NOT emitted — no script context |
| `AgentAssist` event | NOT emitted on consult leg |

### After ORC-53014 — consult script active
| Event Type | When |
|---|---|
| Script-driven events (Indicate, PopURL, PostData) | Once custom consult script reaches those actions |
| `Indicator` event | If script triggers Indicate action targeting Agent B |
| Real-time transcription start | If script activates transcription action |
| Survey trigger | If script runs survey qualification logic |

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| Studio Script Action | `BEGIN` | Entry point for custom consult script |
| Studio Script Action | `OnAnswer` (modified) | May need validation to block on consult calls (or not trigger) |
| Studio Script Action | `Indicate` | Must work on consult leg — requires UI team work |
| Studio Script Action | `OnSignal`, `OnRelease`, `OnReskill`, `OnTransfer` | Existing handlers — do they apply on consult leg? Open question |
| VC Internal | Assignment of consult script to BU/skill/agent | New configuration model — UI team dependency |
| UI Team | Indicate button on consult leg | Currently not surfaced on Agent B's consult UI |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| Customer consult script (new) | `Code/IvrScripts/<tenant>/` | Custom script assigned to consult leg |
| System/Custom_Personal_Queue | `System/` | Reviewed as candidate — but currently tied to transfers only, not consults |
| `OnAnswer` (system) | `System/` | May need modification to validate consult context |

---

## Open Questions / Gaps

- `[OPEN]` **Script assignment level:** BU-wide single script, or configurable per skill / per agent? Most comments favour BU-level as simplest first implementation
- `[OPEN]` **Voice only or digital as well?** Asked in grooming 2026-06-29, not answered. Digital consult (chat/SMS) may need separate handling
- `[OPEN]` **BEGIN vs OnAnswer trigger:** When exactly does the consult script start? On consult accept? On conference join? Grooming note flags confusion — needs decision
- `[OPEN]` **OnConference() action:** No such action exists. When Agent C joins an existing conference, there is currently no hook. New action may be needed, or OnAnswer must be modified with conference-context validation
- `[OPEN]` **OnSignal, OnRelease, OnReskill, OnTransfer:** Do these standard event handlers apply on consult leg scripts? Not confirmed
- `[OPEN]` **Indicate button UI:** Requires Agent Workspace team work to surface the Indicate button on Agent B's consult UI. No UI-side Jira linked yet
- `[CONFIRMED]` Solving ORC-53014 unblocks: Indicate, post-contact surveys, real-time transcription, AutoSummary, Agent Assist — all for consult/conference legs
- `[CONFIRMED]` This is paired with ORC-48255 (change default hangup) — the two together enable AI Feedback Management hero project
- `[CONFIRMED]` Collaboration Layer confirmed cannot absorb this — VC must own

## Unlocks When Delivered

| Capability | Currently | After ORC-53014 |
|---|---|---|
| Indicate action on consult leg | ❌ Blocked | ✅ Supported |
| Post-contact survey on consult/conference | ❌ Workaround only | ✅ Native |
| Real-time transcription on consult leg | ❌ Not activated | ✅ Script can trigger |
| AutoSummary on consult/conference | ❌ Partial | ✅ Full context |
| Agent Assist on consult leg | ❌ Bug ORC-34797 | ✅ Unblocked (pending AI team work) |
| Custom logic (data dips, screen pops) on consult | ❌ None | ✅ Full Studio capability |

---

## Related Scenarios
- SCN-005: Warm transfer (consult leg creation — this is where the script would run)
- SCN-006: Conference call (Agent C join scenario)
- SCN-011: Post-call survey (consult script enables native survey path)
- SCN-013: IVR press path data (consult script press path also needs streaming once ORC-53014 lands)
- ORC-48255: Change default hangup (paired delivery — together enable FI hero project)
