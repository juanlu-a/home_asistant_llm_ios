# iOS Siri Shortcut — Setup Guide

## Overview
One universal Shortcut acts as the voice entry point. The user says:
> "Hey Siri, Assistant"

Siri asks for dictation input, the text is sent to Home Assistant's Conversation API, and the response is spoken back.

---

## Prerequisites
- iPhone/iPad on the same Wi-Fi network as the Home Assistant server.
- Home Assistant reachable at `http://<server-ip>:8123` (or your HA Cloud URL).
- A **Long-Lived Access Token** from HA (see step 0 below).

---

## Step 0 — Generate a Long-Lived Access Token in HA
1. In HA, click your **profile icon** (bottom-left).
2. Scroll to **Long-Lived Access Tokens** → **Create Token**.
3. Name it "iOS Shortcut".
4. Copy the token — you only see it once. Paste it into the Shortcut (step 5).

---

## Shortcut Actions (in order)

### Action 1 — Dictate Text
- **Action:** `Dictate Text`
- **Language:** Default (auto-detects)
- **Prompt:** *(leave empty — Siri will just listen)*
- **Variable output name:** `DictatedText`

---

### Action 2 — Get Contents of URL (POST to HA Conversation API)
- **Action:** `Get Contents of URL`
- **URL:**
  ```
  http://<server-ip>:8123/api/conversation/process
  ```
  *(Replace `<server-ip>` with your server's LAN IP, e.g. `192.168.1.100`)*
  *(If using HA Cloud / Nabu Casa, use `https://<your-id>.ui.nabu.casa/api/conversation/process`)*

- **Method:** `POST`

- **Headers:**
  | Key | Value |
  |-----|-------|
  | `Authorization` | `Bearer <your-long-lived-token>` |
  | `Content-Type` | `application/json` |

- **Request Body:** `JSON`
  ```json
  {
    "text": "DictatedText",
    "language": "en"
  }
  ```
  *(In the Shortcut editor, tap the `"DictatedText"` string value and replace it with the variable from Action 1.)*
  *(Change `"en"` to your language code if needed, e.g. `"es"` for Spanish.)*

- **Variable output name:** `HAResponse`

---

### Action 3 — Get Value from Dictionary
- **Action:** `Get Value for Key in Dictionary`
- **Dictionary:** `HAResponse` (the output of Action 2)
- **Key:** `response`
- **Variable output name:** `ResponseDict`

*(HA wraps the reply text inside `response → speech → plain → speech`)*

---

### Action 4 — Get Nested Value
- **Action:** `Get Value for Key in Dictionary`
- **Dictionary:** `ResponseDict`
- **Key:** `speech`
- **Variable output name:** `SpeechDict`

---

### Action 5 — Get Final Text
- **Action:** `Get Value for Key in Dictionary`
- **Dictionary:** `SpeechDict`
- **Key:** `plain`
- **Variable output name:** `PlainDict`

---

### Action 6 — Extract Speech String
- **Action:** `Get Value for Key in Dictionary`
- **Dictionary:** `PlainDict`
- **Key:** `speech`
- **Variable output name:** `ConfirmationText`

---

### Action 7 — Speak (optional but recommended)
- **Action:** `Speak Text`
- **Text:** `ConfirmationText`

---

### Action 8 — Show Notification (optional fallback)
- **Action:** `Show Notification`
- **Body:** `ConfirmationText`
- **Title:** `Home Assistant`

---

## Setting the Siri Trigger
1. Tap the Shortcut name at the top → **Add to Siri**.
2. Record the phrase: **"Assistant"**
   (so you say "Hey Siri, Assistant")
3. Save.

---

## Simplified JSON Path Note
The full HA Conversation API response structure is:
```json
{
  "response": {
    "speech": {
      "plain": {
        "speech": "Okay — nap mode activated.",
        "extra_data": null
      }
    },
    "card": {},
    "language": "en",
    "response_type": "action_done",
    "data": {}
  },
  "conversation_id": "..."
}
```
Actions 3–6 walk this path. If your HA version differs, use **Developer Tools → Template** in HA to inspect the actual response shape and adjust accordingly.

---

## Testing the Shortcut
1. Open the Shortcut in the Shortcuts app.
2. Tap the **▶ Play** button.
3. Speak: `"I'll take a nap"`
4. Verify in HA → **Logbook** that `script.nap_mode` ran.
5. Verify you hear: `"Okay — nap mode activated."`

---

## Troubleshooting
| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| "Couldn't connect" error | Wrong IP or HA not running | Check `http://<ip>:8123` in Safari |
| 401 Unauthorized | Bad/expired token | Regenerate token in HA profile |
| Empty response spoken | JSON path wrong | Inspect raw `HAResponse` in Shortcut result |
| Wrong mode triggered | LLM misinterpretation | Rephrase; check system prompt in HA |
| Shortcut times out | HA or OpenAI slow | Increase Shortcut timeout (Settings) |
