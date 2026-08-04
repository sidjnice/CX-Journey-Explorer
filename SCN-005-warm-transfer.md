---
id: SCN-005
title: Warm Transfer (Consultative Transfer)
category: transfer
status: supported
source: agent-contact-lifecycle/SKILL.md · Agent Events Confluence 17504254 · CLAUDE.md
last_verified: 2026-07-24
---

## Summary
Agent A places the patron on hold, dials Agent B (or a skill), speaks privately with Agent B, then transfers the patron to Agent B. Agent A is released only after the transfer completes. The patron briefly holds while Agent A and Agent B consult. Also called "consultative transfer."

## Trigger
Agent A dials Agent B / skill via `DialAgent` or `DialSkill` action (`IsConsult=true`), then executes `TransferCall`.

---

## Products Involved

| Product / Component | Role |
|---|---|
| VCSvc — ACDProvider | Creates second (consult) contact leg; manages transfer |
| VCSvc — AgentProvider | `TransferCallStrategy` with `IsConsult=true`; tracks `ConsultCount` |
| VCSvc — ProviderESN or ProviderMRM | Manages hold and second audio leg |
| UIQ | Pushes consult + transfer state changes to both Agent Workspaces |
| Agent Workspace (Agent A) | Shows consult UI, hold indicator, transfer action |
| Agent Workspace (Agent B) | Receives consult ring notification |
| RRR | Skill matching for `DialSkill` variant |

---

## Journey — State Machine

- Agent A and patron in active call
  - Agent A initiates consult via `DialAgent` or `DialSkill`
    - Patron placed on hold
      - `CallContactEvent` patron → Status: **Holding**
      - Second consult contact leg created
        - `CallContactEvent` consult leg → Status: **Active**, CallType: **Consult**
        - `AgentState` Agent A → **OutboundConsult**
          - Agent B receives consult ring
            - `AgentState` Agent B → **InboundConsult**
              - `→ [BRANCH]` Agent B accepts consult
                - Agent A and Agent B speak privately (patron on hold)
                  - Agent A executes `TransferCall` (`IsConsult=true`)
                    - `TransferCallStrategy` fires
                      - Patron leg transferred to Agent B
                        - Agent A released from both legs
                          - `AgentLeg` Agent A → **Disconnected**, FinalState: true
                          - `AgentState` Agent A → **Unavailable**, IsAcw: true (if UseACW=true)
                            - Patron and Agent B now in active call
                              - `CallContactEvent` → Status: **Active** for patron + Agent B
                              - Normal call end flow (see SCN-001)
              - `→ [BRANCH]` Agent B does not accept (timeout / refusal)
                - Consult leg cancelled
                - Patron still on hold with Agent A
                - Agent A can retry or release patron back to queue

---

## Events

### Emitted by VC → Agent Workspace (via UIQ)

| Event Type | Key Fields | When |
|---|---|---|
| `CallContactEvent` | Status: **Holding**, ContactID (patron) | Patron placed on hold |
| `CallContactEvent` | Status: **Active**, CallType: **Consult** | Consult leg created |
| `AgentState` | CurrentState: **OutboundConsult** | Agent A dialling out |
| `AgentState` | CurrentState: **InboundConsult** | Agent B receiving consult |
| `CallContactEvent` | Status: **Active** (consult leg) | Agent B accepts consult |
| `CallContactEvent` | Status: **Disconnected** (consult leg) | Transfer completes; consult leg ends |
| `AgentLeg` | Status: **Disconnected**, FinalState: true | Agent A released |
| `AgentState` | CurrentState: **Unavailable**, IsAcw: true | Agent A enters ACW |
| `CallContactEvent` | Status: **Active** (patron leg, now Agent B) | Transfer complete |

### Consumed by VC (External → VC)

| Source | Signal | Effect in VC |
|---|---|---|
| Agent Workspace | `DialAgent` / `DialSkill` action | Second consult leg created |
| Agent Workspace | `TransferCall` (IsConsult=true) | `TransferCallStrategy` fires |

---

## State Transitions

| Entity | From State | To State | Trigger |
|---|---|---|---|
| Contact (patron) | Active(4) | Hold(8) | Agent A initiates consult dial |
| Contact (consult) | — | Active(4) CallType:Consult | Consult leg created |
| Agent A | InboundContact(3) | OutboundConsult(6) | Dialling Agent B |
| Agent B | Available(1) | InboundConsult(5) | Consult delivered |
| Agent A | OutboundConsult(6) | Unavailable(2) IsAcw:true | Transfer completes |
| Contact (patron) | Hold(8) | Active(4) | Transfer complete, Agent B takes over |
| Agent B | InboundConsult(5) | InboundContact(3) | Patron transferred to Agent B |

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| VC Internal | `TransferCallStrategy.CreateRequest()` | `IsConsult=true`, `SourceContactId`, `DestContactId` |
| VC Internal | `ApiCallTransferInfo` | `IsConsult=true`, `ConsultCount`, `ValidContactsCounter` |
| Studio Script Action | `DialAgent` / `DialSkill` | Creates second leg |
| Studio Script Action | `TRANSFER` (with IsConsult=true) | Initiates warm transfer |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| `ONASSIGNMENT` (system) | `System/AgentUI/` | Fires for Agent B on delivery |
| `ONUNASSIGNMENT` (system) | `System/AgentUI/` | Fires for Agent A on release |

---

## Open Questions / Gaps
- `[CONFIRMED]` `IsConsult=true` differentiates from blind transfer
- `[CONFIRMED]` `ConsultCount` tracked in `ApiCallTransferInfo` to manage multi-consult scenarios

---

## Related Scenarios
- SCN-004: Blind transfer
- SCN-006: Conference call (warm transfer + merge = conference)
- SCN-001: Inbound call — happy path
