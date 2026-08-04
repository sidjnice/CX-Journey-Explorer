---
id: SCN-001
title: Inbound Call — Patron to Agent (Happy Path)
category: inbound-voice
status: supported
source: agent-contact-lifecycle/SKILL.md · contact-lifecycle/SKILL.md · Agent Events Confluence 17504254
last_verified: 2026-07-24
---

## Summary
A patron dials in via PSTN. The call is received by VCSvc, a Studio IVR script runs, the contact is queued to a skill, ACD selects an available agent, delivers the call, the agent accepts, both parties talk, and the call ends normally with the agent entering ACW.

## Trigger
Patron dials DNIS → PSTN delivers SIP INVITE to VC cluster.

---

## Products Involved

| Product / Component | Role |
|---|---|
| PSTN / Carrier | Delivers inbound SIP call to VC |
| VCSvc — ACDProvider | Receives contact, drives routing lifecycle |
| VCSvc — IVR Script Engine | Executes Studio script (REQAGENT, MENU, PLAY actions) |
| VCSvc — AgentProvider | Manages agent session; delivers contact to agent |
| VCSvc — ProviderESN or ProviderMRM | Establishes RTP audio bridge (mutually exclusive per cluster) |
| VCSvc — ProviderEntityStateManagement | Sends state changes to Entity Management (EM) via gRPC |
| Entity Management (EM) | Validates and confirms contact/agent state transitions |
| RRR (Routing Rules Engine) | Evaluates routing rules to select skill/priority |
| UIQ (UI Queue) | Pushes agent events to Agent Workspace via Kafka/MSK |
| Agent Workspace | Surfaces incoming call UI to agent |
| SuiteData / Kinesis | Async contact record written for reporting |

---

## Journey — State Machine

- Patron dials DNIS
  - VCSvc ACDProvider receives SIP INVITE
    - Contact created — ContactID assigned, BusNo scoped
      - Studio IVR script starts executing on processing thread
        - Script plays greeting (`PLAY` action)
          - `→ [BRANCH]` IVR menu input received (`MENU` action)
            - `→ [BRANCH A]` DTMF / speech input matches a branch
              - Script routes to appropriate skill path
            - `→ [BRANCH B]` No input / timeout
              - Script takes default branch (re-prompt or queue)
          - REQAGENT action fires → contact enters queue
            - `IncomingContactStrategy` sends contact to EM via gRPC (port 9880/9881)
            - Script offloaded to persistent store — processing thread released
            - RRR evaluates routing rules → skill + priority assigned
              - ACD FindMatch selects best available agent
                - `ContactDeliveryCompletedStrategy` fires
                  - Agent Workspace receives ring notification via UIQ push
                    - `→ [BRANCH]` Agent accepts within timeout
                      - `AcceptContactStrategy` executes
                        - Audio bridge activated (ESN or MRM)
                          - `ONASSIGNMENT` Studio action fires in agent UI script
                            - Agent and patron in active conversation
                              - `→ [BRANCH]` Patron or agent ends call
                                - Audio bridge torn down
                                  - `RemoteCallDisconnectedStrategy` fires
                                    - `ONUNASSIGNMENT` Studio action fires
                                      - `→ [BRANCH]` Skill has UseACW = true
                                        - Agent enters ACW state
                                          - Agent sets disposition
                                            - Agent exits ACW (manual or timeout)
                                              - Agent returns to Available
                                      - `→ [BRANCH]` Skill has UseACW = false
                                        - Agent returns directly to Available
                    - `→ [BRANCH]` Agent does not accept (timeout / refusal)
                      - Contact re-queued or script takes refusal branch
                      - Agent state → Refused

---

## Events

### Emitted by VC → Agent Workspace (via UIQ)

| Event Type | Key Fields | When |
|---|---|---|
| `CallContactEvent` | Status: **Incoming**, CallType: Regular, DNIS, ANI, SkillId | Contact arrives at VC |
| `AgentState` | CurrentState: **InboundContact** | Contact delivered to agent |
| `AgentLeg` | Status: **Dialing** | Ring begins at agent |
| `CallContactEvent` | Status: **Active**, ContactID, InteractionId | Agent accepts |
| `AgentLeg` | Status: **Active** | Agent leg live |
| `CallContactEvent` | Status: **Disconnected**, FinalState: true | Call ends |
| `AgentLeg` | Status: **Disconnected**, FinalState: true | Agent leg ends |
| `AgentState` | CurrentState: **Unavailable**, IsAcw: true, AcwTimeout | ACW begins |
| `AgentState` | CurrentState: **Available** | ACW complete |

### Consumed by VC (External → VC)

| Source | Signal | Effect in VC |
|---|---|---|
| PSTN / Carrier | SIP INVITE | Contact creation triggered |
| Entity Management (gRPC) | `AcdContactStateChange` callback | Unblocks script after state transition |
| Agent Workspace | Accept action (REST API) | `AcceptContactStrategy` fires |
| PSTN / Carrier | SIP BYE | `RemoteCallDisconnectedStrategy` fires |

---

## State Transitions

| Entity | From State | To State | Trigger |
|---|---|---|---|
| Contact | — | Incoming (arriving) | SIP INVITE received |
| Contact | Incoming | Active | Script running / in queue |
| Agent | Available(1) | InboundContact(3) | Contact delivered |
| Contact | Active(4) | Active(4) | Agent accepts (sustained) |
| Contact | Active(4) | Disconnected | Call ends |
| Agent | InboundContact(3) | Unavailable(2) IsAcw:true | Call ends, UseACW=true |
| Agent | Unavailable(2) | Available(1) | ACW completed |

*State values: Contact from `ACDContactState.ContactState`; Agent from `ScriptHelper.AgentState`*

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| Studio Script Action | `REQAGENT` | Queues contact, triggers routing |
| Studio Script Action | `MENU` | Captures DTMF/speech for IVR branching |
| Studio Script Action | `PLAY` | Renders audio prompts |
| VC Internal | `IncomingContactStrategy.HandleAsync()` | Sends contact to EM |
| VC Internal | `ContactDeliveryCompletedStrategy` | Signals delivery completion |
| VC Internal | `AcceptContactStrategy` | Processes agent accept |
| VC Internal | `RemoteCallDisconnectedStrategy` | Handles disconnect + ACW config |
| VC Internal | `ChangeAgentStateStrategy` | Maps ACW → AGENT_STATE_UNAVAILABLE, routable=false |
| gRPC (VC → EM) | `SelfService.SendAsync()` | State change request |
| gRPC (EM → VC) | `AcdContactStateChange` | EM confirms state transition |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| Customer IVR script | `Code/IvrScripts/<tenant>/` | MENU, PLAY, REQAGENT — business logic |
| `OnAssignment` (system) | `System/AgentUI/` | Fires on contact delivery to agent |
| `Disposition Call` (system) | `System/AgentUI/` | Fires on ACW disposition |
| `ONUNASSIGNMENT` (system) | `System/AgentUI/` | Fires on contact disconnect |

---

## Open Questions / Gaps
- `[CONFIRMED]` ACW is configured at skill level (`UseACW`, `ACWTimeoutSeconds`) set at delivery time
- `[CONFIRMED]` Agent cannot initiate new voice contact while ACW contact is unresolved (two-voice-contact constraint)

---

## Related Scenarios
- SCN-002: Inbound call — patron to agent, patron drops unexpectedly (GM Scenario 1)
- SCN-003: Blind transfer
- SCN-004: Warm (consultative) transfer
- SCN-005: Conference call (3-way)
- SCN-007: Agent session login / logout
