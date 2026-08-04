---
id: LAYER2
title: CXone VC — Technical Capabilities & Platform Limits
type: platform-reference
source: ORC-43660 · ORC-22932 · ORC-36819 · AW-57850 · CLAUDE.md · Agent Events Confluence 17504254
last_verified: 2026-07-24
---

# CXone VC — Technical Capabilities & Platform Limits

> This is Layer 2 of the scenario library.
> Layer 1 = what the patron/agent experiences (scenarios).
> Layer 2 = what the platform can actually do, and where the hard stops are.
> These limits are invisible until a customer hits them.

---

## CAP-01: Script Concurrency (MaxSpawnScriptLimit)

| Attribute | Value |
|---|---|
| **Setting name** | `MaxSpawnScriptPerBULimit` (also shown as `MaxSpawnScriptLimit` in WebAdmin) |
| **Default** | 15,000 scripts per BU |
| **Configurable** | Yes — per BU extended setting |
| **Known high-value BUs** | Walmart: 60,000 · Schwab: 60,000 · Many BUs at 30,000 |
| **Enforcement status** | **PARTIALLY BROKEN** — limit not enforced via API/UI spawn paths (ORC-22932) |
| **Log signal** | `_err=Max Script Count Exceeded` in VC logs |
| **Jira** | ORC-22932 (bug) · ORC-36819 (soft-enforce + alerting) |
| **Status** | New — enforcement not yet enabled; depends on FedRAMP gRPC work (ORC-37696) |

### What scenario triggers this
- High-volume Studio SPAWN actions in customer scripts
- Dialer campaigns with aggressive spawn rates
- API-triggered spawns from external systems (web admin button, REST API)

### PM question to ask
> "Does this customer use SPAWN actions in their scripts? What's their peak concurrent script count? Are they above 15,000 today?"

### Known hit: Skopos Financial LLC (C13)
- Spawned a contact every second, accumulated 60,000+ contacts
- BU was configured to 15,000 limit but limit was not enforced via the spawn path being used
- Fix: new gRPC gate from web/API before SQL spawn message

---

## CAP-02: Conference Party Limit

| Attribute | Value |
|---|---|
| **Max parties in a conference** | Up to 8 (per AW-57850 UC-C1) |
| **Controlling setting** | `CanMultiPartyConference` + `EnabledForMCH` in `AgentSessionStart` |
| **Enforcement** | MCH state machine per conference |
| **Status** | Supported (up to 8) — but persistent conference, leave/rejoin NOT supported |

### What breaks above this
- No documented customer has hit the 8-party ceiling
- The real limit today is architectural: conference collapses when ANY patron drops

---

## CAP-03: Multi-Call (Personal Queue) — ORC-43660

| Attribute | Value |
|---|---|
| **Capability name** | Multi-Call Handling (MCH Phase 1 — not to be confused with conference MCH) |
| **What it is** | Agent handles up to N separate concurrent contacts, toggling between them with hold/resume |
| **What it is NOT** | Conference (all parties together) — these are separate capabilities |
| **Media type** | 51 (new, reserved for multi-call) |
| **Agent concurrent limit** | 3 calls maximum (GlobalCluster setting, hard cap) |
| **Delivery mode** | REQAGENT with explicit target agent + MultiCallSkill flag |
| **Jira** | ORC-43660 (Phase 1 capability) · ORC-45908 (delivery) · ORC-45909 (reskill/callback) |
| **Status** | In Progress · Fix version 26.3 / 26.3.1 · Assigned: Mike Winegar / Brendan Johnston |
| **Customer** | Carnival (showstopper) · H&R Block · CSAA |

### Key constraints (from ORC-43660)
- REQAGENT with MultiCallSkill REQUIRES a target agent — no skill-only routing
- Blind transfer (`blindxfer`) to MultiCall skills: BLOCKED
- Reskill to MultiCall skills: ALLOWED (decision changed 2026-03-19)
- Callback on MultiCall skill: ALLOWED (media type 51 routing if target agent present)
- ACD call while on Personal Queue call: agent NOT routable for new ACD contacts until PQ ends
- DID call → hold to answer ACD call: BLOCKED (breaks queue routing discipline)
- Outbound via DID line: BLOCKED (all outbound via main line only)

### ⚠️ PM question: Can I conference while on a multi-call leg?

