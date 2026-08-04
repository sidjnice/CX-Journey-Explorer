---
id: SCN-003
title: Patron Drops Mid-Conference — Conference Collapses, Both Agents Enter ACW
category: conference
status: partial
source: GM_ask_on_call_back_the_patron transcript 2026-07-23 · agent-contact-lifecycle/SKILL.md · Agent Events Confluence 17504254
last_verified: 2026-07-24
---

## Summary
Agent A is on a call with a patron. Agent A initiates a consultative call to Agent B, brings them into a 3-way conference, and then the patron drops. VC's current behaviour is to collapse the entire conference — both Agent A and Agent B are disconnected, and both enter ACW if their skill has ACW enabled. GM (OnStar) wants the Agent A ↔ Agent B connection to survive the patron's disconnection. This is not currently supported and may not be the confirmed requirement.

## Trigger
Patron leg SIP BYE received while a 3-way conference (`CallContactEvent` Status: **Joined**) is active.

---

## Products Involved

| Product / Component | Role |
|---|---|
| PSTN / Carrier | SIP BYE from patron leg |
| VCSvc — ACDProvider | MCH (Multi-party Call Handling) state machine manages conference legs |
| VCSvc — AgentProvider | `ConferenceStatusChangedStrategy` processes participant changes |
| VCSvc — ProviderESN or ProviderMRM | 3-way audio bridge |
| VCSvc — ProviderEntityStateManagement | Notifies EM of conference state change / contact ends |
| UIQ | Pushes disconnected + ACW state to both Agent Workspaces |
| Agent Workspace (Agent A) | Shows ACW screen after conference collapse |
| Agent Workspace (Agent B) | Shows ACW screen after conference collapse |

---

## Journey — State Machine

### Sub-Scenario A: Full Conference Active When Patron Drops

- Agent A and patron in active call
  - Agent A initiates consult to Agent B
    - `CallContactEvent` Agent B leg → Status: **Active**, CallType: **Consult**
    - `AgentState` Agent A → **OutboundConsult**
    - `AgentState` Agent B → **InboundConsult**
    - Agent A initiates conference (`APIConferenceCall`)
      - `ConferenceStatusChangedStrategy` fires
        - All three legs joined
          - `CallContactEvent` → Status: **Joined** for all legs
          - `MchAgentSettingsChangeEvent` — MCH capacity re-evaluated
            - **Patron leg drops**
              - SIP BYE received from carrier for patron leg
                - `RemoteCallDisconnectedStrategy` fires on patron contact
                  - `→ [GAP]` **Conference collapses — current VC behaviour**
                    - Agent A leg → **Disconnected**, FinalState: true
                    - Agent B leg → **Disconnected**, FinalState: true
                    - `AgentState` Agent A → **Unavailable**, IsAcw: true
                    - `AgentState` Agent B → **Unavailable**, IsAcw: true
                      - Both agents in ACW — cannot call back (same constraint as SCN-002)
                        - `→ [UNKNOWN]` GM wants Agent A ↔ Agent B leg to stay alive
                          - Not supported: VC conference teardown is tied to patron leg presence
                          - Would require MCH state machine change to selectively collapse

### Sub-Scenario B: Patron Drops Before Conference Fully Formed (Pre-Join)

- Agent A and patron in active call
  - Agent A puts patron on hold
    - `CallContactEvent` patron → Status: **Holding**
    - Agent A dials PSAP (emergency services) as third party
      - At this point: patron is on hold, not yet in a joined conference
        - **Patron drops while on hold (before Agent A brings them into conference)**
          - `→ [UNKNOWN]` Is this even a conference in VC's definition at this point?
            - Melena Fenn (3:41): "it may not even be a conference call"
            - VC conference state (`Joined`) not yet reached
            - Patron disconnect tears down the hold leg
              - Agent A continues talking with PSAP
                - `→ [UNKNOWN]` Does Agent A's PSAP leg survive?
                  - Depends on whether patron hold leg collapsing affects the outbound consult leg
                  - Needs confirmation from VC/R&D (Brendan Johnston)

---

## Events

### Emitted by VC → Agent Workspace (via UIQ)

