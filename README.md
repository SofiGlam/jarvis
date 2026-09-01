# JARVIS

Autonomous personal AI assistant with a real-time HUD, voice I/O, vision, long-term memory, and a large skill library. The brain fails over between **Gemini** and **Groq**. Skills can talk, control the desktop, search the web, manage files, and (on macOS) drive native apps.

---

## Table of contents

1. [Overview](#overview)
2. [What this repository contains](#what-this-repository-contains)
3. [Architecture](#architecture)
4. [Project layout](#project-layout)
5. [Requirements](#requirements)
   - [Hardware](#hardware)
   - [Software](#software)
   - [Python packages](#python-packages)
   - [Optional system tools](#optional-system-tools)
6. [Installation](#installation)
7. [Environment configuration](#environment-configuration)
8. [Speech models (Vosk)](#speech-models-vosk)
9. [Running JARVIS](#running-jarvis)
10. [Heads-up display (HUD)](#heads-up-display-hud)
11. [Holographic lab](#holographic-lab)
12. [Skill catalog](#skill-catalog)
13. [Core subsystems](#core-subsystems)
14. [Tests](#tests)
15. [Security and safety](#security-and-safety)
16. [Platform notes](#platform-notes)
17. [Troubleshooting](#troubleshooting)

---

## Overview

JARVIS is a **voice-first, skill-based agent**. You speak or type; the neural switchboard reasons with Gemini (primary) or Groq (failover); an executor runs named skills from `skills/`; results are spoken and pushed to a browser HUD over Socket.IO.

Typical capabilities:

- Conversational TTS (ElevenLabs → Microsoft Edge TTS → offline `pyttsx3`)
- Wake / listen via `SpeechRecognition` (and optional Vosk models)
- Owner voice-print verification (MFCC embeddings)
- Screen and camera vision
- Semantic memory (Gemini embeddings stored in `semantic_index.json`)
- Background watchdog: CPU/RAM/battery, scheduled briefing, filesystem events
- Dozens of tools: weather, calendar, email, WhatsApp, smart home, code assistant, vault, and more

Many desktop and OS skills were written for **macOS** (`osascript`, Spotlight, Shortcuts, HomeKit). Windows is partially supported (for example `shell_execution` remaps a few Unix commands). See [Platform notes](#platform-notes).

---

## What this repository contains

This tree is the **skills, HUD assets, and supporting engines**. Several runtime modules that skills import are **not in this snapshot** and must exist next to the project root for a full boot:

| Module | Role |
| --- | --- |
| Flask + Socket.IO app (typically `jarvis.py` or similar) | Serves `/` and `/lab`, emits HUD events |
| `autonomous_core.py` | Autonomous loop (`start_autonomous_core`) |
| `safety_manager.py` | Command/path policy and audit log |
| `coder_agent.py` | Used by skill synthesis |
| `filesystem_watcher.py` | Used by `SystemSentinel` |
| `requirements.txt` / `.env` | Not present; recreate from sections below |
| `static/style.css` | Referenced by the HUD templates |

If those files live in another checkout, copy them into this folder before running the full assistant.

---

## Architecture

```
User (voice / HUD text)
        │
        ▼
┌───────────────────┐     Socket.IO      ┌────────────────────┐
│  Browser HUD      │◄──────────────────►│  Flask + Socket.IO │
│  templates/index  │                    │  (main app)        │
└───────────────────┘                    └─────────┬──────────┘
                                                   │
                          ┌────────────────────────┼────────────────────────┐
                          ▼                        ▼                        ▼
                 NeuralSwitchboard          Skill executor            SystemSentinel
                 Gemini → Groq              skills/*.execute()       scheduler + telemetry
                          │                        │
                          ▼                        ▼
                 SemanticMemory              Safety manager
                 VisualObserver              SkillSynthesisEngine
```

**Failover:** `utils/neural_switchboard.py` tries Gemini first (rotating keys on HTTP 429), then Groq. The HUD can force `auto`, `gemini`, or `groq`.

**Skills:** Every skill module exposes `execute(params)`. The registry in `utils/skill_registry.py` is the contract the planner/executor should follow (name, description, params, risk).

---

## Project layout

```
jarvis/
├── README.md
├── skills/                    # One Python module per skill (execute())
├── utils/
│   ├── neural_switchboard.py  # Gemini + Groq brain
│   ├── gemini_rotator.py      # Extra Gemini key rotation helper
│   ├── skill_registry.py      # Skill list for prompts + param contracts
│   ├── audio_manager.py       # Speak/echo/music-state helpers
│   ├── voice_verifier.py      # Owner voice print (voice_prints/)
│   ├── system_discovery.py    # Installed apps + OS context (macOS-oriented)
│   └── speech_formatter.py
├── templates/
│   ├── index.html             # Main HUD
│   └── lab.html               # 3D topology / swarm lab
├── static/
│   ├── script.js              # HUD Socket.IO client
│   ├── lab.js / lab.css
│   └── style.css              # Expected; may be missing in this snapshot
├── semantic_memory.py         # Embedding index
├── visual_observer.py         # Periodic screenshots + vision context
├── system_sentinel.py         # Watchdog + daily briefing schedule
├── scheduler_system.py        # Background TaskScheduler
├── topology_engine.py         # File-graph for the lab 3D view
├── skill_synthesis_engine.py  # Auto-generate new skills
├── speech_formatter.py        # TTS text cleanup
├── setup_vosk.py              # Small US English Vosk model
├── setup_small_vosk.py        # Light Indian English model
├── setup_indian_vosk.py       # Large Indian English model (~1 GB)
└── test_*.py                  # Smoke tests
```

Runtime data files (created as you use JARVIS; do not commit secrets):

- `semantic_index.json` — vector memory
- `jarvis_memory.json` — ignored by topology engine; learned facts
- `voice_prints/owner.npy` — enrolled speaker
- `known_faces/` — face skill store
- `knowledge_base/` — RAG chunks
- `vault.enc` / `vault.hash` — encrypted secrets
- `clipboard_history.json`, notes/reminders JSON, `screenshots/`, `audit.log`

---

## Requirements

### Hardware

- Microphone (listen / enroll / verify)
- Speakers or headphones (TTS)
- Webcam (camera, face, mood) — optional
- Display for the HUD (Chrome or Edge recommended)

### Software

- **Python 3.12+** (bytecode in the repo includes 3.12 and 3.13)
- **pip** and a virtual environment
- A modern browser
- **macOS** for the richest OS integration; **Windows 10/11** for core AI, HUD, and a subset of skills
- Internet access for Gemini, Groq, weather, news, search, and optional ElevenLabs

### Python packages

There is no `requirements.txt` in this tree. Install at least:

```text
# Core AI and config
python-dotenv
google-genai
groq
numpy
Pillow
requests
psutil

# Web HUD (expected by templates/static)
flask
flask-socketio
python-socketio

# Voice
edge-tts
SpeechRecognition
pygame
pyttsx3

# Vision and desktop
opencv-python
pyautogui
pytesseract

# Files and watchers
watchdog
PyPDF2
python-docx
cryptography

# Optional / platform
scipy              # better MFCC DCT in voice_verifier
pygetwindow        # active window on Windows
pyobjc             # Quartz window bounds on macOS (via PyObjC)
vosk               # offline STT if you use the setup_*.py models
```

Microphone backends often need **PyAudio** (`pip install pyaudio`). On Windows that may require a matching wheel; on macOS, PortAudio via Homebrew first (`brew install portaudio`).

### Optional system tools

| Tool | Used for |
| --- | --- |
| [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) | `ocr`, `screen_analysis` fallback |
| Spotify / Apple Music | `spotify`, music URL open |
| Home Assistant | `smart_home` (`HA_URL`, `HA_TOKEN`) |
| macOS Shortcuts / HomeKit | `shortcuts`, `smart_home` |
| Twilio account | WhatsApp skill |
| SMTP mailbox | `email_sender` |

---

## Installation

### 1. Clone and enter the project

```bash
git clone <your-repo-url> jarvis
cd jarvis
```

### 2. Create a virtual environment

**Windows (PowerShell):**

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

**macOS / Linux:**

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

### 3. Install Python dependencies

```bash
pip install python-dotenv google-genai groq numpy Pillow requests psutil flask flask-socketio edge-tts SpeechRecognition pygame pyttsx3 opencv-python pyautogui pytesseract watchdog PyPDF2 python-docx cryptography
```

Add platform extras as needed (`pyaudio`, `scipy`, `vosk`, `pygetwindow`, `pyobjc`).

### 4. Restore missing core modules

Copy `safety_manager.py`, `autonomous_core.py`, the Flask entrypoint, and related files into the project root if they are not already here.

### 5. Configure secrets

Create a `.env` file in the project root (see [Environment configuration](#environment-configuration)). Never commit it.

### 6. Optional: Vosk offline models

```bash
python setup_vosk.py
# or: python setup_small_vosk.py
# or: python setup_indian_vosk.py
```

A small Hebrew model zip (`vosk-model-small-he-0.15.zip`) is present in this repo for Hebrew STT experiments.

### 7. Smoke-check APIs and camera

```bash
python test_jarvis.py
```

---

## Environment configuration

Create `.env` in the project root. `load_dotenv()` is used by the test scripts; the main app should load it the same way.

### Required for the brain

| Variable | Description |
| --- | --- |
| `GEMINI_API_KEYS` | One or more Google AI Studio keys, space- or comma-separated. Rotated on 429. |
| `GEMINI_MODEL` | e.g. `gemini-2.5-flash-lite` or `gemini-2.0-flash` |
| `GROQ_API_KEY` | Groq key(s), same multi-key format. Failover / vision-capable models. |

Some tests also read `GEMINI_API_KEY` (singular). Prefer `GEMINI_API_KEYS`.

### Voice and location

| Variable | Default | Description |
| --- | --- | --- |
| `ELEVENLABS_API_KEY` | empty | High-quality TTS; Edge TTS is used if unset |
| `ELEVENLABS_VOICE_ID` | `pNInz6obpgDQGcFmaJgB` (Adam) | ElevenLabs voice |
| `JARVIS_VOICE_THRESHOLD` | `0.78` | Cosine threshold for owner voice match |
| `JARVIS_CITY` | `Tel Aviv` | Default city for weather / briefing |
| `OPENWEATHER_API_KEY` | empty | [OpenWeather](https://openweathermap.org/api) |

### Messaging, news, home

| Variable | Description |
| --- | --- |
| `EMAIL_USER` / `EMAIL_PASS` | SMTP sender for `email_sender` |
| `TWILIO_SID` / `TWILIO_AUTH_TOKEN` / `TWILIO_WHATSAPP_FROM` | WhatsApp via Twilio |
| `TELEGRAM_BOT_TOKEN` | Telegram in `messenger` |
| `DISCORD_WEBHOOK_URL` | Discord in `messenger` |
| `NEWS_API_KEY` | NewsAPI.org for `news` |
| `HA_URL` | Home Assistant base URL (default `http://homeassistant.local:8123`) |
| `HA_TOKEN` | Home Assistant long-lived token |
| `JARVIS_VAULT_MASTER` | Master password for the `vault` skill |

Example `.env`:

```env
GEMINI_API_KEYS=your_gemini_key_here
GEMINI_MODEL=gemini-2.5-flash-lite
GROQ_API_KEY=your_groq_key_here
JARVIS_CITY=Tel Aviv
OPENWEATHER_API_KEY=
ELEVENLABS_API_KEY=
JARVIS_VAULT_MASTER=
```

---

## Speech models (Vosk)

Use these if you want **offline** speech recognition instead of (or in addition to) Google via `SpeechRecognition`.

| Script | Model | Size | Notes |
| --- | --- | --- | --- |
| `setup_vosk.py` | `vosk-model-small-en-us-0.15` → folder `model/` | ~40 MB | US English |
| `setup_small_vosk.py` | `vosk-model-small-en-in-0.4` | ~12 MB | Light Indian English |
| `setup_indian_vosk.py` | `vosk-model-en-in-0.5` | ~1 GB | Higher accuracy Indian English |

Downloads come from [alphacephei.com/vosk/models](https://alphacephei.com/vosk/models).

---

## Running JARVIS

Until the Flask entrypoint is restored, use this convention (adjust the filename to match your main module):

```bash
# with venv active and .env present
python jarvis.py
```

Then open the HUD:

- Main interface: `http://127.0.0.1:5000/` (port may differ in your app)
- Lab: `http://127.0.0.1:5000/lab`

Grant microphone, camera, and (on macOS) Accessibility / Screen Recording permissions when prompted — required for listen, vision, and UI control.

You can still run **individual skills** from Python for debugging:

```python
from skills.weather import execute
print(execute({"city": "Tel Aviv"}))
```

---

## Heads-up display (HUD)

`templates/index.html` + `static/script.js` implement a Stark-style HUD:

- Live clock, uptime, CPU / memory / battery / latency
- Neural bridge selector (auto / Gemini / Groq)
- Planner, researcher, coder, executor, observer, gesture, safety status
- Transcript, typed commands (`ui_command`)
- Voice level pulse, visual awareness, camera preview, image drop → `/api/analyse_image`
- Auth flows: PIN, face verify/register, voice enroll
- Emergency halt command from the UI

Client libraries loaded from CDN: Socket.IO, Three.js, DOMPurify, svg-pan-zoom, Mermaid.

---

## Holographic lab

`templates/lab.html` is a secondary view (`HOLOGRAPHIC_LAB_V4.2`) with:

- 3D digital-twin topology (`topology_engine.py` walks the repo and extracts imports)
- Swarm logs: researcher, coder, sentinel
- Link back to the main HUD

---

## Skill catalog

Skills live in `skills/` and are listed in `utils/skill_registry.py`. Each callable skill should define `execute(params)`.

### Voice, vision, UI

| Skill | Purpose |
| --- | --- |
| `speak` | TTS to the user; drives HUD `voice_level` |
| `listen` | One-shot microphone capture (Google recognizer) |
| `vision` | Screenshot or live camera on HUD |
| `camera` | Webcam still |
| `screen_capture` | Screenshot / region |
| `screen_analysis` | OpenCV + Tesseract on the screen |
| `ocr` | Extract / analyze text from screen or image |
| `generate_visual` | SVG, overlay, composite, Mermaid, HTML, 3D assembly on HUD |
| `face_rec` | Learn / identify / list / forget faces |
| `mood` | Mood from voice and/or face |
| `mouse_control` / `keyboard_control` | Desktop input |
| `volume` | System volume via PyAutoGUI |

### Memory, knowledge, code

| Skill | Purpose |
| --- | --- |
| `learn` / `recall_memory` | Long-term facts |
| `knowledge_base` | Ingest / search / ask (local RAG) |
| `code_assistant` | Write, explain, debug, review, refactor |
| `create_skill` / `synthesize_skill` | Generate new skill modules (gated by safety) |
| `doc_reader` | PDF / DOCX / text summarize and search |
| `summarize` | Text, URL, or clipboard |
| `translate` | Language translation |

### System and files

| Skill | Purpose |
| --- | --- |
| `system_monitor` | CPU, RAM, battery (`psutil`) |
| `shell_execution` | Terminal command (risk 1, safety-checked) |
| `smart_action` | Dynamic `osascript` / shell / `open` (macOS-oriented) |
| `open_app` | Launch apps / files / music links |
| `list_files` / `file_management` / `file_search` | Filesystem (Spotlight on Mac) |
| `file_watcher` | Watch a directory (`watchdog`) |
| `run_script` | Run a Python file from an allowed path |
| `clipboard` | Clipboard history |
| `shortcuts` | macOS Shortcuts + scheduled tasks |
| `scheduler` / `timer` | Timed tasks and countdowns |
| `notifications` | Monitor macOS notifications |
| `vault` | Encrypted secret store |

### Web, comms, daily life

| Skill | Purpose |
| --- | --- |
| `web_search` | DuckDuckGo snippets + spoken summary |
| `weather` | OpenWeather |
| `news` | Headlines by category |
| `calendar` / `reminders` / `notes` | Local JSON productivity |
| `email_sender` | Background SMTP |
| `send_whatsapp_message` | Twilio or browser fallback |
| `messenger` | iMessage / Telegram / Discord |
| `browser` | Safari/Chrome automation |
| `spotify` | Playback via AppleScript |
| `smart_home` | HomeKit Shortcuts or Home Assistant |
| `location` | IP/Wi-Fi location and labels |
| `daily_briefing` | Weather + calendar + reminders + news + health |
| `health` | Water, steps, nutrition, sleep log |
| `convert` / `calculator` | Units, FX, math |
| `hello` | Time-of-day greeting |

Risk **1** skills (`shell_execution`, `file_management`, `vault`, `run_script`, `synthesize_skill`) should always go through `safety_manager`.

---

## Core subsystems

| Component | File | Behavior |
| --- | --- | --- |
| Neural switchboard | `utils/neural_switchboard.py` | Gemini → Groq; streaming Gemini; image payloads for Groq vision |
| Gemini rotator | `utils/gemini_rotator.py` | Stateless generate with key rotation |
| Semantic memory | `semantic_memory.py` | Embed with `gemini-embedding-001`, cosine search, persist JSON |
| Visual observer | `visual_observer.py` | Interval screenshots, optional HUD `visual_awareness` |
| System sentinel | `system_sentinel.py` | Telemetry loop, low battery, security alerts, 09:00 daily briefing |
| Task scheduler | `scheduler_system.py` | One-shot and interval tasks (30 s poll) |
| Topology engine | `topology_engine.py` | Repo graph for the lab; skips `.env`, `audit.log`, memory JSON |
| Skill synthesis | `skill_synthesis_engine.py` | CoderAgent generates code, tests, then installs into `skills/` |
| Voice verifier | `utils/voice_verifier.py` | Enroll/verify owner; skip-open if no print enrolled |
| Skill registry | `utils/skill_registry.py` | Prompt skill list + executor param contract |
| System discovery | `utils/system_discovery.py` | Apps under `/Applications`, user/OS context for prompts |

---

## Tests

Run from the project root with the venv active and `.env` loaded where needed.

```bash
python test_jarvis.py          # Gemini ping + camera + edge-tts
python test_switchboard.py     # NeuralSwitchboard
python test_brain.py           # Gemini generate
python test_groq_direct.py     # Groq
python test_embeddings.py      # Embedding API
python test_all_skills.py      # Each skills/*.py has execute()
python test_skills.py          # Live open_app / shell / email / WhatsApp (needs creds)
python test_imports.py         # autonomous_core import
```

`test_all_skills.py` will report `IMPORT_ERROR` for skills that import missing modules such as `safety_manager`.

---

## Security and safety

- Treat this as a **local personal agent**. Skills can run shell commands, send messages, and read the screen.
- Keep `.env`, `vault.enc`, `audit.log`, and memory JSON **out of git**.
- `create_skill` blocks `eval`/`exec`, shell helpers, and destructive file APIs in generated code.
- `vault` uses Fernet when `cryptography` is installed; otherwise a weaker XOR fallback.
- Voice verification: if no owner print exists, `verify()` currently **allows** the speaker (`True, 1.0`). Enroll a print before relying on it.
- Grant OS accessibility permissions only if you intend JARVIS to control mouse, keyboard, and apps.

---

## Platform notes

**macOS (primary target)**

- `smart_action`, `spotify`, `shortcuts`, `notifications`, `file_search` (mdfind), `open`, AppleScript
- `system_discovery` scans `/Applications`
- `audio_manager` queries Apple Music via `osascript`
- Screen bounds via Quartz (`pyobjc`)

**Windows**

- HUD, Gemini/Groq, many JSON skills, PyAutoGUI, OpenCV, TTS
- `shell_execution` maps `ls` → `dir`, `grep` → `findstr`, `top`/`ps` → `tasklist`
- Active window via `pygetwindow` in vision/observer
- AppleScript, Spotlight, HomeKit, `pbpaste`, and `/Applications` discovery will not work without replacements

**Permissions**

- macOS: Microphone, Camera, Accessibility, Screen Recording
- Windows: Microphone and camera privacy settings

---

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| `ModuleNotFoundError: safety_manager` | Restore `safety_manager.py` to the project root |
| `autonomous_core` import fails | Restore `autonomous_core.py` |
| HUD has no styles | Add `static/style.css` |
| Gemini 429 | Add more keys to `GEMINI_API_KEYS`; Groq should take over |
| No speech | Install `edge-tts`; optional ElevenLabs; `pygame` for HUD pulse; `pyttsx3` offline |
| Mic silent | PyAudio / PortAudio; OS privacy; `listen` uses Google STT so needs network |
| OCR empty | Install Tesseract and ensure it is on `PATH` |
| Camera failed | `python test_jarvis.py`; close other apps using the webcam |
| Weather empty | Set `OPENWEATHER_API_KEY` and `JARVIS_CITY` |

---

## License

Not specified in this repository. Add a `LICENSE` file if you intend to share or publish the project.