**Documented gap (ORC-43660 refinement notes):**
> "Scenario: agent is on a conference call with two parties. A third incoming call arrives. Agent tries to park the active conference to answer the new call. Issue: system SPLITS the call — one party (internal conferenced agent) sent to Personal Queue; the other party (original patron) stays live."

**Answer: Conference + Multi-call interaction is a KNOWN EDGE CASE, not fully resolved.**
- Conference is a single multi-leg contact (Joined state, all parties in same MCH group)
- Multi-call is separate concurrent contacts (each in its own script, media type 51)
- Trying to "park" a conference to take a new call: conference structure breaks
- No Studio action exists to park/resume a conference as a whole unit
- This is out of scope for ORC-43660 Phase 1 — customer scripting must handle it

---

## CAP-04: ACW Timer

| Attribute | Value |
|---|---|
| **Setting** | `ACWTimeoutSeconds` (also `ACWTimer`) — per skill |
| **Range** | 0 (manual only, no auto-exit) to configurable max |
| **Default** | Defined per skill — no platform default documented |
| **Extension** | NOT natively supported (ORC-51895 — Services Australia showstopper) |
| **Enforcement** | `ChangeAgentStateStrategy` — routable=false during ACW |

### PM question to ask
> "Does this customer need agents to extend ACW? If yes, what's the max extension time? (SA needs up to 120 min)"

---

## CAP-05: Conference Concurrent Legs (MCH Capacity)

| Attribute | Value |
|---|---|
| **Event** | `MchAgentSettingsChangeEvent` — VoiceThreshold · TotalContactCount · DeliveryMode |
| **VoiceThreshold** | Max voice contacts an agent can handle — set at skill/agent config level |
| **Conference legs** | Up to 8 per AW-57850; MCH state machine enforces |
| **Hold behavior** | Conference legs: HoldConference.xml holds ALL legs simultaneously |
| **Persistent conference** | NOT supported — conference collapses when patron drops |

---

## CAP-06: Script Spawn Path Coverage

| Spawn path | Limit enforced? |
|---|---|
| VC internal SPAWN action in Studio script | Yes (VC checks `MaxSpawnScriptPerBULimit`) |
| SQL stored proc (`IC_SpawnScript`, `insideWS_SpawnScript`) | **NO — ORC-22932 bug** |
| Web admin "Spawn Script" button | **NO — ORC-22932 bug** |
| REST API spawn endpoint | **NO — ORC-22932 bug** |

**Impact**: Customers who spawn via UI or API can bypass the limit silently.
**Fix path**: New gRPC gate from web/API → VC before SQL spawn. Status: unresolved.

---

## CAP-07: Transfer Metadata Continuity

| What transfers | Does it carry across? |
|---|---|
| ANI | ❌ Stripped on warm/consult transfer — CallerID mismatch |
| Studio script variables | ❌ Stripped — new ContactID created |
| CRM data | ❌ Not retained |
| AI context (Agent Assist, Copilot) | ❌ Non-functional for receiving agent |
| MasterID | Shifts with each new contact — no chain surfaced in data lake |
| InteractionID | Constant across legs — but sequence not surfaced in data lake |

---

## CAP-08: Salesforce Voice (Agent Workspace for Salesforce) Coverage

| Capability | Native AW | SF Voice (AW for SF) |
|---|---|---|
| Standard call events (CallContactEvent, AgentState) | ✅ | ✅ via Agent SDK |
| Conference participant roster (ORC-51786 P1) | ✅ | ❓ Needs validation |
| Multi-call (ORC-43660) | ✅ (in progress) | ❓ Not confirmed |
| New callback/bridge-back events (GM ask) | ❌ Not built | ❌ Not built + SDK not ready |
| Agent Assist / Copilot on consult/conference | ❌ Bug ORC-34797 | ❌ Same bug |
| ACW extension button (ORC-51895) | ❌ Not built | ❌ Not built |

> Note: SF Voice uses Agent SDK CTI event subscription. New VC events must be added to the SDK event contract separately. AW-57850 explicitly calls out that SDK was not designed for complex multi-party reconstruction.


---

## CAP-09: IVR LOG Kinesis Stream — Missing Fields

