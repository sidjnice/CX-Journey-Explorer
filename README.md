# CXone VC Scenario Explorer



<img width="935" height="475" alt="vc_scenarios" src="https://github.com/user-attachments/assets/359d83cc-2ba1-4c7c-9cec-08f3161850e7" />



---


A living reference for how CXone Voice Channel works — built for onboarding, planning, and answering customer asks quickly.

---

## What this is

Every contact center interaction follows the same fundamental journey: a patron calls, waits in queue, an agent answers, something happens mid-call (hold, consult, conference, transfer), the call ends, and the agent wraps up. That journey is the spine of this library.

The journey itself is stable. What changes over releases is what happens *within* each step, and how well the steps connect to each other. A new routing strategy adds capability at the queue step. A consult script feature adds capability at the mid-call step. A metadata gap means the transfer step loses context when it hands off to the next step. This library tracks all of that — what's supported today, what's being built, and what's still broken.

---

## The three layers

Every scenario in this library sits across three layers:

**Layer 1** — The patron journey. ** The phases every contact passes through, from IVR to ACW. These are the columns in the explorer. They don't change much. What changes is the platform capability within each phase and the quality of handoff between phases.

**Layer 2** — Platform capabilities. ** The constraints that exist independent of any scenario — script concurrency limits, conference party caps, ACW timer mechanics, Config Manager reliability, the missing extension point on consult legs. These are the chips in the capability bar. Some are configuration choices. Some are architectural facts. Some are production bugs with unmerged fixes.

**Layer 3** — Customer asks and gaps. ** Where Layer 1 meets Layer 2** and something breaks — discovered when customers hit a limit or a missing feature. These are the gap cards in the Gap Map. Each one has a Jira, a severity, a delivery target, and a list of affected customers.

---

## The 11 archetypes

Rather than enumerate all possible interactions, the library organizes scenarios into 11 canonical archetypes — interaction patterns that each represent a family of related scenarios. The archetype bar at the top of the explorer lets you filter the entire tool by archetype.

| Archetype | What it covers |
|---|---|
| **S1 Basic Inbound** | The atomic unit. Patron calls → IVR → queue → agent answers → ACW. Baseline for everything else. |
| **S2 Overflow / Bullseye** | Queue depth exceeds threshold → Bullseye expands proficiency range → contacts route to broader agent pool. |
| **S3 RRR Mid-Queue** | RRR rule evaluates queue statistics mid-wait → contact proficiency updated → different agents become eligible. |
| **S4 Queue Health** | Primary skill KPIs degrade → Queue Health reserves capacity by deactivating Secondary skills on protected agents. |
| **S5 Mid-Call** | Agent takes a mid-call action: hold, consult, conference, blind or warm transfer. Highest gap density of any archetype. |
| **S6 Personal Queue** | Agent already on a contact receives a second contact via Personal Queue. Max 3 concurrent per agent. |
| **S7 Callback** | Patron opts out of hold wait → system calls back when agent is available. Re-enters routing as outbound-flavored inbound. |
| **S8 Outbound** | Enterprise-initiated contact. Agent previews record, system dials, OnPreview event fires instead of OnAnswer. |
| **S9 Extension Boundary** | Custom script attempts a capability in a context the system script doesn't expose an extension point for. Consult leg is the canonical example. |
| **S10 Infra Degraded** | Any of S1–S9 during Config Manager slowness. Feature toggle checks block scripting threads → cluster unresponsive. 5 of 6 root causes open in production. |
| **S11 Post-Call** | Contact ends → ACW timer → optional post-call survey → agent released. Two distinct sub-phases with different gap profiles. |

The archetype foundations — building block axes, constraint rules (platform physics), and coverage assessment — are documented in `vc-scenario-archetypes.md`.

---

## The 9 journey phases

The phase strip across the top of the explorer represents the patron journey. Each phase is a distinct execution context with its own events, platform states, capability constraints, and gaps.

| Phase | Name | What happens |
|---|---|---|
| **p1** | Patron dials | SIP invite received, contact created, script started |
| **p2** | IVR + Queue | IVR script executes, DTMF captured, REQAGENT fires, patron waits |
| **p3** | Agent accepts | Agent offered contact, accepts, OnAnswer fires, conversation begins |
| **p4** | Active call | Live two-party call, script continues, mid-call actions available |
| **p5** | Conf / MCH | Multi-party state: hold, consult, conference, MCH manages legs |
| **p6** | Patron drops | Patron hangs up mid-conference, VC must detect and handle state change |
| **p7** | Transfer | Blind or warm transfer, new ContactID created, metadata handoff |
| **p8** | Call ends + Survey | Contact terminates, post-call script runs, survey offered |
| **p9** | ACW | After-call wrap, timer runs, disposition, agent released |

