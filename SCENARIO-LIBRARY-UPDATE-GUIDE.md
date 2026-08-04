# Scenario Library — Update Guide & Refresh Protocol

> This library is a living document. This file tells you when to update it, what to update, and how.

---

## The Three-Layer Model (what we're maintaining)

```
Layer 1 — Scenarios    SCN-*.md files     What the patron/agent experiences
Layer 2 — Capabilities layer2/capabilities.md    Platform limits and technical constraints
Layer 3 — Customer asks    Gaps embedded in HTML    What's broken and why
```

All three layers are linked. A new customer ask may update all three simultaneously.

---

## When to Refresh

| Trigger | What to update |
|---|---|
| A Jira ticket changes status (e.g. ORC-53055 ships in 26.3.1) | Update `status:` and `last_verified:` in the relevant SCN file. Update gap card in HTML. |
| A new customer ask arrives | Add a new SCN file. Add a gap card. Add capability chips if new technical limits surface. |
| A grooming session resolves an open question | Find the `[OPEN]` or `[UNKNOWN]` tag in the relevant SCN file and update to `[CONFIRMED]`. |
| A new Jira is created for a known gap | Update the `jira:` field in the gap card. |
| A platform limit changes (e.g. max conference parties raised) | Update `layer2/capabilities.md` and the relevant CAP chip in the HTML. |
| A release ships | Search the SCN files for the release version. Change `status: unsupported` → `status: supported`. |

---

## How to Refresh (step-by-step)

### Step 1 — Pull current Jira status for tracked tickets

Run these queries in this conversation:

```
Atlassian:searchJiraIssuesUsingJql
  jql: "key in (ORC-53055, ORC-53014, ORC-34959, ORC-51895, ORC-53277, ORC-51786, ORC-53113, ORC-48255, ORC-51894, ORC-52330, ORC-22932, ORC-43660, AW-57850)"
  fields: ["summary", "status", "fixVersions", "assignee"]
```

Compare the returned statuses against the `status:` fields in each SCN file and gap card.

### Step 2 — Check for new customer escalations

Pull the Sid - VC sheet from the latest version of `Orchestration_RedAccount_Priorities.xlsx` (or the red account weekly summary email). Look for:
- New rows added since `last_verified` date
- Status changes on existing rows
- New Jira keys

### Step 3 — Update SCN files

For each changed ticket:
1. Open the relevant `SCN-*.md` file
2. Update `status:`, `last_verified:`
3. Change any `[GUESSING]` tags to `[CONFIRMED]` or `[UNKNOWN]` based on new information
4. If a gap is resolved: add `resolved: true` and the fix version

### Step 4 — Update the HTML

The HTML embeds all scenario, gap, and capability data as JavaScript constants. The update process is:

1. Find the relevant constant in the `<script>` block:
   - `CAPS` = capability constraints
   - `GAPS` = gap cards (Layer 3)
   - `PHASES` = journey phases with mm content (Layer 1)
   - `PHASE_CAPS` = which caps apply to which phase
   - `PM_LENSES` = PM stress-test analyses

2. Update the fields that changed. Common changes:
   - Gap status: `status:'In Progress'` → `status:'Done ✅'`
   - Gap delivery: `delivery:'27.1'` → `delivery:'26.3.1 ✅'`
   - Severity: `sev:'gap'` → `sev:'enh'` (if resolved to workaround)
   - Cap status: `status:'block'` → `status:'ok'` (if fixed)

3. For a shipped feature: change the phase colour:
   - `col:'gap'` → `col:'sup'` in the PHASES array
   - Remove the gap card from GAPS (or mark it resolved)

### Step 5 — Add new PM lenses as needed

When a new customer ask arrives that doesn't fit an existing PM lens, add one to `PM_LENSES`:

