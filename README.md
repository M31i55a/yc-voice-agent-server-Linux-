# yc-voice-agent-server-Linux-

Bayview Pharmacy voice agent — a Pipecat-based AI assistant with video calling, visual prescription detection, gesture-based language switching, and a multilingual frontend.

## Features

- **Video call** with AI pharmacy agent (WebRTC via Daily)
- **Visual detection** — camera detects a prescription bottle/cup cue
- **Gesture recognition** — hand gestures select language (Pointing Up → English, Victory/ILoveYou → Spanish)
- **Language switcher** — toggle the UI between English, French, and Spanish

## Language Switcher

The frontend supports three languages:

| Language | Code | Selector label |
|----------|------|----------------|
| English  | `en` | English        |
| French   | `fr` | Français       |
| Spanish  | `es` | Español        |

A `<select>` dropdown appears in the top-right corner of both the pre-join lobby and the in-call view. Changing the language immediately updates all UI text, including:

- Branding and headers
- Status pills, badges, and device labels
- Button text (Mute, Leave, Send, etc.)
- Chat placeholder and system messages
- Avatar status messages

The active language is reflected in `state.activeLanguage` and sent to the backend as the preferred language for the AI agent.

### Adding a new language

1. Add a new `<option>` to both `<select class="lang-switcher">` elements in `demo_client/index.html`
2. Add translation entries for all keys in the `translations` object in `demo_client/app.js`
3. Add the language label to `LANG_CODES`

### Translating dynamic text

Static HTML uses `data-i18n` attributes. Dynamic JavaScript text uses `t("key")` calls. The `t()` function looks up the current language's translation dictionary and falls back to English if a key is missing.
