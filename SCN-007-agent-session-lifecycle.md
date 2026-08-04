---
id: SCN-007
title: Agent Session — Login, State Changes, Logout
category: session
status: supported
source: CLAUDE.md (AgentProvider) · Agent Events Confluence 17504254 · agent-contact-lifecycle/SKILL.md
last_verified: 2026-07-24
---

## Summary
An agent logs into Agent Workspace, which starts a VC session. The agent transitions between Available, Unavailable (with out-reason codes), and contact-handling states during the shift. The session ends on logout or timeout. The `AgentSessionStart` event carries permissions and configuration that gate capabilities like MCH, recording, and supervisor actions for the entire session.

## Trigger
Agent submits login credentials and selects a station in Agent Workspace → REST API calls `StartSession` in VC.

---

## Products Involved

| Product / Component | Role |
|---|---|
| Agent Workspace | Login UI, state controls, station selection |
| VCSvc — AgentProvider | Session management: `AgentProvider.cs`, `StartSession`, state transitions |
| VCSvc — ProviderEntityStateManagement | Syncs agent session to Entity Management |
| UIQ | Pushes all `AgentState` events from VC → Agent Workspace |
| WFM (Workforce Management) | Receives adherence ticks; monitors state durations |
| SQL Server — RT_Agent_Log | Active session record (`VC_AgentStartSync`) |
| SQL Server — Agent_Log_Header | Historical session record (`VC_AgentStartAsync`) |

---

## Journey — State Machine

- Agent opens Agent Workspace and submits login
  - REST API → `StartSession` in `AgentData.cs`
    - SessionId generated (= ContactID of session script)
      - `AgentSessionStart` event emitted — carries full agent configuration
        - Agent in **LoggedIn** state (not yet accepting contacts)
          - Agent sets state to Available
            - `AgentState` → CurrentState: **Available**
              - Agent can now receive contacts
                - `→ [BRANCH A]` Contact delivered — see SCN-001
                - `→ [BRANCH B]` Agent goes Unavailable (break, lunch, etc.)
                  - `AgentState` → CurrentState: **Unavailable**, CurrentOutReason: <code>
                    - `UpdateUnavailableCodes` sent to Agent Workspace (list of valid out-reason codes)
                      - Agent returns to Available when break ends
                        - `AgentState` → CurrentState: **Available**
                - `→ [BRANCH C]` Supervisor remotely changes agent state
                  - `TakeOver` event or `RemoteAgentSessionEnd` with Message: RemoteLogoff
                - `→ [BRANCH D]` Agent logs out voluntarily
                  - Agent clicks logout in Agent Workspace
                    - REST API → VC ends session
                      - `AgentSessionEnd` → Success: true
                        - RT_Agent_Log cleared; Agent_Log_Header completed
                - `→ [BRANCH E]` Session timeout (inactivity)
                  - `RemoteAgentSessionEnd` → Message: **SessionTimeout**
                    - Session cleaned up; agent must log back in

---

## Events

### Emitted by VC → Agent Workspace (via UIQ)

| Event Type | Key Fields | When |
|---|---|---|
| `AgentSessionStart` | AgentId, SessionId, StationId, StationPhoneNumber, CanMultiPartyConference, EnabledForMCH, CanRecord, SupervisorPermissionLevel, MaxConcurrentChats, AgentUUId, EntityMode | Login complete |
| `AgentState` | CurrentState: **LoggedIn** | Session started |
| `AgentState` | CurrentState: **Available** | Agent sets ready |
| `AgentState` | CurrentState: **Unavailable**, CurrentOutReason | Agent takes break |
| `UpdateUnavailableCodes` | (list of out-reason codes) | Sent to AW for agent to pick from |
| `UpdateSkills` | (skill list) | Agent's assigned skills refreshed |
| `UpdatePermissions` | (permission flags) | Permissions refresh |
| `UpdateTeams` | (team assignment) | Team membership refresh |
| `AgentSessionEnd` | Success: true/false, Message | Voluntary logout |
| `RemoteAgentSessionEnd` | Message: **RemoteLogoff** or **SessionTimeout** | Remote/timeout logout |
| `SessionStartFailed` | Message | Login failure |

### Consumed by VC (External → VC)

| Source | Signal | Effect in VC |
|---|---|---|
| Agent Workspace | `StartSession` REST call | `AgentData.cs.StartSession()` creates session |
| WebAdmin | `ForceReset` / `RemoveState` action | `ResetState` custom event sent |
| WebAdmin | Remote logoff action | `RemoteAgentSessionEnd` with RemoteLogoff |

---

## State Transitions

| Entity | From State | To State | Trigger |
|---|---|---|---|
| Agent | — | LoggedIn(8) | `AgentSessionStart` |
| Agent | LoggedIn(8) | Available(1) | Agent sets ready |
| Agent | Available(1) | Unavailable(2) | Agent break/lunch |
| Agent | Unavailable(2) | Available(1) | Break ends |
| Agent | Any | Dialer(7) | Dialer campaign assigned |
| Agent | LoggedIn | — (ended) | Logout / timeout |

---

## APIs & SDK

| Layer | API / Action / Method | Purpose |
|---|---|---|
| REST API | `StartSession` | Agent login; generates SessionId |
| VC Internal | `AgentProvider.cs` | Session lifecycle management |
| VC Internal | `AgentData.cs.StartSession()` | Creates `RT_Agent_Log` entry |
| VC Internal | `ChangeAgentStateStrategy` | Maps state changes → EM |
| VC Internal | `ResetState` custom event | Admin force-reset of agent |

---

## IVR Scripts Involved

| Script | Location | Role |
|---|---|---|
| `Logoff Agent.xml` | `System/AgentUI/` | Triggered on supervisor-initiated logoff |
| `PostCustomData.xml` | `System/AgentUI/` | `PostData` event to agent |
| `AgentMessage.xml` | `System/Notifications/` | `UpdateMessages` event to agent |

---

## Key Permission Gates (from `AgentSessionStart`)

| Field | Purpose |
|---|---|
| `CanMultiPartyConference` | Gates MCH / conference capability |
| `EnabledForMCH` | Enables multi-party handling |
| `CanRecord` | Gates call recording |
| `SupervisorPermissionLevel` | Gates supervisor actions (monitor, barge, takeover) |
| `MaxConcurrentChats` | Gates chat channel capacity |
| `ScoreRecordingsPermission` | Gates recording scoring |
| `HideAgentStatePermission` | Controls state visibility to supervisor |

---

## Open Questions / Gaps
- `[CONFIRMED]` SessionId is generated as the ContactID of the session script (`AgentData.cs.StartSession()`)
- `[CONFIRMED]` `RemoteAgentSessionEnd.Message` has exactly two values: `RemoteLogoff` and `SessionTimeout`

---

## Related Scenarios
- SCN-001: Inbound call (agent must be in Available state)
- SCN-006: Conference (requires `CanMultiPartyConference=true` from this event)
- SCN-008: ACW lifecycle