| Event Type | Key Fields | When |
|---|---|---|
| `CallContactEvent` | Status: **Active**, CallType: **Consult**, ContactID | Agent B leg created |
| `AgentState` | CurrentState: **OutboundConsult** | Agent A during consult dial |
| `AgentState` | CurrentState: **InboundConsult** | Agent B receiving consult |
| `CallContactEvent` | Status: **Joined** | Conference formed (all 3 legs) |
| `MchAgentSettingsChangeEvent` | VoiceThreshold, TotalContactCount | MCH capacity check |
| `CallContactEvent` | Status: **Disconnected**, FinalState: true (patron) | Patron drops |
| `CallContactEvent` | Status: **Disconnected**, FinalState: true (Agent A) | Conference collapses |
| `CallContactEvent` | Status: **Disconnected**, FinalState: true (Agent B) | Conference collapses |
| `AgentState` | CurrentState: **Unavailable**, IsAcw: true (Agent A) | ACW begins |
| `AgentState` | CurrentState: **Unavailable**, IsAcw: true (Agent B) | ACW begins |

### Consumed by VC (External → VC)

| Source | Signal | Effect in VC |
|---|---|---|
| PSTN / Carrier | SIP BYE (patron leg) | `RemoteCallDisconnectedStrategy` on patron contact |
| Agent Workspace | `APIConferenceCall` | `ConferenceStatusChangedStrategy` fires |

---

## State Transitions

| Entity | From State | To State | Trigger |
|---|---|---|---|
| Contact (Agent A leg) | Active(4) | Conference(14) | Conference initiated |
| Contact (patron leg) | Active(4) | Conference(14) | Conference joined |
| Contact (Agent B leg) | Consult | Conference(14) | Brought into conference |
| Contact (patron) | Conference(14) | Disconnected | SIP BYE received |
| Contact (Agent A) | Conference(14) | Disconnected | Conference collapses |
| Contact (Agent B) | Conference(14) | Disconnected | Conference collapses |
| Agent A | OutboundConsult(6) → InboundContact(3) | Unavailable(2) IsAcw:true | Disconnect + ACW |
| Agent B | InboundConsult(5) → InboundContact(3) | Unavailable(2) IsAcw:true | Disconnect + ACW |

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| VC Internal | `APIConferenceCall` (AgentProvider) | Merges legs into conference |
| VC Internal | `ConferenceStatusChangedStrategy.CreateRequest()` | Reports conference state with `ConferenceId` |
| Studio Script Action | `HoldConference` | `System\AgentUI\HoldConference.xml` |
| Studio Script Action | `ResumeConference` | `System\AgentUI\ResumeConference.xml` |
| Studio Script Action | `TransferConference` | `System\AgentUI\TransferConference.xml` |
| VC Internal | `RemoteCallDisconnectedStrategy` | Patron disconnect processing |
| Agent SDK | `[UNKNOWN]` | No SDK event for conference collapse notification today |
| MCH | `MchAgentSettingsChangeEvent` | Capacity thresholds validated before conference |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| `HoldConference.xml` | `System/AgentUI/` | Holds entire conference |
| `ResumeConference.xml` | `System/AgentUI/` | Resumes from hold |
| `TransferConference.xml` | `System/AgentUI/` | Transfers conference |
| `ONUNASSIGNMENT` (system) | `System/AgentUI/` | Fires for each disconnecting agent leg |

---

## Open Questions / Gaps

- `[UNKNOWN]` Does VC support keeping Agent A ↔ Agent B connected when patron drops from a 3-way?
  - Current answer: No — conference collapses when patron drops
  - Melena Fenn (4:25): "disconnect it for both Agent A and Agent B because a conference is no longer in progress"
  - Requires MCH state machine change to implement

- `[UNKNOWN]` Is the requirement to keep the conference alive actually confirmed by GM?
  - Mike Winegar (5:00): "I'm not sure I've heard that one — multiple versions of the commitment"
  - Boris spoke to GM; recording not accessible
  - Owner: GM (Stephen McAlister) via Boris Grinshpun

- `[UNKNOWN]` Sub-Scenario B: When patron is on hold (not yet in Joined conference) and drops — does the Agent A outbound leg to PSAP survive?
  - Not confirmed; needs VC/R&D input (Brendan Johnston)

- `[GAP]` PSAP (emergency services) context: patron dropping while PSAP is being dialled is high-risk
  - This sub-scenario has no action item and needs separate escalation path

- `[CONFIRMED]` MCH requires `CanMultiPartyConference` permission set in `AgentSessionStart` event
  - Source: Agent Events Confluence 17504254, Agent row 1, `CanMultiPartyConference` field

---

## Related Scenarios
- SCN-001: Inbound call — happy path
- SCN-002: Patron drops unexpectedly (single-agent ACW scenario)
- SCN-005: Conference call — happy path
- SCN-004: Warm transfer (consult leg creation shared with conference)
