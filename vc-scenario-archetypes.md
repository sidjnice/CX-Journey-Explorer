# VC Scenario Archetypes — Foundation Document

**Purpose:** Defines the canonical interaction patterns (archetypes) that bound the full scenario
space for CXone VC. This is the foundation for Rovo's coverage assessment and the scenario
explorer's scenario library. Written from two framings simultaneously:
- **Enterprise framing** — the contact center operator configuring and running CXone
- **Customer ask framing** — the patron interacting with the contact center

**Last updated:** 2026-07-24
**Owner:** Siddharatha Joshi

---

## Why Archetypes, Not Combinations

The theoretical combination space across all building block axes exceeds 1,000 states.
Most combinations are invalid — not every building block can co-exist with every other.
Rather than enumerate all combinations, we define archetypes: canonical interaction patterns
that each represent a family of combinations. Scenarios in the explorer are instances of
archetypes. Gaps map to archetypes.

The constraint graph (Layer 0 below) is what makes this tractable. Once platform physics
are established, invalid combinations collapse automatically and the meaningful scenario
space reduces to ~10 archetypes covering ~80-120 distinct scenarios. [Guessing on final count]

---

## Building Block Axes

These are the dimensions that vary across scenarios. Not all axes are independent —
constraints between them are defined in Layer 0.

| Axis | Values |
|---|---|
| Contact origination | Inbound / Outbound / Callback / Transfer-in / Personal Queue re-queue |
| Channel | Voice / Digital (DFO) |
| Skill assignment model | Single-skill agent / Multi-skill agent / Primary+Secondary (Queue Health model) |
| Queue state at offer | Empty / Normal / Overflow threshold breached / Below Queue Health buffer |
| Routing strategy on skill | None / Bullseye (overflow) / RRR proficiency rule / Queue Health — mutually exclusive per skill |
| Script layer executing | System script (always) / Custom script extension (enterprise-added) / No extension point (gap) |
| Script execution phase | IVR pre-answer / Queue wait / Agent answer / Mid-call action / Post-call ACW |
| Mid-call action | None / Hold / Consult / Conference / Blind transfer / Warm transfer / Personal Queue (multi-call) |
| Infrastructure state | Healthy / Config Manager degraded (CAP-11) |

---

## Layer 0 — Platform Physics (Immutable Constraints)

These rules are always true regardless of enterprise configuration. They are not PM decisions.
They eliminate the majority of invalid axis combinations before scenario construction begins.

**Rule 1 — One active script per contact leg**
A contact has exactly one active script executing at any point in time. BEGIN starts every
script. There is no concept of two scripts running in parallel on the same leg.
[Certain — from Studio Action Basics help page, read 2026-07-24]

**Rule 2 — System scripts and custom scripts are layered, not alternatives**
System scripts execute the platform's own lifecycle skeleton — they handle voicemail delivery,
hold mechanics, transfer mechanics, and other foundation events. They always run and cannot
be replaced by the enterprise. Custom scripts extend the capability the system script creates
by adding business logic on top. The enterprise never substitutes a system script — they add
a custom script that executes within the context the system script exposes.
Implication: a gap in custom script capability is a missing extension point in the system
script's context, not a missing system script.
[Certain — from Voice Mail System Scripts Confluence page 483755330, read 2026-07-24;
corrected from session reasoning by Siddharatha Joshi 2026-07-24]

**Rule 3 — Consult leg has no custom script extension point**
The consult leg's system script runs, but it does not create a context that a custom script
can extend. This means no Indicate, no surveys, no transcription, no AutoSummary, no Agent
Assist on consult or conference legs. This is not a missing system script — it is a missing
extension point. The gap is ORC-53014.
[Certain — from ORC-53014, read earlier sessions]

**Rule 4 — Event actions terminate current execution, they do not branch from it**
When an event action fires (OnAnswer, OnHold, OnRelease, OnTransfer), it terminates whatever
script execution was happening and the script proceeds from the event action. Event actions
are not sub-branches — they are independent execution contexts triggered by platform events.
Implication: you cannot "intercept" an event mid-execution. The script either handles the
event or it doesn't.
[Certain — from Studio Action Basics help page, read 2026-07-24]

**Rule 5 — Bullseye and RRR are mutually exclusive per skill**
A skill can have either Bullseye (overflow) routing or RRR proficiency rules active, not both.
The RRR Confluence page explicitly requires conflicting engines to be disabled for skills where
RRR is applicable.
[Certain — from RRR MVP Confluence page 1556120293, read 2026-07-24]