| Attribute | Value |
|---|---|
| **Stream name** | IVR LOG (Kinesis) |
| **What's generated** | DTMF key-press (press path) data on every MENU action |
| **What's missing** | `ACTION_LABEL`, `ACTION_NAME` (unclear if present), `InteractionID` GUID |
| **Data lake join** | Broken — data lake uses regional GUIDs; stream uses `bus_no + contactid` |
| **Scope** | Voice IVR only — digital parity not in scope for ORC-34959 |
| **Fix** | Phase 1: add fields to existing Kinesis stream · CU-worthy · ORC-34959 |
| **Blocker** | ORC-53330 spike (1-day) to confirm volume capacity before enabling |
| **Status** | New · Assigned: Mike Winegar · No fix version set |
| **Jira** | ORC-34959 · ORC-53330 (spike) · DAT-19786 (data governance) |

### PM question to ask
> "Does this customer use IVR press path data for analytics? Which downstream system consumes it? Are they using `bus_no + contactid` joins today or have they migrated to the GUID model?"

---

## CAP-10: Consult Leg Script Context — Blocked

| Attribute | Value |
|---|---|
| **Current behaviour** | No custom script runs on consult leg — Agent B's consult has no Studio context |
| **Blocked capabilities** | Indicate action, post-contact surveys, real-time transcription, AutoSummary, Agent Assist |
| **Scope** | Voice confirmed. Digital: **UNKNOWN — unresolved in grooming** |
| **Script trigger** | Proposed: `BEGIN` action on consult accept. `OnConference()` action does not exist yet |
| **Configuration level** | TBD: BU-wide single script (simplest) vs per-skill vs per-agent |
| **UI dependency** | Agent Workspace team must surface Indicate button on consult leg UI |
| **Jira** | ORC-53014 (surface consult script) · ORC-51894 (Indicate on consult) |
| **Status** | New · No fix version · Services Australia showstopper |
| **Unlocks** | Indicate · Surveys · Transcription · AutoSummary · Agent Assist — ALL for consult/conference |
| **Paired with** | ORC-48255 (change default hangup) — together enable AI Feedback Management hero project |

### What a PM should ask before scoping
> 1. Voice only or digital as well? This is unresolved and doubles the scope if digital is included.
> 2. What level of configurability does the customer need? BU-wide is simplest — is that enough?
> 3. When does BEGIN fire: on consult accept, or on conference join? No OnConference() action exists today.
> 4. Does this need to work in Salesforce Voice? (GM operates in SF Voice — validate separately.)

---

## CAP-11: Feature Toggle Infrastructure (Config Manager Reliability)

| Attribute | Value |
|---|---|
| **Type** | Uptime constraint (not a PM-visible feature limit) |
| **Current behaviour** | Feature toggle checks call Config Manager synchronously on VC scripting threads. Under Config Manager slowness, threads block for up to 100s × retries, draining the scripting thread pool. Cluster goes unresponsive — contacts stop routing, agents show available but receive nothing. |
| **Root causes confirmed** | 6 root causes validated via chaos-test harness against `UserHubAPIRepository` (ORC-52482 description, Darren Solomon). All six: (1) blocking sync-over-async with no request timeout [**most severe**]; (2) no single-flight protection; (3) non-thread-safe Dictionary cache; (4) TTL never resets — thundering herd at ~80-cluster scale; (5) retry amplifies under slowness, no circuit breaker; (6) [positive] toggle VALUES stay correct — availability problem not correctness |
| **Customer-observable symptom** | Contacts not routing. Agents appear available but receive no calls. Unexplained cluster unresponsiveness during Config Manager degradation events. |
| **Breakfix gap** | PR #2636 (AAD-43679) fixed root causes 3+4 (thread safety + resettable TTL) but was **never forward-merged** to MAIN, DEVELOP, or RELEASE/70.1. Production still runs the buggy code. This fix exists in a dead branch. |
| **Phases affected** | p2 (IVR + Queue), p3 (Agent accepts) — phases where scripting threads are actively held |
| **Required changes** | Non-blocking toggle checks (P0); request timeout (P0); single-flight protection; ConcurrentDictionary; resettable jittered TTL; circuit breaker; Originating-Service-Identifier header |
| **Jira** | ORC-52482 (Capability — New, no fix version) · AAD-43679 (breakfix PR — not merged) |
| **Status** | New · No assignee · No fix version |

### PM question to ask
> "If this customer is reporting intermittent cluster unresponsiveness or contact routing failures that trace to no code bug — check whether a Config Manager degradation event coincided. ORC-52482 root causes 1–5 remain open in production. Root causes 3+4 have a fix in PR #2636 (AAD-43679) but it has never shipped. Is that PR still viable for forward-merge, or has MAIN diverged past it?"

