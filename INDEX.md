# CXone VC Scenario Library — Index

> Purpose: Single reference for all supported, partial, and unsupported CXone VC scenarios.
> Audience: New joiners, PMs, engineers, AI agents investigating customer asks.
> Source authority: vc-claude repo skills, Agent Events Confluence (17504254), transcript analysis.

---

## How to Use This Library

Each scenario file is structured identically:
- **Products involved** — which platform components participate
- **Journey** — state machine as nested bullets; gaps and unknowns marked explicitly
- **Events** — exact event type names (verified from Confluence 17504254), key fields, direction
- **State transitions** — entity / from / to / trigger; numeric values from source enums
- **APIs & SDK** — Studio actions, VC internal strategies, REST, Agent SDK
- **IVR scripts** — which system scripts fire
- **Open questions** — tagged `[CONFIRMED]`, `[UNKNOWN]`, or `[GAP]`

---

## Scenario Catalogue

| ID | Title | Category | Status |
|---|---|---|---|
| [SCN-001](SCN-001-inbound-call-patron-to-agent.md) | Inbound Call — Patron to Agent (Happy Path) | inbound-voice | ✅ Supported |
| [SCN-002](SCN-002-patron-drops-unexpectedly-acw-blocked.md) | Patron Drops Unexpectedly — Agent Stuck in ACW | disconnect | ⚠️ Partial (GM open) |
| [SCN-003](SCN-003-patron-drops-mid-conference.md) | Patron Drops Mid-Conference — Both Agents to ACW | conference | ⚠️ Partial (GM open) |
| [SCN-004](SCN-004-blind-transfer.md) | Blind Transfer (Cold Transfer) | transfer | ✅ Supported |
| [SCN-005](SCN-005-warm-transfer.md) | Warm Transfer (Consultative Transfer) | transfer | ✅ Supported |
| [SCN-006](SCN-006-conference-call.md) | Conference Call — 3-Way (Happy Path) | conference | ✅ Supported |
| [SCN-007](SCN-007-agent-session-lifecycle.md) | Agent Session — Login, State Changes, Logout | session | ✅ Supported |
| [SCN-008](SCN-008-acw-after-call-work.md) | ACW — After Call Work Lifecycle | acw | ✅ Supported |

---

## Scenarios To Be Added (Backlog)

| ID | Title | Category | Notes |
|---|---|---|---|
| SCN-009 | Agent-Initiated Outbound Call | outbound-voice | `NaturalCalling` / manual outbound |
| SCN-010 | Callback (PromiseKeeper) | inbound-voice | `PromiseKeeper` / `PromiseKeeperStatus` events |
| SCN-011 | Supervisor Monitor / Barge / Takeover | session | `SupervisorMonitor`, `SupervisorTakeOver` events |
| SCN-012 | Inbound Digital Contact (Chat/SMS) | digital | `ChatContactEvent`, `TextContactEvent` |
| SCN-013 | Voicemail Delivery | inbound-voice | `VoiceMailContactEvent` |
| SCN-014 | Post-Call Survey | inbound-voice | Survey script as child contact |
| SCN-015 | Agent Leg — Softphone / WebRTC | session | `CUSTOM_WebRTC` event |
| SCN-016 | Contact Routing via RRR | inbound-voice | Routing Rules Engine evaluation flow |
| SCN-017 | EM State Sync — VoltronStateMode patterns | platform | gRPC callback patterns; circuit breaker |
| SCN-018 | MCH (Multi-party Call Handling) — capacity limits | conference | `MchAgentSettingsChangeEvent`; threshold breach |
| SCN-019 | Mute / Unmute during call | inbound-voice | `Mute` event, `AgentMuted` field |
| SCN-020 | Screen Pop (PopURL / CustomScreenPop) | inbound-voice | `PopURL`, `CustomScreenPop` events |

---

## Event Reference Quick-Lookup

| Event Type | Category | Key Status Values |
|---|---|---|
| `AgentSessionStart` | Agent | — (one-time on login) |
| `AgentSessionEnd` | Agent | Success: true/false |
| `RemoteAgentSessionEnd` | Agent | RemoteLogoff, SessionTimeout |
| `AgentState` | Agent | Unavailable, Available, InboundContact, OutboundContact, InboundConsult, OutboundConsult, Dialer, LoggedIn |
| `AgentLeg` | Agent | Dialing, Active, Disconnected |
| `CallContactEvent` | Call | Incoming, Ringing, Active, Holding, Joined, Disconnected, CallBackDisconnected, Preview, Masking |
| `CallContactEvent` CallType | Call | Regular, AgentLeg, ReskillProxy, Consult, PersonalQueue, TakeOver, NaturalCalling |
| `MchAgentSettingsChangeEvent` | Agent | — (threshold values) |
| `UpdateSkills` | Agent | — |
| `UpdateUnavailableCodes` | Agent | — |
| `UpdatePermissions` | Agent | — |
| `UpdateTeams` | Agent | — |
| `Mute` | Agent | AgentMuted: true/false |
| `PopURL` | Call | URL, TabTitle, PopDestination |
| `Indicator` | Call | ActionType: RunExe, OpenURL, SpawnScript, SignalScript, ShowCustomForm |
| `PromiseKeeper` | Callback | TargetType: Agent/Skill |
| `PromiseKeeperStatus` | Callback | Dialing, Refused, Delivered, Rescheduled, Cancelled |
| `SupervisorContact` | SupervisorCall | (same statuses as CallContactEvent) |
| `SupervisorMonitor` | SupervisorCall | MonitorType: Monitor, StopMonitor |
| `SupervisorTakeOver` | SupervisorCall | ContactTakenOverByOther, Failed... |
| `ChatContactEvent` | Chat | Waiting, Incoming, Inviting, Active, Interrupted, Disconnected, Holding |
| `VoiceMailContactEvent` | Voicemail | Incoming, Active, Discarded, Holding |

*Source: Agent Events List, Confluence page 17504254*

---

## Status Legend
- ✅ **Supported** — fully implemented, events and states confirmed
- ⚠️ **Partial** — implemented but has known gaps or open customer questions
- ❌ **Unsupported** — not currently supported by the platform
- ❓ **Unknown** — requires investigation before status can be confirmed
