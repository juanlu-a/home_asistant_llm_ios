# Architecture — Home Assistant + Siri + OpenAI

## High-Level Flow

```
User (iPhone)
    │
    │  "Hey Siri, Assistant"
    ▼
Siri Shortcut
    │  POST /api/conversation/process
    │  { "text": "<dictated phrase>" }
    ▼
Home Assistant  (Ubuntu Server :8123)
    │
    ├── Assist Pipeline
    │       │
    │       └── Conversation Agent: OpenAI (gpt-4o-mini)
    │               │  System prompt: mode catalog + rules
    │               │  Tool: script.turn_on (allowlisted scripts only)
    │               ▼
    │          Selects one mode script
    │
    ├── Script Executor
    │       │
    │       └── script.<mode>_mode
    │               ├── light.turn_on / turn_off
    │               ├── climate.set_temperature
    │               └── input_text/boolean updates
    │
    └── Returns confirmation text
            │
            ▼
        Siri Shortcut
            │  Speaks response
            ▼
        User hears: "Okay — nap mode activated."
```

---

## Component Responsibilities

| Component | Role | Notes |
|-----------|------|-------|
| **Siri Shortcut** | Voice entry point, HTTP client | One universal shortcut; no per-phrase logic |
| **Home Assistant** | Orchestrator, script runner, API server | Docker on Ubuntu; port 8123 |
| **OpenAI GPT** | Intent interpreter | Constrained to allowlisted scripts via system prompt + tool restrictions |
| **HA Scripts** | Mode "macros" | Stable entity IDs; LLM references these by name |
| **Homebridge** | Existing device bridge | Unchanged; continues serving iOS Home app |

---

## Security Model

```
Internet
    │
    │  HTTPS (OpenAI API calls FROM HA — outbound only)
    ▼
OpenAI API

LAN
    │
    ├── iPhone  ──── HTTP POST ────► HA :8123  (Long-Lived Token auth)
    │
    └── HA  ──── Homebridge (separate process, separate port)
```

- HA is **LAN-only** by default. No inbound internet exposure.
- All OpenAI calls are **outbound** from HA to `api.openai.com`.
- The Siri Shortcut authenticates to HA with a **Long-Lived Access Token** (bearer token in HTTP header).
- Token is stored in the iOS Shortcut (not in any file in this repo).
- The LLM can **only call** `script.turn_on` on an allowlisted set of script entity IDs.

---

## Port Map

| Service | Port | Protocol |
|---------|------|----------|
| Home Assistant | 8123 | HTTP (LAN) |
| Homebridge UI | 8581 | HTTP (LAN) |
| Homebridge HAP | 51826 | HAP/TCP |

No port conflicts between HA and Homebridge.

---

## Data Flow — Observability Hooks

```
Voice command received
    │
    ├── automation: log_voice_command
    │       └── input_text.last_voice_command ← raw phrase
    │
    └── (per script execution)
            └── input_text.last_selected_mode ← mode name
```

Both `input_text` helpers are visible on the HA dashboard and in the Logbook.

---

## Failure Modes

| Failure | Behavior |
|---------|----------|
| OpenAI unreachable | HA returns "I couldn't reach the assistant right now." No script runs. |
| Ambiguous intent | LLM asks clarifying question. No script runs. |
| Forbidden action (e.g. "unlock door") | LLM refuses per system prompt rules. No action taken. |
| HA unreachable from iPhone | Shortcut shows HTTP error. No action taken. |
| Bad auth token | HA returns 401. Shortcut shows error. |
