# Siri Voice Control Setup for Home Assistant + ChatGPT

## Step 1: Create a Long-Lived Access Token

1. Open Home Assistant in your browser: http://YOUR_HA_IP:8123
2. Click your profile (bottom left)
3. Scroll down to "Long-Lived Access Tokens"
4. Click "Create Token"
5. Name it: "Siri Shortcut"
6. Copy the token and save it somewhere safe

## Step 2: Create the iOS Shortcut

1. Open the **Shortcuts** app on your iPhone
2. Tap **+** to create a new shortcut
3. Add these actions in order:

### Action 1: Dictate Text
- Search for "Dictate Text"
- Add it
- Set language to English (or your preference)

### Action 2: Get Contents of URL
- Search for "Get Contents of URL"
- URL: `http://YOUR_HA_IP:8123/api/conversation/process`
- Method: **POST**
- Headers:
  - `Authorization`: `Bearer YOUR_LONG_LIVED_TOKEN`
  - `Content-Type`: `application/json`
- Request Body: **JSON**
  ```json
  {
    "text": "Dictated Text",
    "language": "en",
    "agent_id": "conversation.openai_conversation"
  }
  ```
  (Replace "Dictated Text" with the variable from Action 1)

### Action 3: Get Dictionary Value
- Search for "Get Dictionary Value"
- Get value for key: `response.speech.plain.speech`
- From: Contents of URL

### Action 4: Speak Text
- Search for "Speak Text"
- Text: Dictionary Value (from previous action)

## Step 3: Add to Siri

1. Tap the shortcut settings (...)
2. Tap "Add to Home Screen" and "Add to Siri"
3. Record a phrase like "Hey Siri, Home Control"

## Usage

Just say "Hey Siri, Home Control" and then speak naturally:
- "Good morning"
- "I'm going to sleep"
- "Movie time"
- "Turn everything off"
- "Start the vacuum"

ChatGPT will understand and execute the command!

## Example Commands

| Say this... | ChatGPT does... |
|-------------|-----------------|
| "Good morning" | Activates good_morning script |
| "I'm going to sleep" | Activates sleep_mode script |
| "Movie time" | Activates movie_mode script |
| "Turn off all lights" | Activates all_off script |
| "Start the vacuum" | Starts vacuum.xiaomi_ov31gl_971e_robot_cleaner |

## Troubleshooting

- **No response**: Check the token is correct
- **"Unauthorized"**: Token expired, create a new one
- **ChatGPT says it can't do something**: Update the OpenAI prompt in HA settings

## Your Device Entity IDs

For reference, here are your devices:

**Vacuum:**
- vacuum.xiaomi_ov31gl_971e_robot_cleaner

**Air Fryer:**
- button.xiaomi_maf07d_3dd7_start_cook
- button.xiaomi_maf07d_3dd7_pause
- button.xiaomi_maf07d_3dd7_cancel_cooking

**Smart Plug:**
- switch.cuco_v2eur_d335_switch

**Govee Lights (when working):**
- light.cocina
- light.lampara_living
- light.cuarto