**Rule 6 — Queue Health operates on agents, not contacts**
Queue Health protects Primary skill capacity by temporarily deactivating Secondary skills on
selected agents. It does not modify contact proficiency. Contacts already in queue and matched
to an agent are not affected by Queue Health firing — the effect is on future routing eligibility
of the protected agents.
[Likely — from Queue Health Confluence page 3014623684, read 2026-07-24;
exact behavior of in-flight matches during Queue Health activation is unresolved in that page]

**Rule 7 — Voice and digital do not mix mid-interaction**
A contact is on exactly one channel. There is no mid-interaction channel switch.
[Certain]

---

## Layer 1 — Enterprise Configuration Choices

These are decisions the enterprise makes before any contact arrives. They define which
archetypes are possible for that deployment.

- Which routing strategy is active per skill: None / Bullseye / RRR (mutually exclusive)
- Whether Queue Health is enabled and which agents have Primary skill designations
- What custom scripts are written and which extension points they use
- Which media types are enabled per skill (voice, digital, or both)
- ACW timer duration per skill (no agent extension mechanism — CAP-04)

Layer 1 collapses the axis space significantly. Most enterprises configure one routing
strategy and one script model per skill. The intersection of routing strategy + skill
assignment model + script extension scope defines the scenario family for that deployment.

---

## Layer 2 — Runtime Variables

These vary contact by contact within the same Layer 1 configuration.

- Queue depth at the moment of offer
- Agent state at the moment of offer (Available / ACW / Personal Queue occupied)
- What mid-call action the agent takes (if any)
- Whether Config Manager is healthy at the time of execution (CAP-11)

---

## The 10 Archetypes

### S1 — Basic Inbound Queue *(the atomic unit)*

**What it is:** Patron calls → IVR custom script runs → Reqagent → agent answers → ACW → END

**Enterprise framing:** Single-skill agents, no routing rules active, healthy infrastructure.
The simplest possible configuration. This is the baseline every other archetype deviates from.

**Customer ask framing:** "I called in, pressed some menu options, waited, an agent answered."

**Routing strategy:** None (default ACD routing)
**Script layer:** System script + custom IVR script extension
**Mid-call action:** None
**Gap exposure:** None for the basic path. ACW timer gap (CAP-04) applies if agent needs
more time. MaxSpawnScriptLimit (CAP-01) applies at scale.

**Why it matters:** Every other archetype inherits this structure. If S1 is broken,
everything is broken. It is the sanity check scenario.

---

### S2 — Overflow / Bullseye Active

**What it is:** Queue depth exceeds threshold → Bullseye expands proficiency range →
contacts now routable to agents who weren't previously eligible

**Enterprise framing:** Multi-skill agents configured. Bullseye overflow rule active on skill.
Queue analyst (Mike in the Queue Health example) creates overflow rule to reduce abandons.

**Customer ask framing:** "I've been waiting a long time. The system found me an agent who
doesn't usually handle my type of call."

