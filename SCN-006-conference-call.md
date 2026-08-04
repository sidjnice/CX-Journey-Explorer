---
id: SCN-006
title: Conference Call — 3-Way (Happy Path)
category: conference
status: supported
source: agent-contact-lifecycle/SKILL.md · Agent Events Confluence 17504254 · CLAUDE.md
last_verified: 2026-07-24
---

## Summary
Agent A is on an active call with the patron. Agent A dials Agent B (consult leg), then merges all three into a conference. All parties can hear each other. Any agent can leave; when all agents leave or the patron disconnects, the conference ends normally. MCH (Multi-party Call Handling) permission required.

## Trigger
Agent A already on call with patron. Agent A initiates `DialAgent` / `DialSkill`, then calls `APIConferenceCall` to merge all legs.

---

## Products Involved

| Product / Component | Role |
|---|---|
| VCSvc — ACDProvider | MCH state machine manages all conference legs |
| VCSvc — AgentProvider | `ConferenceStatusChangedStrategy`; `APIConferenceCall` |
| VCSvc — ProviderESN or ProviderMRM | 3-way audio bridge |
| VCSvc — ProviderEntityStateManagement | Notifies EM of all state changes for all legs |
| UIQ | Pushes `Joined` state and conference events to all Agent Workspaces |
| Agent Workspace (Agent A) | Shows conference controls (hold, leave, transfer) |
| Agent Workspace (Agent B) | Shows conference controls |

---

## Journey — State Machine

- Agent A and patron in active call
  - Agent A dials Agent B (`DialAgent` or `DialSkill`)
    - Patron placed on hold
      - `CallContactEvent` patron → Status: **Holding**
      - Consult leg created
        - `CallContactEvent` → Status: **Active**, CallType: **Consult**
        - `AgentState` Agent A → **OutboundConsult**
        - `AgentState` Agent B → **InboundConsult**
          - Agent B accepts consult
            - Agent A calls `APIConferenceCall`
              - `ConferenceStatusChangedStrategy.CreateRequest()` fires with `ConferenceId`
                - MCH validates `CanMultiPartyConference` permission from `AgentSessionStart`
                  - 3-way bridge established
                    - `CallContactEvent` → Status: **Joined** (all legs)
                    - `MchAgentSettingsChangeEvent` — thresholds re-evaluated
                      - All three parties in conference
                        - `→ [BRANCH A]` Normal end — patron hangs up voluntarily
                          - `CallContactEvent` patron → **Disconnected**
                          - Conference collapses (current behaviour — see SCN-003 for unexpected drop)
                          - Both agents → ACW
                        - `→ [BRANCH B]` Agent A leaves conference
                          - Agent A leg → **Disconnected**
                          - `AgentState` Agent A → ACW or Available
                          - Agent B and patron continue in 2-way call
                        - `→ [BRANCH C]` Conference placed on hold
                          - `HoldConference` system script fires
                          - `CallContactEvent` all legs → **Holding**
                          - `ResumeConference` to restore
                        - `→ [BRANCH D]` Conference transferred
                          - `TransferConference` system script fires
                          - New agent or skill added; Agent A/B may be released

---

## Events

### Emitted by VC → Agent Workspace (via UIQ)

| Event Type | Key Fields | When |
|---|---|---|
| `CallContactEvent` | Status: **Holding** (patron) | Patron held while consult dialled |
| `CallContactEvent` | Status: **Active**, CallType: **Consult** | Consult leg created |
| `AgentState` | CurrentState: **OutboundConsult** | Agent A dialling |
| `AgentState` | CurrentState: **InboundConsult** | Agent B receiving |
| `CallContactEvent` | Status: **Joined** (all legs) | Conference fully formed |
| `MchAgentSettingsChangeEvent` | VoiceThreshold, TotalContactCount, DeliveryMode | MCH capacity check |
| `CallContactEvent` | Status: **Holding** (all legs) | Conference on hold |
| `CallContactEvent` | Status: **Disconnected** (per leg) | Leg leaves conference |
| `AgentState` | CurrentState: **Unavailable**, IsAcw: true | Per-agent ACW |

---

## State Transitions

| Entity | From State | To State | Trigger |
|---|---|---|---|
| Contact (patron) | Active(4) | Hold(8) | Agent A initiates consult |
| Contact (consult leg) | — | Active(4) CallType:Consult | Consult created |
| Agent A | InboundContact(3) | OutboundConsult(6) | Dialling Agent B |
| Agent B | Available(1) | InboundConsult(5) | Consult delivered |
| All contacts | Active(4)/Consult | Conference(14) | `APIConferenceCall` merges legs |
| Contact (patron) | Conference(14) | Disconnected | Patron hangs up |
| Agent A/B | InboundContact(3) | Unavailable(2) IsAcw:true | Conference ends |

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| VC Internal | `APIConferenceCall` (AgentProvider) | Merges legs into conference |
| VC Internal | `ConferenceStatusChangedStrategy.CreateRequest()` | Reports `ConferenceId`, `ConferenceStatusType` |
| Studio Script Action | `HoldConference` | `System\AgentUI\HoldConference.xml` — holds all legs |
| Studio Script Action | `ResumeConference` | `System\AgentUI\ResumeConference.xml` |
| Studio Script Action | `TransferConference` | `System\AgentUI\TransferConference.xml` |
| `AgentSessionStart` field | `CanMultiPartyConference` | Permission gate — must be true for MCH |
| `AgentSessionStart` field | `EnabledForMCH` | Enables MCH capabilities for agent |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| `HoldConference.xml` | `System/AgentUI/` | Holds entire conference |
| `ResumeConference.xml` | `System/AgentUI/` | Resumes conference |
| `TransferConference.xml` | `System/AgentUI/` | Transfers conference |
| `ONASSIGNMENT` (system) | `System/AgentUI/` | Fires when Agent B joins |
| `ONUNASSIGNMENT` (system) | `System/AgentUI/` | Fires for each agent leaving |

---

## Open Questions / Gaps
- `[CONFIRMED]` Conference requires `CanMultiPartyConference=true` from `AgentSessionStart` and `EnabledForMCH=true`
- `[CONFIRMED]` When patron drops from a joined conference, both agent legs are currently also disconnected (see SCN-003)
- `[CONFIRMED]` Valid conference states for merge: legs must be in Active, Holding, or Joined states

---

## Related Scenarios
- SCN-003: Patron drops mid-conference (failure scenario)
- SCN-005: Warm transfer (conference setup is identical; differs in final action)
- SCN-001: Inbound call — happy path