```javascript
  pN: {
    scenario: '<customer ask in one sentence>',
    lenses: [
      {name:'State machine feasibility', icon:'⬡', cls:'li-b', scls:'gc-r', status:'BLOCKED',
       body:'<plain English explanation>',
       findings:[{t:'b', v:'<gap>'}, {t:'w', v:'<warning>'}, {t:'o', v:'<ok/mitigant>'}]},
      // ... 4 more lenses: agent behaviour risk, platform surface, multi-leg, reporting
    ]
  }
```

---

## Tracked Jira Keys (as of 2026-07-24)

| Key | Title | Phase | Current Status |
|---|---|---|---|
| ORC-53055 | Hold music missing after conference disbands | p5 | In Progress · 26.3.1 |
| ORC-53014 | Surface consult system script | p4, p5 | New — Sizing |
| ORC-34959 | IVR press path → Data Streams | p2 | New — Sizing · 26.4 |
| ORC-51895 | On-demand ACW extension | p9 | Sized XL-XXL · 27.3 |
| ORC-53277 | Conference attendees get ACW | p9 | Sized SM-MED · 27.1 |
| ORC-51786 | Conference participant visibility P1 | p5 | Done ✅ 26.1.7 |
| ORC-53113 | Conference participant visibility P2 | p5 | Sizing · 27.3 |
| ORC-48255 | Change default hangup behavior | p8 | Sized XL-XXL · 27.1 |
| ORC-51894 | Indicate on consult leg | p4, p5 | Sized XL-XXL · 27.1 |
| ORC-52330 | Cross-BU transfer tracking | p7 | Sized L · 27.1 |
| ORC-22932 | MaxSpawnScriptLimit enforcement | p1, p2 | New — Unresolved |
| ORC-43660 | Carnival multi-call MCH Phase 1 | p5 | In Progress · 26.3 |
| AW-57850 | Agent transfers, consults, conferences | p5–p9 | Open — Planning |
| ORC-53330 | IVR data spike | p2 | New |

---

## Where These Files Should Live

**Recommended home:** `vc-capability-qa/scenario-library/`

```
vc-capability-qa/
  scenario-library/
    INDEX.md                          ← scenario catalogue
    SCENARIO-LIBRARY-UPDATE-GUIDE.md  ← this file
    00-SCENARIO-TEMPLATE.md           ← schema for new scenarios
    SCN-001-inbound-call.md
    SCN-002-patron-drops-unexpectedly.md
    ... (all SCN files)
    SCN-013-ivr-press-path.md
    SCN-014-consult-system-script.md
    layer2/
      capabilities.md                 ← CAP-01 through CAP-10+
    output/
      cxone-scenario-explorer.html    ← rendered explorer (regenerated on refresh)
```

**Why here:** The `vc-capability-qa` repo is already shared with engineering (Brendan Johnston et al). Putting the scenario library here means engineers see the same PM-layer context that was used to build QA test sets. When a new escalation comes in, the scenario file is the bridge between "what the customer experienced" and "what the QA test should cover."

---

## What This Library Is NOT

- Not a substitute for Jira. Jira owns delivery tracking. This owns understanding.
- Not a specification. This is a reference map. Specs live in Jira descriptions.
- Not static. Every release cycle, run the refresh protocol above.

---

## PM Questions This Library Should Be Able to Answer in < 3 Clicks

1. "A new customer says they can't trigger a survey after a conference call — what's the issue?"
   → Phase p5 or p8 → Gap G13 (ORC-53014) + G3 (ORC-48255)

2. "Carnival wants to conference during a multi-call — is that supported?"
   → Phase p5 → PM Lens → Conference + Multi-call composition → BLOCKED (edge case ORC-43660)

3. "Blackhawk says their scripts are spawning out of control — what do we know?"
   → Phase p2 → CAP-01/C08 chip → ORC-22932 (limit not enforced via API/UI spawn paths)

4. "GM says the patron can hear silence after the conference ends — is there a fix?"
   → Phase p5 → Gap G1 (ORC-53055) → In Progress · 26.3.1 · Vahid Shaikh

5. "Services Australia says agents in a conference don't get ACW — what's the fix timeline?"
   → Phase p9 → Gap G6 (ORC-53277) → Sized SM-MED · 27.1

