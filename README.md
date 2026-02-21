# home_asistant_llm_ios

**Natural-language home control: Siri → Home Assistant → OpenAI → your devices.**

Say _"Hey Siri, I'm going to take a nap"_ or _"Hey Siri, make it cozy"_ and the right home mode activates — no pre-coding every phrase. Homebridge stays untouched.

```
Siri → iOS Shortcut → Home Assistant (Ubuntu/Docker) → OpenAI GPT → HA Script → Devices
```

---

## Home Modes

| Mode | Script | Example phrases |
|------|--------|----------------|
| Nap | `script.nap_mode` | "I'll take a nap", "I need a quick rest" |
| Sleep | `script.sleep_mode` | "Good night", "I'm going to sleep" |
| Away | `script.away_mode` | "I'm leaving", "Nobody's home" |
| Movie | `script.movie_mode` | "Let's watch a movie", "Movie time" |
| Cozy | `script.cozy_mode` | "Make it cozy", "Set a chill vibe" |
| All Off | `script.all_off` | "Turn everything off" |
| Good Morning | `script.good_morning` | "Good morning", "Wake up the house" |

Adding a new mode = add a script to `homeassistant/scripts/modes.yaml` + one line in the system prompt. No new Shortcuts ever needed.

---

## Step 1 — Install Docker on the Ubuntu Server

SSH into the server and run:

```bash
# Install Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER

# Log out and back in so the group takes effect, then verify:
docker --version
```

Install the Docker Compose plugin:

```bash
sudo apt-get install -y docker-compose-plugin
docker compose version   # should print v2.x.x
```

---

## Step 2 — Deploy Home Assistant

```bash
# Create working directory
mkdir -p ~/homeassistant/ha_config
cd ~/homeassistant

# Download the compose file from this repo
curl -o docker-compose.yml \
  https://raw.githubusercontent.com/juanlu-a/home_asistant_llm_ios/main/docker/docker-compose.yml
```

Open it and set your timezone:

```bash
nano docker-compose.yml
# Change TZ=Europe/Madrid to your timezone, e.g. TZ=America/New_York
```

Start it:

```bash
docker compose up -d

# Watch the logs — first start takes ~60 seconds
docker compose logs -f homeassistant
```

When you see `Starting Home Assistant` → open from any device on the same Wi-Fi:

```
http://<ubuntu-server-ip>:8123
```

Complete the onboarding wizard: create an admin user, set your location, skip auto-discovered devices for now.

> **Find your server IP:** run `ip addr show` on the Ubuntu server and look for the `inet` address on your LAN interface (e.g. `192.168.1.100`).

---

## Step 3 — Copy HA Config Files

Clone this repo on your laptop and sync the config to the server:

```bash
git clone https://github.com/juanlu-a/home_asistant_llm_ios.git
cd home_asistant_llm_ios

rsync -av homeassistant/ user@<server-ip>:~/homeassistant/ha_config/
```

Then on the server, create `secrets.yaml` (this file is gitignored — never commit it):

```bash
cat > ~/homeassistant/ha_config/secrets.yaml << 'EOF'
home_latitude: "40.4168"      # your latitude
home_longitude: "-3.7038"     # your longitude
home_elevation: "667"         # metres above sea level
EOF
```

**Edit entity IDs to match your devices.**
Open `ha_config/scripts/modes.yaml` and replace the placeholder entity IDs:

- `light.living_room_main` → your actual light entity ID
- `light.living_room_ambient` → your ambient light
- `climate.home` → your thermostat
- `area_id: home` / `area_id: bedroom` → your actual area names

Find entity IDs in HA: **Settings → Devices & Services → Entities**.

Restart HA to pick up the new config:

```bash
docker compose restart homeassistant
```

Verify: **Settings → Scripts** — you should see nap_mode, sleep_mode, etc. Try running one manually from the UI.

---

## Step 4 — Configure OpenAI in Home Assistant

### 4.1 Add the OpenAI Conversation integration

