---
id: SCN-002
title: Patron Drops Unexpectedly — Agent Stuck in ACW, Callback Blocked
category: disconnect
status: partial
source: GM_ask_on_call_back_the_patron transcript 2026-07-23 · agent-contact-lifecycle/SKILL.md · Agent Events Confluence 17504254
last_verified: 2026-07-24
---

## Summary
A patron call is unexpectedly lost (signal failure or accidental hang-up) while the agent is mid-call. VC treats the disconnect identically to a normal hang-up. If the skill has ACW enabled, the agent is forced into ACW state. While in ACW, the agent cannot initiate a new outbound call to the patron because VC enforces a one-active-voice-contact constraint. GM (OnStar) has committed to a solution. The solution is not yet fully defined.

## Trigger
Patron leg disconnects unexpectedly — SIP BYE received from carrier while agent and patron are in active call.

---

## Products Involved

| Product / Component | Role |
|---|---|
| PSTN / Carrier | SIP BYE delivery (may or may not include disconnect reason code) |
| VCSvc — ACDProvider | Processes disconnect; applies ACW config from skill |
| VCSvc — ProviderESN or ProviderMRM | Audio bridge torn down |
| VCSvc — AgentProvider | Transitions agent to ACW state |
| VCSvc — ProviderEntityStateManagement | Notifies EM of contact end / agent state change |
| UIQ | Pushes ACW state to Agent Workspace |
| Agent Workspace | Shows ACW screen; blocks outbound dial actions |
| Media Server (ESN/MRM) | May carry disconnect reason — patron-drop vs agent-drop |

---

## Journey — State Machine

- Agent and patron in active call
  - `CallContactEvent` Status: **Active** sustained
    - Patron leg drops unexpectedly
      - SIP BYE received by VC from carrier
        - `RemoteCallDisconnectedStrategy` fires
          - `ONUNASSIGNMENT` Studio action fires
            - VC checks skill config: `UseACW = true`
              - Agent forced into ACW
                - `AgentState` CurrentState: **Unavailable**, IsAcw: true
                  - Agent Workspace shows ACW screen
                    - `→ [GAP]` Agent wants to call patron back immediately
                      - Agent attempts to dial patron from call history
                        - `→ [GAP]` **BLOCKED: Cannot initiate outbound voice contact while ACW is unresolved**
                          - VC enforces: one active voice contact at a time
                          - Agent must first complete or dismiss ACW
                            - Agent manually exits ACW (completes disposition)
                              - `AgentState` → **Available**
                                - Agent navigates to call history
                                  - Agent initiates outbound call to patron ANI
                                    - `→ [UNKNOWN]` Does dialing the ANI reach the patron?
                                      - For GM/OnStar: ANI received is a proprietary number — may NOT be dialable back
                                    - `→ [BRANCH]` ANI is dialable — new outbound contact created
                                      - Normal outbound call flow (SCN-006)
                                    - `→ [BRANCH]` ANI is not dialable (OnStar proprietary)
                                      - Callback fails silently or errors
                                      - **Commitment gap — no platform path to patron**

---

## Events

### Emitted by VC → Agent Workspace (via UIQ)

| Event Type | Key Fields | When |
|---|---|---|
| `CallContactEvent` | Status: **Disconnected**, FinalState: true, DisconnectCode | Patron drops |
| `AgentLeg` | Status: **Disconnected**, FinalState: true | Agent leg ends |
| `AgentState` | CurrentState: **Unavailable**, IsAcw: true, AcwTimeout | ACW begins |
| `AgentState` | CurrentState: **Available** | ACW manually completed |

### Consumed by VC (External → VC)

| Source | Signal | Effect in VC |
|---|---|---|
| PSTN / Carrier | SIP BYE (+ optional reason code) | `RemoteCallDisconnectedStrategy` fires |
| Media Server | Disconnect reason signal (if exposed) | `[UNKNOWN]` — currently same event path as normal hang-up |

---

## State Transitions

| Entity | From State | To State | Trigger |
|---|---|---|---|
| Contact | Active(4) | Disconnected | SIP BYE from carrier |
| Agent | InboundContact(3) | Unavailable(2) IsAcw:true | Disconnect + UseACW=true on skill |
| Agent | Unavailable(2) | Available(1) | Agent manually exits ACW |
| Contact (new) | — | Outbound | Agent initiates callback (if ACW resolved) |

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| VC Internal | `RemoteCallDisconnectedStrategy` | Processes disconnect; sets `EndReason`, `CauseCode`, `RefusalReason` |
| VC Internal | `ChangeAgentStateStrategy` | Maps ACW state → `AGENT_STATE_UNAVAILABLE`, routable=false |
| Studio Script Action | `ONUNASSIGNMENT` | Fires on disconnect, runs cleanup logic |
| Agent SDK | `[UNKNOWN]` | No SDK event for patron-drop vs agent-drop currently exposed |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| `ONUNASSIGNMENT` (system) | `System/AgentUI/` | Fires on disconnect |
| `Disposition Call` (system) | `System/AgentUI/` | Fires for ACW disposition |

---

## Open Questions / Gaps

- `[UNKNOWN]` Does VC receive a distinguishable signal from the media server for patron-drop vs agent-initiated hang-up?
  - Mike Winegar (19:54): VC team says currently the same event path
  - Braden Call (9:41): Would need a more granular event from media server
  - Owner: Don (media server team) to confirm

- `[UNKNOWN]` For GM/OnStar specifically: does dialing back the incoming ANI reach the patron?
  - Melena Fenn (14:48): OnStar is proprietary; ANI may not be a dialable number
  - Owner: GM (Stephen McAlister) to confirm

- `[GAP]` Can ACW be auto-dismissed / auto-populated when patron-drop is detected, enabling one-click callback?
  - Braden Call (8:36): Proposed — auto-populate ACW then present callback button in one action
  - Blocked by: requires triggering event to know it was a drop not a deliberate hang-up

- `[GAP]` Agent abuse risk: if a simple callback button is always present, agents could end calls early to meet AHT/SLS targets and use callback button
  - Sid (17:24): system should give supervisors tools to detect and control this pattern

- `[CONFIRMED]` Agent cannot initiate new outbound voice contact while ACW is in progress
  - Source: Melena Fenn (8:24); VC contact state constraint, not UX

---

## Proposed Solutions (from transcript, not committed)

| Option | Description | Blocker |
|---|---|---|
| A | Disable ACW on those skills | GM wants ACW maintained |
| B | Detect patron-drop → skip ACW for that event | Requires granular media server signal (not available today) |
| C | Always show callback button after any disconnect, agent decides | Needs auto-dismiss ACW in same action; agent abuse risk |
| D | Phase 1: technical drops only; Phase 2: any disconnect | Phase 1 requires media server signal |

---

## Related Scenarios
- SCN-001: Inbound call — happy path (base scenario)
- SCN-003: Patron drops mid-conference (GM Scenario 2)
- SCN-006: Agent-initiated outbound call
- SCN-008: ACW — after call work lifecycle