**Phase colour coding** in the strip: green = supported, amber = enhancement in progress, red = known gap. The badge number on each phase is the count of open gaps touching that phase.

---

## How to read the explorer

### The layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  HEADER    Mode switcher (Journey / Gap Map / PM Lens)  Legend      │ ← dark nav bar
├─────────────────────────────────────────────────────────────────────┤
│  ARCHETYPE BAR    S1 · S2 · S3 · S4 · S5 ... S11    Filter meta    │ ← select to filter
├─────────────────────────────────────────────────────────────────────┤
│  PHASE STRIP    p1 → p2 → p3 → p4 → p5 → p6 → p7 → p8 → p9        │ ← click any phase
├─────────────────────────────────────────────────────────────────────┤
│  CAPABILITY BAR    C01  C02  C03 ...                                 │ ← constraints for selected phase
├──────────────────────────┬──────────────────────────────────────────┤
│  SIDEBAR                 │  RIGHT PANEL                             │
│  Info / Gaps / PM Qs     │  Markmap diagram for selected phase      │
│  tabs for selected phase │  (zooms, pans, expands)                  │
└──────────────────────────┴──────────────────────────────────────────┘
```

### Reading a phase

1. Click any phase in the strip. The sidebar fills with three tabs: **Info** (products, events, states, APIs active in this phase), **Gaps** (open Jiras touching this phase), and **PM Qs** (open questions that need engineering input before committing).

2. The **capability bar** updates to show only the platform constraints relevant to that phase. Each chip is colour-coded: green = works fine, amber = partial / workaround exists, red = blocked / no fix in current release.

3. The **right panel** renders a markmap diagram of the phase — a visual breakdown of the products, events, API actions, and Layer 2 constraints in play. It zooms and pans. The diagram is the answer to "what exactly is happening inside this step?"

### Using the archetype filter

Selecting an archetype from the bar does three things simultaneously: phases outside the archetype dim in the strip, the Gap Map filters to only gaps belonging to that archetype, and the right panel shows the archetype's coverage assessment — including which gaps are tracked and which archetypes have zero Jira coverage (a signal, not reassurance).

### Three modes

**Journey** — start here. Phase strip + sidebar + diagram. Best for understanding what the platform does today and where it falls short within a given phase.

**Gap Map** — all 13 tracked gaps across all phases, each with Jira status, severity, delivery target, and affected customers. Use this when you need to answer "is this already filed and when does it ship?"

**PM Lens** — stress-test analyses for specific customer scenarios. Currently populated for: ORC-53014 consult script, Carnival conference vs multi-call, and GM conference survival after patron drops. Each lens asks the questions a PM should ask before committing to a customer.

---

## Who this is for

**New joiners** — the journey strip and the archetype framework give you the mental model of how CXone VC works before you need to read any code or Confluence pages. Start with S1 Basic Inbound, step through each phase, then compare S5 Mid-Call to understand where most of the open work sits.

**PMs and product leads** — when a new customer ask arrives, open the Gap Map, find the relevant phase, check whether there's already a Jira. If there isn't, use the PM Lens as a template for stress-testing the ask before grooming. The archetype filter shows you which interaction patterns have zero gap coverage — those are the unknown-unknowns.

**Claude** — this repo is the knowledge base for answering VC product questions. The scenario archetypes document bounds the full scenario space. The Gap Map links every known gap to a Jira. The capability bar documents platform physics that don't change. When a new customer ask arrives, the first question is which archetype it belongs to and whether its phases have open gaps.

---

## How to view without downloading the repo

GitHub Enterprise does not render HTML inline. Two options:

**Option 1 — htmlpreview (no install needed)**

```
https://htmlpreview.github.io/?https://github.com/inContact/vc-scenarios/blob/main/scenario-library/cxone-scenario-explorer.html
```

Bookmark this URL — it always serves the latest version from `main`. Requires your GitHub session to be authenticated.

**Option 2 — download the single file**

Navigate to `scenario-library/cxone-scenario-explorer.html` in GitHub → click **Raw** → right-click → **Save As** → open in any browser. One file, no dependencies, works offline.

---

## What this repo is not

This is not an engineering QA repo. Bug classification, test sets, and unknown-unknown extrapolation live in [`vc-capability-qa`](https://github.com/inContact/vccapabilityqaclaude). That repo and this one share the same source of truth but serve different audiences. A CAT 3 bug classification in vc-capability-qa maps to archetype S5 or S6 here. A gap in this repo with no CAT classification is a signal to run Flow 2 in vc-capability-qa.

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