**Routing strategy:** Bullseye (mutually exclusive with RRR on same skill)
**Script layer:** Same as S1 — Bullseye operates at the routing layer, not the script layer
**Mid-call action:** Any (Bullseye doesn't constrain mid-call behavior)
**Key variable:** Does the matched agent have a Primary skill designation under Queue Health?
If yes, Queue Health and Bullseye are interacting on the same skill — open question whether
this is a supported configuration.

**Gap exposure:** No current Jira for Bullseye + Queue Health interaction. Potential unknown-unknown.

---

### S3 — RRR Proficiency Adjustment Mid-Queue

**What it is:** Contact enters queue → RRR rule evaluates queue statistics →
contact proficiency updated → different agent pool now eligible

**Enterprise framing:** RRR active on skill. Rule fires at evaluation interval (configurable,
minimum ~120s per Queue Health page). Contact may have been waiting before the rule fired.

**Customer ask framing:** "I waited a while and then suddenly got connected — felt like
something changed about how they were routing me."

**Routing strategy:** RRR (mutually exclusive with Bullseye on same skill)
**Script layer:** RRR is a routing layer change — the IVR script continues executing
independently during the queue wait. RRR sends contact update to AOR.
**Key variable:** Was the contact already matched to an agent before RRR fired?
Race condition between RRR contact update and AOR match decision is unresolved.
[Guessing — no Jira found for this specific race condition]

**Gap exposure:** Potential race condition (no Jira). RRR reporting for proficiency updates
not yet implemented (from RRR Confluence: "We have yet to implement reporting for skill
level proficiency updates from the Bullseye rules").

---

### S4 — Queue Health Buffer Reserved

**What it is:** Queue Health detects Primary skill KPIs degrading → deactivates Secondary
skills on N protected agents → those agents now handle only Primary skill contacts

**Enterprise framing:** Queue Health enabled. Primary/Secondary skill designations assigned
to agents. Rohan (from Queue Health example) created a Queue Health rule protecting Sales
Support skill capacity.

**Customer ask framing (Primary skill patron):** "I called Sales Support and got through quickly
even though the contact center seemed busy."

**Customer ask framing (Secondary skill patron):** "I was waiting for Tech Support and an agent
who could have helped me wasn't available — they were being held for something else."

**Routing strategy:** Queue Health (operates alongside either Bullseye or RRR — interaction
between Queue Health and RRR on same skill is an open question)
**Script layer:** Queue Health operates at agent eligibility layer via VC/State Engine.
No script changes — system script runs normally. Custom script unaffected.
**Key variable:** What happens to contacts already matched to an agent whose Secondary skill
is deactivated by Queue Health mid-match? Not resolved in Confluence page.

**Gap exposure:** Primary skill assignment ownership unresolved (ACD vs Routing Center —
open question from Queue Health Confluence page, as of Feb 2026). Queue Health is entirely
absent from the current scenario explorer — this is a new archetype with no existing coverage.

---

### S5 — Mid-Call: Hold / Consult / Conference / Transfer

**What it is:** Agent accepts contact → takes a mid-call action

**Enterprise framing:** This is the highest-complexity archetype. The sub-variants have
very different capability profiles:

| Sub-variant | Script context on new leg | Gap |
|---|---|---|
| Hold | System script handles hold music. Custom script continues on main leg. | CAP-04 (ACW timer can't be extended post-hold) |
| Consult | System script runs on consult leg. **No custom script extension point.** | ORC-53014 (CAP-10) |
| Conference | System script manages conference. Party limit: max 8. MCH capacity constraint. | CAP-02, ORC-43660, CAP-05 |
| Blind transfer | Script ends on transfer. No continuity of script context to target. | CAP-07 (metadata continuity) |
| Warm transfer | Consult leg first (no script context), then transfer. | ORC-53014 + CAP-07 |

**Customer ask framing:** "The agent put me on hold / brought in a specialist / transferred me."

**Routing strategy:** Any — routing strategy doesn't constrain mid-call behavior
**Script layer:** This is where Layer 0 Rule 2 and Rule 3 matter most.
System script handles each sub-variant. Custom scripts can only extend where extension
points exist. Consult and conference legs have no extension points today.

**Gap exposure:** Highest gap density of any archetype.
ORC-53014, ORC-43660, CAP-02, CAP-05, CAP-07, CAP-10 all originate here.

---

### S6 — Personal Queue / Multi-Call

**What it is:** Agent already handling a contact → second contact offered via Personal Queue

**Enterprise framing:** Multi-call enabled for skill (media type 51, GlobalCluster setting).
Maximum 3 concurrent contacts per agent (CAP-03). Agent is simultaneously active on
Contact A and receives offer for Contact B.

**Customer ask framing:** "I was put on hold while the agent handled something else.
Then they came back to me."

**Routing strategy:** Any — but RRR does not know about Personal Queue occupancy state.
[Guessing — not confirmed in any read Jira or Confluence; needs engineering validation]
**Script layer:** Each contact has its own script execution. Scripts run independently.
The interaction between scripts when agent is multi-call is not documented in sources read.

**Gap exposure:** CAP-03 (limit: 3 concurrent). MCH race condition when agent is on
conference AND receives Personal Queue contact (ORC-43660 edge case — needs Phase 2 Jira).

---

### S7 — Callback

**What it is:** Patron requests callback → leaves queue → callback contact created →
re-enters routing as an outbound-flavored inbound contact

**Enterprise framing:** Callback configured on skill. Patron opts out of hold wait.
System creates a callback contact. When agent becomes available, system dials patron.

**Customer ask framing:** "I didn't want to wait so I asked them to call me back."

**Routing strategy:** Unknown — does callback contact re-evaluate against active RRR/Bullseye
rules when it re-enters routing? [Guessing — not covered in sources read this session]
**Script layer:** Callback uses a different script path from standard inbound.
Whether custom script extension points are the same as S1 is unconfirmed. [Guessing]
**Key variable:** Does Queue Health apply to callback contacts? If Primary skill agents are
protected, does that affect callback offer timing? [Guessing]

**Gap exposure:** No current gap Jiras for callback-specific scenarios in our tracker.
This archetype has the most [Guessing] tags — least well-understood in current knowledge base.

---

### S8 — Outbound / Preview

**What it is:** Enterprise initiates contact → script runs → agent previews contact
record → system dials → connects

**Enterprise framing:** Outbound campaign configured. Agent receives preview of contact
data before dial. OnPreview event fires instead of OnAnswer family.

**Customer ask framing:** "They called me." (patron is reactive, not initiating)

**Routing strategy:** Outbound routing is separate from inbound skill routing.
Bullseye/RRR/Queue Health do not apply to outbound contacts. [Likely]
**Script layer:** System script handles outbound dial mechanics. Custom script handles
what to do on connect, no-answer, voicemail detection.
**Key variable:** If agent is mid-outbound and receives inbound Personal Queue contact,
which script context takes priority? [Guessing]

**Gap exposure:** No current gap Jiras specific to outbound in our tracker.
IVR LOG Kinesis missing fields (ORC-34959) may apply to outbound records — unconfirmed.

---

### S9 — Custom Script Extension Boundary

**What it is:** The enterprise's custom script attempts to use a capability that requires
an extension point the system script does not expose.

**Enterprise framing:** Script writer adds an action (Indicate, AutoSummary, custom survey)
expecting it to work in a specific execution context — consult leg, conference leg, ACW.
The action either silently does nothing or throws an error because the system script
did not create that extension point.

**Customer ask framing:** "The agent said they'd get me a survey but I never received one."
Or: "The agent's notes from the conference call were missing."

**Routing strategy:** Any
**Script layer:** This is the archetype defined by Layer 0 Rule 2 and Rule 3.
The gap is always a missing extension point, not a missing action.
**Key variable:** Which extension points exist today vs which are gaps?

**Gap exposure:** ORC-53014 (consult leg — no extension point).
This archetype is the correct framing for the "consult script context" problem.
It generalizes: any future capability request that involves a new execution context
(e.g. post-conference ACW, warm transfer handoff data) is an instance of S9.

---

### S10 — Infrastructure Degraded Overlay

**What it is:** Any of S1–S9 executing during Config Manager degradation.

**Enterprise framing:** Contact center is running normally. Config Manager slows.
Feature toggle checks begin blocking scripting threads. Cluster becomes unresponsive.
Contacts stop routing. Agents show available but receive nothing. No code bug — pure
infrastructure reliability failure.

**Customer ask framing:** "I called and nothing happened. The call just sat there."

**Routing strategy:** Any
**Script layer:** Any — the failure is at the infrastructure layer, below script execution.
**Key constraint:** This is not a script-level gap. It's CAP-11. The fix requires
PR #2636 (AAD-43679) forward-merge to MAIN plus additional hardening (circuit breaker,
non-blocking checks, single-flight protection). 5 of 6 root causes remain open in production.

**Gap exposure:** CAP-11 (ORC-52482). Breakfix exists but was never merged to production.


---

### S11 — Post-Call Activity

**What it is:** Contact ends → agent enters ACW (After Call Work) → optional post-call
survey executes → agent released back to available

**Enterprise framing:** Two distinct sub-phases with separate capability profiles:

| Sub-phase | Script context | Gap |
|---|---|---|
| ACW | Timer-controlled. Agent performs wrap-up work. No script extension during ACW itself — timer is set at skill level. Agent cannot extend it. | CAP-04 (no extension mechanism) |
| Post-call survey | Requires a script to execute after contact ends but before agent is released. Extension point must exist in system script to hand off to custom survey script. | Unknown — extension point existence unconfirmed [Guessing] |

**Customer ask framing:** "After the call I got a survey asking how the agent did."
Or from the enterprise: "Our agents need more wrap-up time for complex calls but the
system cuts them off."

**Routing strategy:** None — post-call activity occurs after routing is complete
**Script layer:** System script manages ACW timer. Post-call survey requires a custom
script extension point in the post-call execution context. Whether this extension point
exists cleanly or has the same gap as the consult leg (S9) is unconfirmed. [Guessing]
**Key variable:** Does the post-call survey script run on the agent leg, the contact leg,
or both? If the contact has already disconnected, which execution context owns the survey?

**Gap exposure:** CAP-04 (ACW timer — no extension). Post-call survey extension point
not yet in gap tracker — potential S9-class gap (missing extension point).

**Why it's an archetype, not a characteristic of S1:**
Post-call activity has its own execution phase, its own timer model, and its own gap
profile. Treating it as a property of S1 would mean we never ask "what gaps exist
specifically in the post-call execution context?" — which is a real and distinct question.

---

## What This Means for Rovo

Rovo's job is to assess coverage against these 10 archetypes using Jira and Confluence.
The assessment question for each archetype is not "does a Jira exist" but
"does the Jira's fix mechanism address the gap this archetype exposes."

Coverage mapping (current state):

| Archetype | Jira coverage | Coverage quality |
|---|---|---|
| S1 Basic Inbound | ORC-22932 (CAP-01 at scale) | Partial — basic path covered, limit gap tracked |
| S2 Bullseye | No gap Jira for Bullseye+Queue Health interaction | Unknown — potential unknown-unknown |
| S3 RRR | No race condition Jira | Unknown — race condition unJira'd |
| S4 Queue Health | No Jira in current gap tracker | Not covered — new archetype |
| S5 Mid-Call | ORC-53014, ORC-43660, CAP-02, CAP-05, CAP-07 | Partial — multiple open gaps |
| S6 Personal Queue | ORC-43660 (partial), CAP-03 | Partial — MCH edge case unJira'd |
| S7 Callback | None | Not covered — unknown |
| S8 Outbound | None | Not covered — assumed low gap exposure |
| S9 Extension Boundary | ORC-53014 | Partial — only consult leg documented |
| S10 Infra Degraded | ORC-52482 (CAP-11) | Tracked — fix unmerged |
| S11 Post-Call Activity | ORC-51895 (CAP-04 — ACW timer) | Partial — ACW tracked, post-call survey extension point not in tracker |

---


## Open Questions Before Rovo Assessment

1. Does Queue Health interact with Bullseye or RRR when both are configured on overlapping skills?
2. Does RRR evaluate Personal Queue occupancy when making proficiency updates?
3. Does Callback re-enter routing against active RRR/Bullseye rules?
4. Does Queue Health apply to callback contacts?
5. What happens to contacts already matched to an agent when Queue Health fires mid-match?
6. Are the extension point gaps in S9 limited to consult/conference legs, or do other execution contexts also lack extension points?
7. Does the post-call execution context expose a clean extension point for custom survey scripts, or does it have the same missing extension point problem as the consult leg (S9)?
8. When the contact disconnects in S11, which leg owns the post-call survey execution — agent, contact, or a new system-created context?




## Characteristics vs Archetypes — Decision Rule

A building block is an **archetype** if it has its own execution context with distinct
gap exposure. It is a **characteristic** if it is a configurable property within an
existing execution context.

Examples:
- **Post-call activity (S11)**: Archetype — own execution phase, own timer model, own gap profile
- **Music during hold/transfer**: Characteristic of S5 — system script property,
  configurable (which audio plays) but the execution model is fixed. The Music action
  plays during hold because the system script sets that up. Enterprise configures the
  content, not the context. No distinct gap exposure beyond S5's existing gaps.
- **Queue depth at offer time**: Characteristic — runtime variable in Layer 2, not a
  distinct execution context
- **Infrastructure degraded (S10)**: Archetype — it overlays all other archetypes but
  has its own distinct gap (CAP-11) and its own fix requirement independent of any script

## Source URLs (for Rovo and future session reference)

| Source | URL | Content | Confidence |
|---|---|---|---|
| RRR MVP Architecture | https://nice-ce-cxone-prod.atlassian.net/wiki/spaces/IN/pages/1556120293 | Routing strategy constraints, skill queue segments, RRR+Bullseye exclusivity | [Certain] |
| Queue Health Problem Statement | https://nice-ce-cxone-prod.atlassian.net/wiki/spaces/IN/pages/3014623684 | Queue Health architecture, Primary/Secondary skill model, VC integration approach | [Certain] |
| Voice Mail System Scripts | https://nice-ce-cxone-prod.atlassian.net/wiki/spaces/IN/pages/483755330 | System script categories (AgentUI, Callers, VoiceMail), 12 voice scripts | [Certain] |
| Studio Action Basics | https://help.nicecxone.com/content/studio/fundamentals/actionbasicscx.htm | Action categories, event action behavior, terminating actions, BEGIN | [Certain] |
| Studio Work with Actions | https://help.nicecxone.com/content/studio/fundamentals/workwithactionsinscriptscx.htm | Connector types, branch conditions, custom conditions | [Certain] |
| Agent Events List | https://nice-ce-cxone-prod.atlassian.net/wiki/pages/viewpage.action?pageId=17504254 | All event types, Status/CallType/CurrentState enums, Long/Short Lived | [Certain] |

---
