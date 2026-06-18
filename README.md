# yc-voice-agent-server-Linux-

Bayview Pharmacy voice agent — a Pipecat-based AI assistant for pharmacy calls with video, visual prescription detection, gesture-based language selection, and a multilingual UI.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Browser (vanilla HTML/CSS/JS)                  │
│  ├─ Pre-join lobby (camera/mic check)          │
│  ├─ In-call view (agent video, chat, controls) │
│  └─ MediaPipe (visual + gesture detection)     │
├────────────── WebRTC + Data Channel ────────────┤
│  Python Backend (FastAPI / Pipecat)             │
│  ├─ bot-gpt.py / bot-nemotron.py — LLM agent   │
│  ├─ language_router.py — spoken language match │
│  ├─ nvidia_stt.py — speech-to-text             │
│  ├─ nemotron_llm.py — LLM inference            │
│  └─ mock_backend.py — demo patient data        │
└─────────────────────────────────────────────────┘
```

**Frontend**: Vanilla ES module JavaScript (no framework). Served as static files via `demo_frontend.py`.

**Backend**: Python 3.11+ server powered by [Pipecat AI](https://github.com/pipecat-ai/pipecat) for real-time voice/video pipelines. Uses Daily WebRTC for transport.

---

## How It Works

### 1. Pre-join Lobby

The user arrives at the lobby, selects a microphone/camera/speaker, sees a camera preview, and clicks **Join now**.

- `initPreview()` starts the local camera and mic, populates device dropdowns, starts the mic level meter, and initialises MediaPipe visual + gesture detectors.
- When the user clicks **Join now**, `startCall()` does:

  1. Creates an `RTCPeerConnection` with the backend's `/start` endpoint
  2. Negotiates audio/video transceivers + a chat data channel
  3. Sends local audio/video tracks to the agent
  4. Switches to the in-call view

### 2. In-call View

Once connected, the user sees the agent's video tile, a status indicator, mute/leave controls, and a chat panel.

The backend streams:
- **Audio** from the AI agent (LLM-generated speech) via a remote audio track
- **Video** from an avatar or placeholder image
- **Transcript messages** over the data channel using the RTVI protocol

### 3. Visual Detection (Prescription Bottle Cue)

The frontend runs [MediaPipe Object Detection](https://developers.google.com/mediapipe/solutions/vision/object_detector) on the camera feed:

| Stage | Behaviour |
|-------|-----------|
| Loading | Badge shows *"Visual detection loading"* |
| Ready | Badge shows *"Watching for prescription bottle cue"* |
| Partial detection | Badge shows *"Possible prescription bottle cue"* |
| Confirmed (hits threshold) | Sends a `VISUAL_CONTEXT` system message to the LLM, which asks the caller if they're calling for a refill |

Detection is gated by `VISUAL_DETECTION_THRESHOLD`, `VISUAL_DETECTION_INTERVAL_MS`, and `VISUAL_DETECTION_REQUIRED_HITS` constants.

### 4. Gesture Recognition (Language Selection)

[MediaPipe Gesture Recognition](https://developers.google.com/mediapipe/solutions/vision/gesture_recognizer) maps hand gestures to languages:

| Gesture | Language | Code |
|---------|----------|------|
| Pointing Up | English | `en` |
| Victory / ILoveYou | Spanish | `es` |

When a gesture is confirmed (`GESTURE_DETECTION_REQUIRED_HITS`), the frontend sends a `GESTURE_CONTEXT` system message instructing the AI to switch languages. The active language value (`state.activeLanguage`) is also consumed by the UI language switcher.

### 5. Language Switcher

A `<select>` dropdown in the top-right corner of both views lets the user switch the UI language at any time.

| Language | Code | Selector label |
|----------|------|----------------|
| English  | `en` | English        |
| French   | `fr` | Français       |
| Spanish  | `es` | Español        |

- Static HTML text uses `data-i18n="key"` attributes
- Dynamic JS text uses `t("key")` calls
- All translations live in the `translations` object in `app.js`
- The `t()` function falls back to English if a key is missing in the current language
- Switching language also updates `state.activeLanguage`, which the backend sees in gesture-based language requests

#### Adding a new language

1. Add a `<option value="xx">Native Name</option>` to both `<select class="lang-switcher">` in `index.html`
2. Add translation entries for all keys in the `translations` object in `app.js` (three language blocks: `en`, `fr`, `es`)
3. Add the language to `LANG_CODES`

---

## Setup

### Requirements

- Python 3.11+
- `uv` package manager (or `pip`)

### Installation

```bash
uv sync
```

### Configuration

Copy `.env.example` to `.env` and fill in the required API keys:

```
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-...
DAILY_API_KEY=...
```

### Running

```bash
uv run python demo_frontend.py
```

Then open `http://localhost:8765` in a browser.

---

## Project Structure

| File | Purpose |
|------|---------|
| `demo_client/index.html` | Frontend HTML (pre-join lobby + in-call view) |
| `demo_client/app.js` | Frontend JS (WebRTC, MediaPipe, chat, i18n) |
| `demo_client/styles.css` | Frontend styles |
| `demo_frontend.py` | FastAPI server serving static files and WebRTC endpoints |
| `bot-gpt.py` | OpenAI GPT-based voice agent pipeline |
| `bot-nemotron.py` | NVIDIA Nemotron-based voice agent pipeline |
| `language_router.py` | Detects spoken language from transcript and sets LLM context |
| `nvidia_stt.py` | NVIDIA speech-to-text integration |
| `nemotron_llm.py` | NVIDIA Nemotron LLM inference client |
| `mock_backend.py` | Mock patient/prescription database for demo |
| `stt_provider.py` | Abstract STT provider interface |
| `video_avatar.py` | Avatar video rendering / animation |
| `test_language_router.py` | Tests for language routing |
| `test_nemotron_llm.py` | Tests for Nemotron LLM |
| `.env` | API keys and secrets (not tracked in git) |
| `pyproject.toml` | Python dependencies |

---

## Frontend-Backend Communication

### WebRTC

- The frontend creates a peer connection and negotiates with the backend's `/start` endpoint
- Audio/video tracks are sent/received via transceivers
- ICE candidates are exchanged over an HTTP signalling channel

### Data Channel (RTVI)

- A `chat` labelled data channel carries RTVI messages
- **System messages**: `VISUAL_CONTEXT` (bottle detected), `GESTURE_CONTEXT` (language change)
- **Transcripts**: `user-transcription`, `user-llm-text`, `bot-ready`, `bot-started-speaking`, `bot-stopped-speaking`
- **User chat**: plain text sent via `send-text` RTVI action

### Status Flow

```
Connecting → Requesting media → Media connected → Live
```

Errors at any stage return the user to the lobby with a descriptive message.
