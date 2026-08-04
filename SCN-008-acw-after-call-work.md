---
id: SCN-008
title: ACW — After Call Work Lifecycle
category: acw
status: supported
source: agent-contact-lifecycle/SKILL.md · Agent Events Confluence 17504254 · GM transcript 2026-07-23
last_verified: 2026-07-24
---

## Summary
After a voice contact ends, an agent on an ACW-enabled skill enters ACW state. The agent is unavailable for new contacts. The agent sets a disposition code and exits ACW manually or waits for the timer. ACW configuration lives at the skill level, not the agent level. While in ACW, the agent cannot initiate a new outbound voice contact — this is a VC contact-state constraint, not a UX decision.

## Trigger
`RemoteCallDisconnectedStrategy` fires on contact end AND `UseACW = true` on the assigned skill.

---

## Products Involved

| Product / Component | Role |
|---|---|
| VCSvc — ACDProvider | Determines ACW config from skill; holds agent out of routing |
| VCSvc — AgentProvider | `ChangeAgentStateStrategy` maps ACW to Unavailable, routable=false |
| VCSvc — ProviderEntityStateManagement | Notifies EM agent is in ACW (not routable) |
| UIQ | Pushes `AgentState` Unavailable/IsAcw=true to Agent Workspace |
| Agent Workspace | Renders ACW timer, disposition picker |
| WFM | Records ACW duration for adherence reporting |
| SuiteData | `AgentContactMessage` written on ACW completion |

---

## Journey — State Machine

- Contact ends (patron hangs up, agent ends call, or transfer completes)
  - `RemoteCallDisconnectedStrategy` checks skill config
    - `UseACW = true` → ACW begins
      - `AgentState` → CurrentState: **Unavailable**, IsAcw: true
        - `AcwTimeout = skill.ACWTimer` (seconds; 0 = no auto-timeout)
          - Agent Workspace shows ACW panel
            - `UpdateUnavailableCodes` — disposition code list sent
              - `→ [BRANCH A]` Agent sets disposition and clicks Done
                - Disposition recorded
                  - `AgentState` → CurrentState: **Available**
                  - Agent re-enters routing queue
              - `→ [BRANCH B]` ACW timer expires (if AcwTimeout > 0)
                - System auto-exits ACW
                  - `AgentState` → CurrentState: **Available**
              - `→ [BRANCH C]` Agent attempts to make outbound call during ACW
                - `→ [BLOCKED]` VC contact-state constraint: cannot have two active voice contacts
                - Agent must exit ACW first (see SCN-002 for GM failure scenario)
    - `UseACW = false` → Agent goes directly to Available
      - `AgentState` → CurrentState: **Available**

---

## Events

### Emitted by VC → Agent Workspace (via UIQ)

| Event Type | Key Fields | When |
|---|---|---|
| `AgentState` | CurrentState: **Unavailable**, IsAcw: true, AcwTimeout | ACW begins |
| `UpdateUnavailableCodes` | (list of out-reason / disposition codes) | Disposition options sent |
| `AgentState` | CurrentState: **Available** | ACW complete |

### Consumed by VC (External → VC)

| Source | Signal | Effect in VC |
|---|---|---|
| Agent Workspace | Disposition set + Done action | `ChangeAgentStateStrategy` transitions to Available |
| ACW timer | Timeout | Auto-transition to Available |

---

## State Transitions

| Entity | From State | To State | Trigger |
|---|---|---|---|
| Agent | InboundContact(3) | Unavailable(2) IsAcw:true | Call ends, UseACW=true |
| Agent | Unavailable(2) IsAcw:true | Available(1) | Agent clicks Done or timer expires |

ACW state in EM maps to: `AGENT_STATE_UNAVAILABLE` with `routable = false`

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| VC Internal | `RemoteCallDisconnectedStrategy` | Reads `UseACW`, `ACWTimeoutSeconds` from skill; passes to `ChangeAgentStateStrategy` |
| VC Internal | `ChangeAgentStateStrategy` | Sets `AGENT_STATE_UNAVAILABLE`, `routable=false`, `IsAcw=true` |
| VC Internal | `AgentContactMessage` (SuiteData) | Written on ACW completion for reporting |
| REST API | Exit ACW | Agent Workspace calls VC to transition state |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| `Disposition Call.xml` | `System/AgentUI/` | Fires for disposition action during ACW |
| `ONUNASSIGNMENT` (system) | `System/AgentUI/` | Fires at contact end (triggers ACW path) |

---

## Configuration (Skill-Level)

| Setting | Type | Effect |
|---|---|---|
| `UseACW` | boolean | Enables ACW for all contacts on this skill |
| `ACWTimeoutSeconds` (ACWTimer) | int | Auto-exit time; 0 = manual only |

---

## Key Constraint
**An agent cannot initiate an outbound voice contact while in ACW.** This is enforced by VC's contact state machine (not Agent Workspace UX). The agent must exit ACW — resolving the current contact — before a new voice contact can be created. Relevant to: SCN-002 (patron drops unexpectedly), SCN-003 (patron drops mid-conference).

---

## Open Questions / Gaps
- `[CONFIRMED]` ACW config comes from skill, set at `ContactDeliveryCompletedStrategy` execution time
- `[CONFIRMED]` `routable=false` is set in EM during ACW — agent will not receive new contacts
- `[GAP]` No mechanism today to auto-dismiss ACW on patron-drop detection without a triggering event from media server

---

## Related Scenarios
- SCN-001: Inbound call (ACW is final stage)
- SCN-002: Patron drops — ACW blocks callback (GM Scenario 1)
- SCN-003: Both agents in ACW after conference collapse (GM Scenario 2)
- SCN-007: Agent session lifecycle
