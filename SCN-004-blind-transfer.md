---
id: SCN-004
title: Blind Transfer (Cold Transfer)
category: transfer
status: supported
source: agent-contact-lifecycle/SKILL.md · Agent Events Confluence 17504254 · CLAUDE.md
last_verified: 2026-07-24
---

## Summary
Agent A transfers the patron to a skill or another agent without speaking to the receiving party first. Agent A is immediately released. The patron is re-queued or delivered directly to Agent B. Agent A enters ACW (if skill configured).

## Trigger
Agent A clicks Transfer / executes `TransferCall` action with `IsConsult = false`.

---

## Products Involved

| Product / Component | Role |
|---|---|
| VCSvc — ACDProvider | Re-queues or re-routes contact to target |
| VCSvc — AgentProvider | `TransferCallStrategy` with `IsConsult=false`; releases Agent A |
| VCSvc — IVR Script Engine | Runs reskill/transfer script |
| UIQ | Pushes state changes to both Agent Workspaces |
| Agent Workspace (Agent A) | Shows transfer action, then ACW |
| Agent Workspace (Agent B) | Receives incoming ring notification |
| RRR | Re-evaluates routing rules for new skill/agent |

---

## Journey — State Machine

- Agent A and patron in active call
  - Agent A initiates blind transfer
    - `TransferCallStrategy` fires, `IsConsult = false`
      - Contact re-routed: `CallContactEvent` → Status: **Incoming**, CallType: **ReskillProxy**
        - Agent A leg released
          - `AgentLeg` → Status: **Disconnected**, FinalState: true
          - `AgentState` Agent A → **Unavailable**, IsAcw: true (if skill UseACW=true)
            - ACD selects Agent B from target skill
              - `CallContactEvent` Agent B leg → Status: **Incoming**, CallType: **AgentLeg**
              - `AgentState` Agent B → **InboundContact**
                - Agent B accepts
                  - `CallContactEvent` → Status: **Active**
                  - `ONASSIGNMENT` fires for Agent B
                    - Agent B and patron in active call
                      - Normal call end flow (see SCN-001)

---

## Events

### Emitted by VC → Agent Workspace (via UIQ)

| Event Type | Key Fields | When |
|---|---|---|
| `CallContactEvent` | Status: **Incoming**, CallType: **ReskillProxy** | Contact re-queued after transfer |
| `AgentLeg` | Status: **Disconnected**, FinalState: true | Agent A leg ends |
| `AgentState` | CurrentState: **Unavailable**, IsAcw: true | Agent A enters ACW |
| `AgentState` | CurrentState: **Available** | Agent A ACW complete |
| `CallContactEvent` | Status: **Incoming**, CallType: **AgentLeg** | Ring at Agent B |
| `AgentState` | CurrentState: **InboundContact** | Agent B receiving call |
| `CallContactEvent` | Status: **Active** | Agent B accepts |

---

## State Transitions

| Entity | From State | To State | Trigger |
|---|---|---|---|
| Contact | Active(4) | Incoming (re-queued) | Blind transfer initiated |
| Agent A | InboundContact(3) | Unavailable(2) IsAcw:true | Released after transfer |
| Agent B | Available(1) | InboundContact(3) | Contact delivered |
| Contact | Incoming | Active(4) | Agent B accepts |

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| VC Internal | `TransferCallStrategy.CreateRequest()` | `IsConsult=false`, `SourceContactId`, `SkillId` |
| Studio Script Action | `TRANSFER` | Initiates transfer in IVR script |
| VC Internal | `ApiCallTransferInfo` | `IsConsult=false`, `ValidContactsCounter` checked |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| `ONUNASSIGNMENT` (system) | `System/AgentUI/` | Fires for Agent A on release |
| `ONASSIGNMENT` (system) | `System/AgentUI/` | Fires for Agent B on delivery |

---

## Open Questions / Gaps
- `[CONFIRMED]` `ValidContactsCounter` is checked in `ApiCallTransferInfo` before transfer executes
- `[CONFIRMED]` `IsConsult=false` is the differentiator from warm transfer

---

## Related Scenarios
- SCN-001: Inbound call — happy path
- SCN-005: Warm (consultative) transfer
- SCN-006: Conference call
