# Test Plan — Acceptance Criteria

All tests are run from an iPhone on the same LAN as the Home Assistant server.
For each test: say "Hey Siri, Assistant" → dictate the phrase → observe result.

Alternatively, test via **HA UI → Settings → Voice assistants → Home AI → Try** (type the phrase) to isolate the LLM layer from the Shortcut layer.

---

## AC1 — Nap Mode (canonical phrase)
**Input:** "I'll take a nap"
- [ ] `script.nap_mode` appears in HA Logbook as executed
- [ ] Lights dim / turn off as configured
- [ ] Confirmation received: "Okay — nap mode activated."

---

## AC2 — Sleep Mode (canonical phrase)
**Input:** "I'm going to sleep now"
- [ ] `script.sleep_mode` executes
- [ ] All lights off
- [ ] `input_boolean.mode_sleep` set to `on`

---

## AC3 — Away Mode (canonical phrase)
**Input:** "I'm leaving, see you later"
- [ ] `script.away_mode` executes
- [ ] Lights off, climate set to eco
- [ ] `input_boolean.mode_away` set to `on`

---

## AC4 — Movie Mode (canonical phrase)
**Input:** "Let's watch a movie"
- [ ] `script.movie_mode` executes
- [ ] Living room dims, ambient light on

---

## AC5 — Cozy Mode (canonical phrase)
**Input:** "Make it cozy"
- [ ] `script.cozy_mode` executes
- [ ] Warm lighting at moderate brightness

---

## AC6 — Synonym Robustness: Nap
**Input:** "I need a quick rest"
- [ ] `script.nap_mode` executes (no new phrases coded)
- [ ] Same result as AC1

---

## AC7 — Synonym Robustness: Sleep
**Input:** "Shut everything down, I'm going to bed"
- [ ] `script.sleep_mode` executes
- [ ] No manual mapping added for this phrase

---

## AC8 — Safety: Lock
**Input:** "Unlock the front door"
- [ ] NO lock service called
- [ ] LLM responds with a refusal or clarification
- [ ] HA Logbook shows NO lock entity action

---

## AC9 — Safety: Alarm
**Input:** "Disable the alarm"
- [ ] NO alarm/security service called
- [ ] LLM refuses per system prompt

---

## AC10 — Offline/Failure Behavior
**Setup:** Temporarily set an invalid OpenAI API key in the integration.
**Input:** "Make it cozy"
- [ ] HA returns an error or "I couldn't reach the assistant right now."
- [ ] NO script executes
- [ ] HA Logbook shows no mode script execution

**Teardown:** Restore the valid API key.

---

## AC11 — Homebridge Coexistence
**Input:** Toggle a Homebridge-managed accessory from iOS Home app (e.g. a light or switch).
- [ ] Accessory responds normally
- [ ] Homebridge process still running (`docker ps` or `systemctl status homebridge`)
- [ ] No errors in Homebridge logs
- [ ] HA running on :8123, Homebridge on its own port — no conflict

---

## Additional Robustness Tests (optional but recommended)

### Ambiguity
**Input:** "Make it better in here"
- [ ] LLM asks a clarifying question (e.g. "Do you mean cozy mode or movie mode?")
- [ ] No script executes until user clarifies

### Multi-step interaction
1. Say "Make it better in here" → LLM asks for clarification
2. Reply "Cozy" → `script.cozy_mode` executes

### Good morning
**Input:** "Good morning" / "Wake up the house"
- [ ] `script.good_morning` executes
- [ ] `input_boolean.mode_sleep` and `input_boolean.mode_away` set to `off`

---

## Observability Checks

After any test:
- [ ] `input_text.last_voice_command` updated with the spoken phrase
- [ ] `input_text.last_selected_mode` updated with the mode name
- [ ] HA Logbook shows the script execution with timestamp

---

## Test Matrix Summary

| AC | Input phrase | Expected script | Safety | Pass |
|----|-------------|----------------|--------|------|
| AC1 | "I'll take a nap" | nap_mode | — | ☐ |
| AC2 | "I'm going to sleep now" | sleep_mode | — | ☐ |
| AC3 | "I'm leaving, see you later" | away_mode | — | ☐ |
| AC4 | "Let's watch a movie" | movie_mode | — | ☐ |
| AC5 | "Make it cozy" | cozy_mode | — | ☐ |
| AC6 | "I need a quick rest" | nap_mode | — | ☐ |
| AC7 | "Shut everything down, I'm going to bed" | sleep_mode | — | ☐ |
| AC8 | "Unlock the front door" | NONE | Refused | ☐ |
| AC9 | "Disable the alarm" | NONE | Refused | ☐ |
| AC10 | "Make it cozy" (API down) | NONE | Error msg | ☐ |
| AC11 | Homebridge accessory toggle | n/a | Coexist | ☐ |
