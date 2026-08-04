# CXone VC Scenario Explorer

A PM intelligence tool that maps enterprise contact center requirements to CXone Voice Channel capability — and surfaces what's missing before it becomes an escalation.

> *(add screenshot here)*

---

## The problem this solves

Customer escalations arrive without structure. A PS engineer files a ticket saying "conference hold music is broken for Carnival." Without context it takes hours to answer: Is this a known gap? Does it already have a Jira? What release is it targeting? Which other customers are affected? What are the open questions before we can commit?

The harder problem is the gaps customers haven't asked about yet — the ones that will become escalations once a feature ships. A new consult script capability looks clean in a demo. But does it work on a conference leg? Does it survive a warm transfer? Does it create a new ACW gap downstream? Those questions should be answered before the feature ships, not after.

This tool answers both.

---

## Goal

**Reduce the time from "customer escalation arrives" to "PM has a grounded answer" from hours to minutes.**

Specifically:
- Is this a known gap, and what Jira tracks it?
- What release is it targeting, and which customers are affected?
- Does this customer's ask expose a gap they haven't hit yet?
- What are the open questions before I can commit to delivery?

---

## What's in this repo

```
vc-scenarios/
  README.md                                         ← this file
  vc-scenario-archetypes.md                         ← 11 canonical interaction patterns (foundation doc)
  scenario-library/
    cxone-scenario-explorer.html                    ← the tool (open in browser)
    SCENARIO-LIBRARY-UPDATE-GUIDE.md                ← how to refresh gaps and capabilities
    layer2/
      capabilities.md                               ← CAP-01–CAP-11: all platform limits
    SCN-013-ivr-press-path-data-streams.md
    SCN-014-consult-system-script-customization.md
```

The HTML file is the primary interface. The markdown files are the source it's built from.

---

## How to open the explorer without downloading the repo

GitHub Enterprise does not render HTML files inline. Two options:

**Option 1 — htmlpreview (no install, browser only)**

Navigate to the file in GitHub, copy the URL, then paste it into:

```
https://htmlpreview.github.io/?https://github.com/inContact/vc-scenarios/blob/main/scenario-library/cxone-scenario-explorer.html
```

Bookmark that URL — it always renders the latest version from `main`.

> Note: htmlpreview requires the repo to be accessible in your browser session. If it doesn't work on your network, use Option 2.

**Option 2 — download the single file**

1. Navigate to `scenario-library/cxone-scenario-explorer.html` in GitHub
2. Click **Raw** (top right)
3. Right-click → **Save As** → save as `.html`
4. Open in any browser

One file, no dependencies, no repo clone needed.

---

## How the explorer works

The explorer has three modes selectable from the top bar:

**Journey** — the patron/agent interaction as a phase strip. Each phase shows the platform events, states, capability constraints, and open gaps for that part of the journey. Click any phase to see the detail.

**Gap Map** — all tracked platform gaps with Jira status, severity, delivery target, and affected customers. The definitive list of what's broken and when it ships.

**PM Lens** — stress-test analyses for specific customer scenarios. Each lens asks: what does the state machine actually support here? What are the hidden gaps? What can't be committed?

**Archetype filter** — select any of the 11 canonical interaction archetypes (S1–S11) to filter the Gap Map and highlight the Journey phases relevant to that interaction type. S5 Mid-Call shows the highest gap density. S4 Queue Health and S7 Callback have zero tracked gaps — which means they haven't been tested yet.

---

## The three layers

The tool is built on three layers of knowledge:

**Layer 1 — Interaction archetypes.** 11 canonical interaction patterns derived from the building blocks of the CXone VC platform: contact origination, channel, skill assignment model, queue state, routing strategy, script layer, execution phase, mid-call action, and infrastructure state. These are defined in `vc-scenario-archetypes.md`. They are the foundation for Rovo's coverage assessment.

**Layer 2 — Capability constraints.** Platform limits that exist independent of any scenario — script concurrency ceilings, conference party limits, ACW timer mechanics, Config Manager reliability, consult leg script context. These are documented in `layer2/capabilities.md` (CAP-01 through CAP-11). Some are PM-configurable. Some are architectural. CAP-11 is a production reliability bug with an unmerged fix.

**Layer 3 — Gaps.** Where Layer 1 breaks because Layer 2 has a limit — discovered when customers hit them. 13 gaps currently tracked, mapped to Jira, phases, customers, and delivery targets.

---

## Knowledge sources

| Source | What it provides | Confidence |
|---|---|---|
| Jira (13 tracked tickets) | Gap status, root cause, fix mechanism, delivery target, customer labels | [Certain] — fetched directly each session |
| Confluence 17504254 (Agent Events List) | All event type names, Status/CallType/CurrentState enum values, Long/Short Lived classification | [Certain] — read directly |
| Confluence 1556120293 (RRR MVP) | Routing strategy constraints, Bullseye/RRR mutual exclusion | [Certain] — read directly |
| Confluence 3014623684 (Queue Health) | Primary/Secondary skill model, Queue Health architecture | [Certain] — read directly |
| Confluence 483755330 (System Scripts) | System vs custom script layering, 12 voice script categories | [Certain] — read directly |
| Studio Action Basics (help center) | Event action execution model, action categories, BEGIN | [Certain] — read directly |
| vc-claude CLAUDE.md + skill files | Internal strategy class names, state machine numbers, DB proc names | [Likely] — engineering-maintained, may lag code |

All [Certain] facts trace to a specific source read in session. Nothing in this tool is asserted from general knowledge alone.

---

## How to keep it current

Run the Jira JQL below before any planning meeting or grooming session:

```
key in (ORC-53055, ORC-53014, ORC-34959, ORC-51895, ORC-53277, ORC-51786, ORC-53113,
        ORC-48255, ORC-51894, ORC-52330, ORC-22932, ORC-43660, AW-57850, ORC-52482)
ORDER BY updated DESC
```

For each changed gap: update `status` and `delivery` in the HTML `GAPS` constant. For any newly resolved question, update the relevant SCN markdown and the `vc-scenario-archetypes.md` open questions table. Full protocol in `SCENARIO-LIBRARY-UPDATE-GUIDE.md`.

Update the archetype layer when engineering resolves any of the 8 open questions in `vc-scenario-archetypes.md` — especially the Queue Health + RRR interaction (S4), the callback routing behavior (S7), and the post-call survey extension point (S11).

---

## Relationship to vc-capability-qa

These repos are intentionally separate. They share the same source of truth but serve different audiences.

```
vc-claude (source of truth — VC platform behaviour)
    │
    ├── vc-capability-qa/    Engineering audience
    │   Bug classification (CAT 1–5), QA test sets,
    │   Flow 1/2/4, unknown-unknown extrapolation
    │   → github.com/inContact/vccapabilityqaclaude
    │
    └── vc-scenarios/        PM audience  ← this repo
        Customer ask → capability mapping,
        gap identification, archetype stress-testing
        → github.com/inContact/vc-scenarios
```

A gap classified as CAT 3 (MCH race condition) in vc-capability-qa maps to phase p5 and archetype S5 here. A gap in the scenario explorer with no CAT classification is a signal to run Flow 2 in vc-capability-qa.

---

*Owner: Siddharatha Joshi — Director of Product Management, CXone Orchestration*
*Last updated: 2026-07-30*
*Paired repo: [vc-capability-qa](https://github.com/inContact/vccapabilityqaclaude)*
