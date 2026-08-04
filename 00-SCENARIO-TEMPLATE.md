# CXone VC Scenario Library — Template & Schema

> This template defines the structure for every scenario in this library.
> Each scenario = one markdown file. File naming: `SCN-NNN-slug.md`

---

## Frontmatter (every file starts with this)

```yaml
---
id: SCN-NNN
title: <Human-readable scenario name>
category: <inbound-voice | outbound-voice | conference | transfer | acw | session | digital | disconnect>
status: <supported | partial | unsupported | unknown>
source: <repo skill / confluence page / transcript / jira ticket>
last_verified: YYYY-MM-DD
---
```

---

## Section Schema

### 1. Summary
One paragraph. What is this scenario. Who initiates it. What the expected outcome is.

### 2. Trigger
What starts this scenario. Could be a patron action, agent action, system event, or script action.

### 3. Products Involved
Table format:

| Product / Component | Role in this scenario |
|---|---|
| VCSvc (ACDProvider) | ... |
| Agent Workspace | ... |
| UIQ | ... |

### 4. Journey — State Machine
Nested bullet list showing the state transitions from trigger to completion.
Each node = a state or decision point.
Branch nodes prefixed with `→ [BRANCH]`.
Unknown/unresolved nodes prefixed with `→ [UNKNOWN]`.
Problem nodes prefixed with `→ [GAP]`.

### 5. Events

#### Emitted (VC → Agent Workspace via UIQ)
| Event Type | Key Fields | When |
|---|---|---|
| `CallContactEvent` | Status: Active, CallType: Regular | On agent accept |

#### Consumed (External → VC)
| Source | Signal | Effect in VC |
|---|---|---|
| PSTN / Carrier | SIP BYE | Patron disconnect detected |

### 6. State Transitions

| Entity | From State | To State | Trigger |
|---|---|---|---|
| Contact | Inbound(6) | Active(4) | Agent accepts |
| Agent | Available(1) | InboundContact(3) | Contact delivered |

State values sourced from:
- Contact states: `inContact.DSL.DataModels.VC.ACDContactState.ContactState`
- Agent states: `ScriptHelper.AgentState`

### 7. APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| Studio Script Action | `REQAGENT` | Queue contact for routing |
| VC Internal | `AgentProvider.AcceptContact()` | Process agent accept |
| Agent SDK | — | — |
| REST API | — | — |

### 8. IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| `OnAssignment.xml` | `System/AgentUI/` | Fires on contact delivery |
| `Disposition Call.xml` | `System/AgentUI/` | Fires on ACW |

### 9. Open Questions / Gaps
Bullet list. Each question tagged `[UNKNOWN]`, `[GAP]`, or `[CONFIRMED]`.

### 10. Related Scenarios
Links to other scenario files in this library.
