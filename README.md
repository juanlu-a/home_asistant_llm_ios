# home_asistant_llm_ios

**Natural-language voice control for your home via Siri → Home Assistant → OpenAI.**

Say things like "Hey Siri, I'm going to take a nap" or "Hey Siri, make it cozy" and have the right home mode activate — no pre-coding every phrase.

---

## What This Does

```
Siri → iOS Shortcut → Home Assistant (Ubuntu/Docker) → OpenAI GPT → HA Script → Devices
```

- **Siri** captures your voice via one universal Shortcut ("Hey Siri, Assistant").
- **Home Assistant** receives the phrase and routes it through an **OpenAI Conversation Agent**.
- The LLM maps your intent to one of a finite set of **home modes** (nap, sleep, away, movie, cozy, …).
- HA executes the corresponding script, which controls lights, climate, etc.
- A short confirmation is spoken back: *"Okay — nap mode activated."*
- **Homebridge stays installed and fully functional.**

---

## Repo Structure

```
├── docker/
│   └── docker-compose.yml          # HA container (Ubuntu server)
├── homeassistant/
│   ├── configuration.yaml          # Main HA config
│   ├── scripts/
│   │   └── modes.yaml              # Home mode scripts (nap, sleep, away, movie, cozy…)
│   ├── automations/
│   │   └── voice_logging.yaml      # Logs last voice command for debugging
│   ├── helpers/
│   │   ├── input_helpers.yaml      # input_text helpers (last_voice_command, last_selected_mode)
│   │   └── input_helpers_bool.yaml # input_boolean helpers (mode_sleep, mode_away)
│   └── openai/
│       └── system_prompt.txt       # LLM system prompt — paste into HA OpenAI integration
├── shortcuts/
│   └── siri_shortcut_setup.md      # Step-by-step iOS Shortcut instructions
└── docs/
    ├── setup_guide.md              # Full installation walkthrough
    ├── architecture.md             # System design + security model
    └── test_plan.md                # Acceptance criteria (AC1–AC11)
```

---

## Quick Start

1. **Read [`docs/setup_guide.md`](docs/setup_guide.md)** — full step-by-step from Docker install to working voice control.
2. Deploy HA with `docker/docker-compose.yml`.
3. Copy the YAML files from `homeassistant/` into your HA config directory.
4. Add the **OpenAI Conversation** integration in HA and paste the system prompt from `homeassistant/openai/system_prompt.txt`.
5. Create the Siri Shortcut following `shortcuts/siri_shortcut_setup.md`.
6. Run through the tests in `docs/test_plan.md`.

---

## Home Modes (initial set)

| Mode | Script | Example phrases |
|------|--------|----------------|
| Nap | `script.nap_mode` | "I'll take a nap", "I need a quick rest" |
| Sleep | `script.sleep_mode` | "Good night", "I'm going to sleep" |
| Away | `script.away_mode` | "I'm leaving", "Nobody's home" |
| Movie | `script.movie_mode` | "Let's watch a movie", "Movie time" |
| Cozy | `script.cozy_mode` | "Make it cozy", "Set a chill vibe" |
| All Off | `script.all_off` | "Turn everything off" |
| Good Morning | `script.good_morning` | "Good morning", "Wake up the house" |

Adding a new mode = create a new HA script + add one line to the system prompt. No new Shortcuts needed.

---

## Safety Guarantees
- The LLM can **only** call `script.turn_on` on the allowlisted mode scripts.
- Locks, alarms, garage doors, and security devices are **explicitly forbidden** in the system prompt.
- Ambiguous commands → LLM asks a clarifying question; no action taken.
- If OpenAI is unreachable → error message returned; no script executed.

---

## Requirements
- Ubuntu Server with Docker + Docker Compose
- OpenAI API key
- iPhone/iPad (iOS 16+) on the same LAN
- Homebridge already running (untouched by this setup)

---

## Docs
- [Setup Guide](docs/setup_guide.md)
- [Architecture](docs/architecture.md)
- [Test Plan](docs/test_plan.md)
- [Siri Shortcut Setup](shortcuts/siri_shortcut_setup.md)