1. In HA, go to **Settings → Devices & Services → Add Integration**
2. Search for **OpenAI Conversation** → click it
3. Enter your [OpenAI API key](https://platform.openai.com/api-keys)
4. Model: **`gpt-4o-mini`** (fast and cheap; ~$0.15/1M tokens — practically free for home use)
5. Click **Submit**

### 4.2 Paste the system prompt

1. Go to **Settings → Devices & Services → OpenAI Conversation → Configure**
2. Open `homeassistant/openai/system_prompt.txt` from this repo
3. Paste the entire contents into the **Instructions / System prompt** field
4. Save

The prompt tells the LLM it can only pick from your approved modes, refuses any security-related actions, and asks for clarification when the intent is ambiguous.

### 4.3 Restrict what the LLM can control (safety)

Still in the OpenAI integration settings:
- Set **"Control Home Assistant"** to **`No control`** — this means the LLM can only respond with text, and your YAML scripts do the actual work.

> If you want the LLM to call scripts directly instead, set it to "Assist" and add an allowed-scripts allowlist. But "No control" + explicit script calls in the prompt is simpler and safer for this setup.

### 4.4 Create a Voice Assistant

1. **Settings → Voice assistants → Add Assistant**
2. Name: `Home AI`
3. Conversation agent: **OpenAI Conversation**
4. Language: your language
5. Save

### 4.5 Test it from the HA UI

1. **Settings → Voice assistants → Home AI → Try**
2. Type: `I'll take a nap`
3. Check **Settings → Logbook** — you should see `script.nap_mode` executed

If this works, the LLM layer is correct. Now wire it to Siri.

---

## Step 5 — Generate a Long-Lived Access Token

The Siri Shortcut needs to authenticate to HA.

1. In HA, click your **profile icon** (bottom-left of the sidebar)
2. Scroll to **Long-Lived Access Tokens** → **Create Token**
3. Name: `iOS Shortcut`
4. Copy the token — **you only see it once**
5. Keep it somewhere safe (you'll paste it into the Shortcut in the next step)

---

## Step 6 — Create the Siri Shortcut on iPhone

Open the **Shortcuts** app on your iPhone and create a new Shortcut with these actions in order:

### Action 1 — Dictate Text
- Search: `Dictate Text`
- Language: Default
- Leave prompt empty
- Output variable: `DictatedText`

### Action 2 — Get Contents of URL
- Search: `Get Contents of URL`
- **URL:**
  ```
  http://<server-ip>:8123/api/conversation/process
  ```
- **Method:** `POST`
- **Headers** (tap "Add new header" for each):

  | Key | Value |
  |-----|-------|
  | `Authorization` | `Bearer <paste-your-token-here>` |
  | `Content-Type` | `application/json` |

- **Request body:** `JSON`
  - Add a key `text` → value: tap the field → choose **Variable** → pick `DictatedText`
  - Add a key `language` → value: `en` (or `es`, etc.)

- Output variable: `HAResponse`

### Action 3 — Get Value from Dictionary
- Dictionary: `HAResponse`
- Key: `response`
- Output: `ResponseDict`

### Action 4 — Get Value from Dictionary
- Dictionary: `ResponseDict`
- Key: `speech`
- Output: `SpeechDict`

### Action 5 — Get Value from Dictionary
- Dictionary: `SpeechDict`
- Key: `plain`
- Output: `PlainDict`

### Action 6 — Get Value from Dictionary
- Dictionary: `PlainDict`
- Key: `speech`
- Output: `ConfirmationText`

### Action 7 — Speak Text
- Text: `ConfirmationText`

> **Why so many dictionary steps?** The HA Conversation API nests the reply text inside `response → speech → plain → speech`. Each step unwraps one level.

### Set the Siri trigger

1. Tap the Shortcut name at the top → **Add to Siri**
2. Record the phrase: **`Assistant`**
3. Save

Now you can say:
> "Hey Siri, Assistant" → Siri listens → you say your phrase → Siri speaks the confirmation

---

## Step 7 — End-to-End Test

Try each of these:

| Say this | Expected result |
|----------|----------------|
| "I'll take a nap" | `script.nap_mode` runs |
| "I'm going to sleep" | `script.sleep_mode` runs |
| "I'm leaving" | `script.away_mode` runs |
| "Let's watch a movie" | `script.movie_mode` runs |
| "Make it cozy" | `script.cozy_mode` runs |
| "Unlock the front door" | Refused — no action |
| "Disable the alarm" | Refused — no action |

Check **HA → Logbook** after each test. You should also see the last command in the `input_text.last_voice_command` entity.

Full test plan with acceptance criteria: [`docs/test_plan.md`](docs/test_plan.md)

---

## Adding a New Mode

1. Add a script block to `homeassistant/scripts/modes.yaml` (follow existing pattern)
2. Reload in HA: **Developer Tools → YAML → Scripts**
3. Add the new mode name + description to the **Instructions** field in the OpenAI integration settings
4. Done — no new Shortcut needed

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Can't reach `:8123` | HA not running or wrong IP | `docker compose ps` on server; check IP with `ip addr` |
| 401 Unauthorized from Shortcut | Bad/expired token | Regenerate token in HA profile |
| Shortcut gets empty response | JSON path mismatch | Check raw `HAResponse` in Shortcut results |
| Wrong mode triggers | LLM misinterpretation | Rephrase; review system prompt |
| OpenAI integration not found | Old HA version | Update HA: `docker compose pull && docker compose up -d` |

---

## Safety Guarantees

- The LLM can only activate scripts from the approved mode list
- Locks, alarms, garage doors, and security devices are explicitly forbidden in the system prompt
- Ambiguous commands → LLM asks for clarification; no script runs
- If OpenAI is unreachable → error message returned; no script runs
- HA is LAN-only by default — not exposed to the internet

---

## Repo Structure

```
├── docker/
│   └── docker-compose.yml              # HA container for Ubuntu
├── homeassistant/
│   ├── configuration.yaml              # Main HA config
│   ├── scripts/modes.yaml              # All home mode scripts
│   ├── automations/voice_logging.yaml  # Logs voice commands for debugging
│   ├── helpers/input_helpers.yaml      # input_text state helpers
│   ├── helpers/input_helpers_bool.yaml # input_boolean mode trackers
│   └── openai/system_prompt.txt        # LLM system prompt
├── shortcuts/
│   └── siri_shortcut_setup.md          # Expanded Shortcut reference
└── docs/
    ├── setup_guide.md                  # Detailed setup walkthrough
    ├── architecture.md                 # System design + security model
    └── test_plan.md                    # AC1–AC11 acceptance criteria
```
