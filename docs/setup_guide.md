# Setup Guide — Home Assistant + OpenAI + Siri on Ubuntu Server

## Prerequisites
- Ubuntu Server 22.04+ with Docker and Docker Compose installed
- SSH access to the server
- iPhone/iPad on the same LAN
- OpenAI API key ([platform.openai.com](https://platform.openai.com))
- Homebridge already running (stays untouched)

---

## Phase 1 — Install Home Assistant via Docker

### 1.1 Install Docker (if not already installed)
```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# log out and back in for group change to take effect
```

### 1.2 Install Docker Compose plugin
```bash
sudo apt-get install -y docker-compose-plugin
docker compose version   # verify
```

### 1.3 Create the HA working directory
```bash
mkdir -p ~/homeassistant/ha_config
cd ~/homeassistant
```

### 1.4 Copy docker-compose.yml
Copy the file from `docker/docker-compose.yml` in this repo to `~/homeassistant/docker-compose.yml`.

Edit the `TZ` environment variable to your timezone (e.g. `Europe/Madrid`).

### 1.5 Start Home Assistant
```bash
docker compose up -d
docker compose logs -f homeassistant   # watch startup (takes ~60s first run)
```

### 1.6 Complete onboarding
1. From any device on the LAN, open: `http://<server-ip>:8123`
2. Follow the onboarding wizard:
   - Create admin user
   - Set home location
   - Skip auto-discovered devices for now

**Checkpoint:** HA dashboard loads. Homebridge still running (different port).

---

## Phase 2 — Add at Least One Device to HA

Goal: validate the end-to-end pipeline before wiring up the LLM.

### Options (pick whichever you have):
- **HACS + your existing devices** — install HACS and add your device integration
- **Demo mode** — HA ships with demo entities; enable via `configuration.yaml`:
  ```yaml
  demo:
  ```
  This gives you fake lights/sensors to test scripts against.
- **HomeKit Controller** — if any devices are directly paired to HomeKit (not via Homebridge), HA can discover them.

**Checkpoint:** At least one light entity appears in HA and can be toggled from the UI.

---

## Phase 3 — Deploy HA Configuration Files

### 3.1 Copy config files from this repo
```bash
# From the repo root on your laptop, rsync to the server:
rsync -av homeassistant/ user@<server-ip>:~/homeassistant/ha_config/
```

Or copy file-by-file if rsync is not available.

### 3.2 Create secrets.yaml (never commit this file)
```bash
cat > ~/homeassistant/ha_config/secrets.yaml << 'EOF'
home_latitude: "40.4168"       # your latitude
home_longitude: "-3.7038"      # your longitude
home_elevation: "667"          # meters above sea level
EOF
```

### 3.3 Adjust entity IDs in scripts/modes.yaml
Open `ha_config/scripts/modes.yaml` and replace placeholder entity IDs:
- `light.living_room_main` → your actual light entity ID
- `light.living_room_ambient` → your ambient/bias light
- `climate.home` → your thermostat entity
- `area_id: bedroom` / `area_id: home` → your actual area names

Find entity IDs in HA: **Settings → Devices & Services → Entities**

### 3.4 Reload / restart HA
```bash
docker compose restart homeassistant
```

**Checkpoint:** Scripts appear under **Settings → Scripts**. Run `nap_mode` manually from UI — lights respond.

---

## Phase 4 — Configure OpenAI Conversation Integration

### 4.1 Add the integration
1. **Settings → Devices & Services → Add Integration**
2. Search for **"OpenAI Conversation"** → click it
3. Enter your OpenAI API key
4. Model: `gpt-4o-mini` (fast + cheap) or `gpt-4o` (more capable)
5. Click Submit

### 4.2 Set the System Prompt
1. Go to **Settings → Devices & Services → OpenAI Conversation → Configure**
2. Paste the full contents of `homeassistant/openai/system_prompt.txt`
3. Save

### 4.3 Restrict tool access (IMPORTANT for safety)
In the same OpenAI integration settings:
- **Control Home Assistant:** set to `No control` OR configure the allowlist
- If your HA version supports an "allowed intents" or "allowed scripts" list, add only:
  ```
  script.nap_mode
  script.sleep_mode
  script.away_mode
  script.movie_mode
  script.cozy_mode
  script.all_off
  script.good_morning
  ```
- This prevents the LLM from triggering arbitrary HA services.

### 4.4 Create a Voice Assistant using this agent
1. **Settings → Voice assistants → Add Assistant**
2. Name: `Home AI`
3. Conversation agent: `OpenAI Conversation`
4. Language: your language
5. Save

### 4.5 Test via HA UI
1. Go to **Settings → Voice assistants → Home AI → Try**
2. Type: `I'll take a nap`
3. Verify: `script.nap_mode` appears in Logbook

**Checkpoint:** LLM correctly routes all 5 mode phrases from HA's own UI chat.

---

## Phase 5 — iOS Siri Shortcut

See [`shortcuts/siri_shortcut_setup.md`](../shortcuts/siri_shortcut_setup.md) for full step-by-step instructions.

**Checkpoint (end-to-end):**
> "Hey Siri, Assistant" → "I'll take a nap" → lights dim → Siri speaks "Okay — nap mode activated."

---

## Phase 6 — Verify Homebridge Coexistence

```bash
# Check Homebridge is still running
docker ps          # if Homebridge runs in Docker
# or
systemctl status homebridge   # if running as a systemd service
```

Open iOS **Home** app → verify all existing Homebridge accessories still respond.

**Checkpoint:** Homebridge accessories fully functional. HA runs on port 8123, Homebridge on its own port (typically 51826/51827). No conflict.

---

## Ongoing Maintenance

### Adding a new mode
1. Add a new script block to `scripts/modes.yaml` (follow existing pattern).
2. Reload scripts: HA UI → **Developer Tools → YAML → Scripts**.
3. Add the new mode to the system prompt in `openai/system_prompt.txt`.
4. Update the OpenAI integration's system prompt in HA settings.

### Updating the OpenAI model
Settings → Devices & Services → OpenAI Conversation → Configure → change model.

### Monitoring costs
Check [platform.openai.com/usage](https://platform.openai.com/usage). `gpt-4o-mini` is ~$0.15/1M input tokens — typical home use costs cents per month.
