# Agentic AI Video Pipeline

> An end-to-end **Visual Novel Video Generator** that turns a natural-language prompt into a ~1-minute animated MP4 — powered by LangGraph multi-agent orchestration, Groq LLM, Pollinations.ai image generation, MoviePy compositing, and a natural-language **Edit Agent** with multi-level undo.

---

## Table of Contents

1. [Overview](#overview)
2. [Key Features](#key-features)
3. [System Architecture](#system-architecture)
4. [Technology Stack](#technology-stack)
5. [Project Structure](#project-structure)
6. [Pipeline Flow](#pipeline-flow)
7. [Agent Details](#agent-details)
8. [MCP Tools Layer](#mcp-tools-layer)
9. [Data Schemas](#data-schemas)
10. [State Management & Undo](#state-management--undo)
11. [Backend API Reference](#backend-api-reference)
12. [Frontend Application](#frontend-application)
13. [Setup & Installation](#setup--installation)
14. [Running the Application](#running-the-application)
15. [Docker Deployment](#docker-deployment)
16. [Render Deployment](#render-deployment)
17. [Free Deployment (Full Quality)](#free-deployment-full-quality)
18. [Running Tests](#running-tests)
19. [Edit Agent](#edit-agent)
19. [Generated Output Files](#generated-output-files)
20. [Environment Variables](#environment-variables)
21. [Example Prompts](#example-prompts)
22. [Troubleshooting](#troubleshooting)
23. [License](#license)

---

## Overview

This project implements a fully autonomous AI-powered visual novel video generation system. A user types a story prompt in a React dashboard; the backend orchestrates four specialized agents through a LangGraph DAG to produce a complete animated video with dialogue, character sprites, scene backgrounds, lip-sync animation, and cinematic Ken-Burns camera movement.

After generation, the user can apply **natural-language edits** ("Change Alice's hair to red", "Make scene 2 a rainy city", "undo") which are classified by a Groq LLM, applied to pipeline state, and re-rendered — with full **multi-level undo** that restores both JSON state and generated asset files.

**Typical output:** A 50–70 second MP4 at 1280×720, 24fps, with two characters, 3–5 scenes, and gender-aware text-to-speech.

---

## Key Features

| Feature | Description |
|---------|-------------|
| **Multi-agent orchestration** | LangGraph DAG: Story → Audio → Video, with a separate Edit Agent |
| **Structured LLM output** | Groq LLM generates validated JSON via Pydantic schemas |
| **Free image generation** | Pollinations.ai — no API key required |
| **Background removal** | `rembg` produces transparent character sprites |
| **Cinematic backgrounds** | Ken-Burns pan/zoom with vignette and contrast boost |
| **Gender-aware TTS** | Female characters get Zira voice, male get David (Windows SAPI5) |
| **Lip-sync mouth overlay** | Amplitude-based open/close animation at face position |
| **Turn-wise rendering** | Only the current speaker is visible per dialogue line |
| **Natural-language editing** | 10 intent types classified by Groq LLM |
| **Multi-level undo** | In-memory stack + disk-persisted snapshots with full asset restoration |
| **Real-time progress** | WebSocket streaming of pipeline phase updates |
| **Automated tests** | 70+ unit and integration tests |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   React Frontend (port 5173)                     │
│   PromptForm │ PhaseProgress │ VideoPlayer │ EditPanel │ History │
└────────────────────────────┬────────────────────────────────────┘
                             │  REST API  +  WebSocket
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  FastAPI Backend (port 8001)                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Routes: /api/pipeline, /api/edit, /api/assets           │  │
│  │ WebSocket: /ws/progress/{job_id}                        │  │
│  │ PipelineService: async job lifecycle                    │  │
│  └───────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌────────────┐
       │Story Agent │ │Audio Agent │ │Video Agent │
       │ (Groq LLM) │ │ (pyttsx3)  │ │ (MoviePy)  │
       └────────────┘ └────────────┘ └────────────┘
              │              │              │
              └──────────────┼──────────────┘
                             ▼
              ┌──────────────────────────────┐
              │       Edit Agent             │
              │  Intent Classification (Groq)│
              │  State Mutation + Rerender   │
              │  Undo Stack + Disk Snapshots │
              └──────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
    ┌──────────────────┐         ┌──────────────────────┐
    │  MCP Tool Layer  │         │   State Manager      │
    │  LLM / Audio /   │         │  history.json        │
    │  Vision / Video  │         │  v_N.json + assets   │
    └──────────────────┘         └──────────────────────┘
              │
              ▼
    data/outputs/{job_id}/
    data/state_versions/{job_id}/
```

---

## Technology Stack

| Layer | Technologies |
|-------|-------------|
| **Orchestration** | LangGraph, LangChain, LangChain-Groq |
| **Backend** | Python 3.10, FastAPI, Uvicorn, Pydantic, WebSockets |
| **Frontend** | React 18, TypeScript, Vite |
| **LLM** | Groq API (`llama-3.3-70b-versatile`) |
| **Image Generation** | Pollinations.ai (free, no key) |
| **Background Removal** | rembg (u2net model, ~170 MB, downloaded once) |
| **Video Compositing** | MoviePy, Pillow, OpenCV, NumPy, SciPy |
| **Text-to-Speech** | pyttsx3 with Windows SAPI5 voices |
| **Testing** | pytest |

---

## Project Structure

```
.
├── agents/                          # All pipeline agents
│   ├── story_agent/
│   │   ├── agent.py                 # StoryAgent — calls Groq, validates StorySpec
│   │   └── planner.py               # build_story_prompt() — LLM prompt template
│   ├── audio_agent/
│   │   └── agent.py                 # AudioAgent — gender-aware TTS + timing manifest
│   ├── video_agent/
│   │   └── agent.py                 # VideoAgent — image gen, compositing, MP4 export
│   ├── edit_agent/
│   │   ├── agent.py                 # EditAgent — intent classification + undo stack
│   │   ├── intent_classifier.py     # Groq LLM classifier
│   │   ├── executor.py              # Applies edit plans to job state
│   │   ├── planner.py               # Edit planning logic
│   │   └── tests/
│   │       └── test_edit_agent.py   # 29 unit tests for edit agent
│   └── orchestrator/
│       ├── graph.py                 # LangGraph StateGraph builder
│       ├── workflow.py              # PipelineWorkflow — coordinates all agents
│       └── state.py                 # OrchestratorState TypedDict
│
├── backend/                         # FastAPI server
│   ├── app.py                       # App entry point, CORS, WebSocket, routers
│   ├── routes/
│   │   ├── pipeline.py              # POST /start, GET /{job_id}
│   │   ├── edit.py                  # POST /edit, POST /undo, GET /history
│   │   └── assets.py                # GET /{job_id}/{name} — serve MP4/PNG files
│   ├── services/
│   │   └── pipeline_service.py      # Job lifecycle, async graph invocation
│   └── websocket/
│       └── manager.py               # ConnectionManager — broadcast to clients
│
├── frontend/                        # React dashboard
│   ├── src/
│   │   ├── App.tsx                  # Main app — pipeline start, WS, edit, undo
│   │   ├── components/
│   │   │   ├── PromptForm.tsx       # Story prompt input + Generate button
│   │   │   ├── PhaseProgress.tsx    # Real-time pipeline phase display
│   │   │   ├── VideoPlayer.tsx      # Final MP4 player
│   │   │   ├── EditPanel.tsx        # Natural-language edit input + Undo
│   │   │   └── VersionHistoryPanel.tsx  # Edit version history list
│   │   └── styles.css               # Dark-mode glassmorphism UI
│   └── package.json
│
├── mcp/                             # Model Context Protocol tool layer
│   ├── base_tool.py                 # Abstract BaseTool interface
│   ├── tool_registry.py             # Dynamic tool registration
│   ├── tool_executor.py             # Tool execution wrapper
│   └── tools/
│       ├── llm_tools/
│       │   ├── json_structurer.py   # Groq → validated JSON (StorySpec, EditIntent)
│       │   └── text_generator.py    # Groq → free-form text
│       ├── audio_tools/
│       │   ├── tts_tool.py          # pyttsx3 gender-aware TTS → WAV
│       │   ├── audio_merger.py      # Concatenate WAV files → master_dialogue.wav
│       │   └── bgm_tool.py          # Background music tool
│       ├── vision_tools/
│       │   ├── image_gen_tool.py    # Pollinations.ai image generation
│       │   ├── image_edit_tool.py   # Image editing utilities
│       │   └── style_transfer.py    # Style transfer utilities
│       ├── video_tools/
│       │   ├── compositor_tool.py   # Video compositing helpers
│       │   ├── ffmpeg_tool.py       # FFmpeg utilities
│       │   └── subtitle_tool.py     # Subtitle rendering
│       └── system_tools/
│           ├── file_tool.py         # File I/O helpers
│           ├── logger_tool.py       # Logging utilities
│           └── state_tool.py        # State read/write helpers
│
├── shared/                          # Shared contracts and utilities
│   ├── schemas/__init__.py          # All Pydantic models (StorySpec, EditIntent, etc.)
│   ├── constants/__init__.py        # Video FPS, resolution, phase enums
│   └── utils/
│       ├── config.py                # Settings from .env (Pydantic Settings)
│       ├── io.py                    # read_json / write_json helpers
│       └── logging.py               # setup_logger()
│
├── state_manager/                   # Versioned state persistence
│   ├── state_manager.py             # snapshot(), undo(), list_versions()
│   ├── snapshot.py                  # Snapshot dataclass
│   ├── history.py                   # Append-only history.json manager
│   └── storage.py                   # Path helpers for state directories
│
├── tests/
│   ├── unit/
│   │   ├── test_schemas.py          # Pydantic schema validation tests
│   │   ├── test_video_agent.py      # VideoAgent geometry/math tests
│   │   └── test_state_manager.py    # Snapshot/undo tests
│   └── integration/
│       └── test_edit_agent_integration.py  # Live Groq LLM edit tests
│
├── scripts/
│   ├── demo_edit_agent.py           # CLI demo: 11 scripted edits + undo
│   ├── start_backend.ps1            # Windows helper to start backend
│   └── start_frontend.ps1           # Windows helper to start frontend
│
├── docs/
│   └── demo/
│       └── edit_agent_demo.mp4      # Pre-recorded demo video
│
├── data/
│   ├── outputs/{job_id}/            # Generated artifacts per job (gitignored)
│   └── state_versions/{job_id}/     # Versioned edit snapshots (gitignored)
│
├── .env                             # API keys (not committed)
├── .env.example                     # Environment variable template
├── requirements.txt                 # Python dependencies
└── Readme.md                        # This file
```

---

## Pipeline Flow

### End-to-End Generation

```
User types prompt in React UI
        │
        ▼
POST /api/pipeline/start  { "prompt": "..." }
        │
        ▼
PipelineService.start_job()
  - Creates job_id (UUID hex)
  - Spawns async LangGraph execution
        │
        ▼
┌─────────────────────────────────────────────────────┐
│  LangGraph DAG (agents/orchestrator/graph.py)       │
│                                                     │
│  [Story Node]                                       │
│    StoryAgent.run(job_id, user_prompt)              │
│    → Groq LLM generates StorySpec JSON              │
│    → Saved to data/outputs/{job_id}/story_spec.json │
│    → WebSocket: phase=story                         │
│                                                     │
│  [Audio Node]                                       │
│    AudioAgent.run(job_id, story_spec)               │
│    → Per-line TTS WAV files (gender-aware)          │
│    → timing_manifest.json (ms-accurate markers)     │
│    → master_dialogue.wav (concatenated)             │
│    → WebSocket: phase=audio                         │
│                                                     │
│  [Video Node]                                       │
│    VideoAgent.run(job_id, story, timing, audio)     │
│    → Character sprites via Pollinations.ai          │
│    → Background removal via rembg                   │
│    → Scene backgrounds via Pollinations.ai          │
│    → Ken-Burns animation + lip-sync compositing     │
│    → final_output.mp4                               │
│    → WebSocket: phase=video                         │
│                                                     │
│  [END] → WebSocket: phase=done                      │
└─────────────────────────────────────────────────────┘
        │
        ▼
Frontend VideoPlayer loads /api/assets/{job_id}/final_output.mp4
```

### Edit Flow

```
User types edit query in EditPanel
        │
        ▼
POST /api/edit/{job_id}  { "query": "Change Alice's hair to red" }
        │
        ▼
EditAgent.handle(query, state)
  1. Groq LLM classifies intent → EditIntent JSON
  2. EditExecutor applies state mutation
  3. StateManager.snapshot() — saves v_N.json + asset copy
  4. If rerender_required → background thread reruns Audio + Video
  5. WebSocket broadcasts progress during rerender
        │
        ▼
POST /api/edit/{job_id}/undo
  → StateManager.undo() restores previous v_N.json + asset files
```

---

## Agent Details

### Story Agent (`agents/story_agent/`)

**Purpose:** Convert a free-text user prompt into a structured `StorySpec` JSON.

**How it works:**
1. `build_story_prompt()` wraps the user prompt with hard constraints:
   - Exactly 2 characters
   - 3–5 scenes, 50–70 second total runtime
   - Dialogue only between the two characters
   - Visual descriptions suitable for image generation
2. `GroqJsonStructurerTool` calls Groq LLM with the `StorySpec` Pydantic schema
3. Up to 5 retry attempts if validation fails
4. Output saved to `data/outputs/{job_id}/story_spec.json`

**Revise mode:** When an edit changes dialogue or story structure, `StoryAgent.revise()` sends the current story + edit request back to Groq for a targeted revision.

### Audio Agent (`agents/audio_agent/`)

**Purpose:** Generate per-line TTS audio and a timing manifest.

**How it works:**
1. Reads `StorySpec` and builds a gender map from `visual_traits` + `voice_personality` keywords
2. For each dialogue line in each scene:
   - Selects voice: **Zira** (female) or **David** (male) via pyttsx3/SAPI5
   - Adjusts speech rate by emotion (sad → slower, angry → faster)
   - Generates `{scene_id}_{idx}_{speaker}.wav`
   - Records `start_ms` / `end_ms` in timing manifest
3. Merges all WAVs into `master_dialogue.wav`
4. Saves `timing_manifest.json`

**Platform note:** TTS requires Windows 10/11 with SAPI5 voices installed.

### Video Agent (`agents/video_agent/`)

**Purpose:** Composite all visual and audio assets into a final MP4.

**Pipeline stages inside VideoAgent:**

| Stage | What happens |
|-------|-------------|
| **Character generation** | Pollinations.ai generates full-body anime sprites (768×1024) per character |
| **Background removal** | `rembg` removes background → transparent PNG (`char_{name}.png`) |
| **Scene backgrounds** | Pollinations.ai generates 1280×720 backgrounds per scene |
| **Ken-Burns animation** | Slow pan + zoom + vignette + contrast boost on backgrounds |
| **Mouth overlay** | Amplitude-based lip-sync ellipse drawn at 20% down character body |
| **Turn-wise compositing** | Only the current speaker's sprite is visible per dialogue line |
| **Audio sync** | Video trimmed to exact dialogue duration from timing manifest |
| **Export** | MP4 via libx264 at 1280×720, 24fps |

**Key constants** (from `shared/constants/` and `video_agent/agent.py`):
- Resolution: 1280 × 720
- FPS: 24
- Character height: 88% of frame, anchored 12% from top
- Mouth position: 20% down character body height

### Edit Agent (`agents/edit_agent/`)

**Purpose:** Classify natural-language edit requests and apply them with undo support.

**Components:**
- `EditAgent` — main handler, Groq classification, state mutation
- `EditAgentState` — in-memory undo stack + disk persistence via StateManager
- `EditExecutor` — writes edit plans to `job_state.json`
- `intent_classifier.py` — structured Groq output → `ClassifiedIntent`

**Rerender logic** (`backend/routes/edit.py`):
When an edit requires visual/audio changes, a background thread:
1. Optionally revises the story spec via `StoryAgent.revise()`
2. Re-runs `AudioAgent` and `VideoAgent`
3. Broadcasts WebSocket progress events throughout

---

## MCP Tools Layer

The MCP (Model Context Protocol) layer provides a standardized tool interface used by all agents.

**Base interface** (`mcp/base_tool.py`):
```python
class BaseTool:
    name: str
    description: str
    def run(self, **kwargs) -> Any: ...
```

**Tool Registry** (`mcp/tool_registry.py`): Agents register and instantiate tools by name.

| Tool | File | Purpose |
|------|------|---------|
| `GroqJsonStructurerTool` | `llm_tools/json_structurer.py` | Prompt → Groq → validated Pydantic JSON |
| `GroqTextGeneratorTool` | `llm_tools/text_generator.py` | Prompt → Groq → free text |
| `CoquiTTSTool` | `audio_tools/tts_tool.py` | Text → pyttsx3 → WAV file |
| `AudioMergerTool` | `audio_tools/audio_merger.py` | List[WAV] → master_dialogue.wav |
| `HFImageGenTool` | `vision_tools/image_gen_tool.py` | Prompt → Pollinations.ai → PNG |
| `ImageBackgroundRemovalTool` | `vision_tools/image_gen_tool.py` | PNG → rembg → transparent PNG |

**Image generation details:**
- Backgrounds: 1280×720 via `https://image.pollinations.ai/prompt/{encoded}?width=1280&height=720`
- Characters: 768×1024
- Style suffix appended: `", high quality anime visual novel style, masterpiece"`
- 429 rate-limit retry with exponential backoff (3 attempts)

---

## Data Schemas

All data contracts are defined in `shared/schemas/__init__.py` using Pydantic v2.

### StorySpec
```python
class StorySpec:
    story: str                          # Narrative summary
    theme: str                          # e.g. "mystery", "romance"
    characters: List[CharacterSpec]     # Exactly 2
    scenes: List[SceneSpec]             # 3–8 scenes, total 50–70s

class CharacterSpec:
    name: str
    voice_personality: str              # Used for gender detection + TTS
    visual_traits: str                    # Used for image generation prompt

class SceneSpec:
    scene_id: str                       # e.g. "scene1"
    title: str
    visual_description: str             # Background image prompt
    duration_seconds: int               # 6–25 seconds per scene
    dialogue: List[DialogueLine]

class DialogueLine:
    speaker: str                        # Must be one of the 2 character names
    text: str
    emotion: str = "neutral"            # Affects TTS rate
```

### TimingManifest
```python
class TimingEntry:
    scene_id: str
    speaker: str
    text: str
    audio_file: str                     # Path to per-line WAV
    start_ms: int
    end_ms: int

class TimingManifest:
    entries: List[TimingEntry]
    total_duration_ms: int
```

### EditIntent / EditResult
```python
class EditIntent:
    intent: str                         # One of 10 supported intent types
    target: str                         # "audio" | "video_frame" | "video" | "script" | "system"
    confidence: float                 # 0.0–1.0
    params: Dict[str, Any]             # Extracted edit parameters

class EditResult:
    applied: bool
    target: str
    details: str
    updated_state_path: Optional[str]
    rerender_required: bool
```

---

## State Management & Undo

The `StateManager` (`state_manager/state_manager.py`) provides versioned, disk-persisted undo that restores **both JSON state and generated asset files**.

### On each successful edit:
```
data/state_versions/{job_id}/
├── history.json          # Ordered list of all versions
├── v_1.json              # State snapshot after edit 1
├── assets_v_1/           # Full copy of data/outputs/{job_id}/ at edit 1
├── v_2.json
├── assets_v_2/
└── ...
```

### Undo operation:
1. Pop the latest entry from `history.json`
2. Load the previous `v_N.json` state
3. Copy `assets_v_{N-1}/` back to `data/outputs/{job_id}/`
4. Return restored state

This means undo restores the actual MP4, PNG, and WAV files — not just metadata.

---

## Backend API Reference

Interactive API docs available at: **http://localhost:8001/docs**

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/pipeline/start` | Start video generation. Body: `{ "prompt": "..." }`. Returns `{ "job_id": "..." }` |
| `GET` | `/api/pipeline/{job_id}` | Get job status, events, and metadata |
| `GET` | `/api/assets/{job_id}/{name}` | Stream a generated asset (e.g. `final_output.mp4`) |
| `POST` | `/api/edit/{job_id}` | Apply natural-language edit. Body: `{ "query": "..." }` |
| `POST` | `/api/edit/{job_id}/undo` | Undo the last edit |
| `GET` | `/api/edit/{job_id}/history` | List all edit versions with timestamps |
| `WS` | `/ws/progress/{job_id}` | Real-time progress events |
| `GET` | `/health` | Health check — returns `{ "ok": true }` |

### WebSocket Progress Event Format
```json
{
  "phase": "story | audio | video | edit | done | error",
  "status": "queued | running | completed | failed",
  "percent": 0,
  "meta": { "final_video_path": "...", "msg": "..." }
}
```

---

## Frontend Application

**Stack:** React 18 + TypeScript + Vite, hand-written CSS (dark glassmorphism theme).

**Components:**

| Component | File | Purpose |
|-----------|------|---------|
| `App` | `App.tsx` | Orchestrates pipeline start, WebSocket subscription, edit/undo |
| `PromptForm` | `components/PromptForm.tsx` | Textarea + Generate button |
| `PhaseProgress` | `components/PhaseProgress.tsx` | Live pipeline phase event list |
| `VideoPlayer` | `components/VideoPlayer.tsx` | HTML5 video player for final MP4 |
| `EditPanel` | `components/EditPanel.tsx` | Edit query input + Apply + Undo buttons |
| `VersionHistoryPanel` | `components/VersionHistoryPanel.tsx` | Edit version history with refresh |

**API base URL:** Configured via `VITE_API_BASE` env variable (default: `http://localhost:8001`).

**WebSocket flow:**
1. On pipeline start, opens `ws://localhost:8001/ws/progress/{job_id}`
2. Listens for phase events and updates `PhaseProgress`
3. On `phase === "done"`, polls asset endpoint with HEAD retry before setting video src

---

## Setup & Installation

**Estimated time:** 10–15 minutes on a fresh Windows machine.

### Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Python | 3.10 | Via Miniconda/Anaconda recommended |
| Node.js | 18+ | For frontend |
| ffmpeg | latest | Must be on system PATH |
| Git | latest | For cloning |

> **Windows TTS:** Uses built-in SAPI5 voices (Microsoft David & Zira). No install needed on Windows 10/11.

### Step 1 — Create Python Environment

```bash
conda create -n agenticai python=3.10 -y
conda activate agenticai
pip install -r requirements.txt
```

> `rembg` downloads a ~170 MB model on first run. This is normal and only happens once.

### Step 2 — Configure Environment

```bash
copy .env.example .env
```

Edit `.env` and set at minimum:
```env
GROQ_API_KEY=your_groq_api_key_here
GROQ_MODEL=llama-3.3-70b-versatile
FRONTEND_ORIGIN=http://localhost:5173
```

Get a free Groq API key at: https://console.groq.com

### Step 3 — Install Frontend Dependencies

```bash
cd frontend
npm install
cd ..
```

---

## Running the Application

### Start Backend (Terminal 1)

```bash
conda activate agenticai
uvicorn backend.app:app --reload --host 0.0.0.0 --port 8001
```

Expected output:
```
INFO:     Uvicorn running on http://0.0.0.0:8001
INFO:     Application startup complete.
```

Or use the Windows helper script:
```powershell
.\scripts\start_backend.ps1
```

### Start Frontend (Terminal 2)

```bash
cd frontend
npm run dev
```

Expected output:
```
VITE v5.x.x  ready in XXXms
➜  Local:   http://localhost:5173/
```

Or use the Windows helper script:
```powershell
.\scripts\start_frontend.ps1
```

### Use the App

Open **http://localhost:5173**, enter a story prompt, and click **Generate**.

---

## Docker Deployment

The recommended way to run this project in production. Docker uses **edge-tts** (cross-platform) instead of Windows SAPI5 voices.

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows/Mac) or Docker Engine (Linux)
- A free [Groq API key](https://console.groq.com)

### Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/ShaheerZamanShah/VisualStoryGenerator.git
cd VisualStoryGenerator

# 2. Configure environment
cp .env.example .env
# Edit .env and set GROQ_API_KEY=your_key

# 3. Build and start
docker compose up --build -d

# 4. Open the app
# http://localhost:8080
```

### Architecture (Docker)

```
Browser → http://localhost:8080
              │
              ▼
    ┌─────────────────────┐
    │  frontend (nginx)   │  serves React UI
    │  proxies /api /ws   │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │  backend (FastAPI)  │  port 8001 (internal)
    │  edge-tts + rembg   │
    │  + MoviePy + ffmpeg │
    └─────────────────────┘
               │
               ▼
         Docker volume: app-data
         (data/outputs + state_versions)
```

### Docker Commands

```bash
# Start in background
docker compose up -d --build

# View logs
docker compose logs -f

# Stop
docker compose down

# Rebuild after code changes
docker compose up --build -d

# Remove volumes (clears generated videos)
docker compose down -v
```

### Docker Files

| File | Purpose |
|------|---------|
| `Dockerfile` | Python backend with ffmpeg, fonts, rembg |
| `frontend/Dockerfile` | Multi-stage React build + nginx |
| `docker-compose.yml` | Orchestrates backend + frontend |
| `nginx/nginx.conf` | Proxies `/api`, `/ws`, `/docs` to backend |
| `.dockerignore` | Excludes venv, node_modules, generated data |

### Production Notes

- Frontend is served on **port 8080** (mapped to nginx port 80)
- API docs available at **http://localhost:8080/docs**
- Generated videos persist in the `app-data` Docker volume across restarts
- Set `FRONTEND_ORIGIN` in `.env` to your public URL when deploying to a VPS
- First video generation downloads the rembg model (~170 MB) — allow extra time

### Deploy to a VPS (DigitalOcean, AWS, etc.)

```bash
# On your server
git clone https://github.com/ShaheerZamanShah/VisualStoryGenerator.git
cd VisualStoryGenerator
cp .env.example .env
nano .env   # set GROQ_API_KEY and FRONTEND_ORIGIN=http://your-server-ip:8080

docker compose up -d --build
```

Open `http://your-server-ip:8080` in your browser.

---

## Render Deployment

Deploy to [Render](https://render.com) using the included **`render.yaml`** blueprint. This creates two services:

| Service | Type | URL |
|---------|------|-----|
| `visualstory-backend` | Docker Web Service | `https://visualstory-backend.onrender.com` |
| `visualstory-frontend` | Static Site | `https://visualstory-frontend.onrender.com` |

The frontend is built with `VITE_API_BASE` pointing at the backend automatically. WebSockets connect directly to the backend over `wss://`.

### Step-by-step

1. **Push this repo to GitHub** (already done if you cloned from GitHub)

2. **Create a Render account** at https://render.com

3. **New → Blueprint** → connect `ShaheerZamanShah/VisualStoryGenerator`

4. Render reads `render.yaml` and creates both services

5. When prompted, set **`GROQ_API_KEY`** (mark as secret)

6. Click **Apply** and wait for both services to deploy (~5–10 min first build)

7. Open your frontend URL: `https://visualstory-frontend.onrender.com`

### Manual setup (without Blueprint)

If you prefer creating services manually:

**Backend (Web Service → Docker)**
- Repository: `ShaheerZamanShah/VisualStoryGenerator`
- Dockerfile path: `./Dockerfile`
- Plan: **Starter** ($7/mo) recommended
- Health check path: `/health`
- Environment variables:
  - `GROQ_API_KEY` = your key
  - `TTS_ENGINE` = `edge_tts`
  - `FRONTEND_ORIGIN` = `https://your-frontend-name.onrender.com`
- Add a **Persistent Disk** (1 GB, mount at `/app/data`) on Starter+

**Frontend (Static Site)**
- Root directory: `frontend`
- Build command: `npm install && npm run build`
- Publish directory: `dist`
- Environment variable:
  - `VITE_API_BASE` = `https://your-backend-name.onrender.com`

### Render architecture

```
Browser → https://visualstory-frontend.onrender.com
              │  (React static site)
              │  API calls + WebSocket
              ▼
         https://visualstory-backend.onrender.com
              │  FastAPI + edge-tts + rembg + MoviePy
              ▼
         Persistent disk: /app/data  (Starter plan+)
```

### Plan recommendations

| Plan | RAM | Notes |
|------|-----|-------|
| **Free** | 512 MB | OK for testing UI; video generation may OOM or timeout. No persistent disk. |
| **Starter** | 512 MB | Persistent disk included. Minimum for keeping generated videos. |
| **Standard** | 2 GB | **Recommended** for reliable video pipeline (rembg + MoviePy) |

### Render troubleshooting

| Problem | Solution |
|---------|----------|
| Build fails on backend | Check Render logs; first build downloads rembg model (~170 MB) |
| CORS errors | Set `FRONTEND_ORIGIN` to exact frontend URL (with `https://`) |
| WebSocket not connecting | Ensure `VITE_API_BASE` is the backend URL (not frontend) |
| Service times out generating video | Upgrade to Standard plan (2 GB RAM) |
| Videos disappear after restart | Add persistent disk on `/app/data` (Starter+) |
| Backend sleeps on Free tier | First request after idle takes ~30s to wake up |

---

## Free Deployment (Full Quality)

**Honest answer:** Render's free tier (512 MB RAM) cannot reliably run the full video pipeline at perfect quality — video AI needs ~1–2 GB RAM. Cloud-mode optimizations help but are a compromise (960×540, lighter background removal).

For **$0/month AND the complete pipeline working perfectly**, use one of these:

### Option 1 — Oracle Cloud Always Free VM (recommended)

Oracle gives you a **free ARM VM forever** with up to **4 CPUs and 24 GB RAM** — enough to run the full pipeline at 1280×720 with no compromises.

1. Sign up at https://www.oracle.com/cloud/free/
2. Create a VM: **Ubuntu 22.04**, shape **VM.Standard.A1.Flex** (4 OCPU, 24 GB RAM)
3. Open inbound port **8080** in Oracle VCN security rules + Ubuntu firewall
4. SSH into the VM and run:

```bash
# Install Docker
sudo apt update && sudo apt install -y docker.io docker-compose-v2 git
sudo usermod -aG docker $USER
newgrp docker

# Deploy
git clone https://github.com/ShaheerZamanShah/VisualStoryGenerator.git
cd VisualStoryGenerator
cp .env.example .env
nano .env          # set GROQ_API_KEY=your_key
docker compose up -d --build
```

5. Open `http://YOUR_VM_PUBLIC_IP:8080`

> Do **not** set `CLOUD_MODE=1` on Oracle — you have enough RAM for full quality (1280×720, rembg, subtitles).

| | Render Free | Oracle Free VM |
|--|-------------|----------------|
| Cost | $0 | $0 |
| RAM | 512 MB | up to 24 GB |
| Full 1280×720 video | Unreliable | Yes |
| Always on | No (sleeps) | Yes |
| Setup effort | Easy | ~30 min one-time |

### Option 2 — Cloudflare Tunnel + your PC (demo / portfolio)

Run the pipeline on your Windows machine (where it already works perfectly) and expose it with a free public URL:

```bash
# Install cloudflared: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/
cloudflared tunnel --url http://localhost:8080
```

Use `docker compose up` locally, then share the Cloudflare URL in your CV. **Free and perfect quality**, but only works while your PC is on.

### Option 3 — Render Free (compromise)

Keep Render free for a **live portfolio URL**, accepting:
- 960×540 cloud-mode output (not full HD)
- Lighter background removal (not rembg)
- May still fail on very long stories
- Backend sleeps after 15 min idle

Set `CLOUD_MODE=1` (already in `render.yaml`).

### Summary — pick based on your goal

| Goal | Best free option |
|------|------------------|
| CV link that always works perfectly | **Oracle Cloud VM** |
| Quick demo while you're at your PC | **Cloudflare Tunnel + local Docker** |
| Easiest setup, OK with lower quality | **Render Free** |
| Paid but zero setup, reliable | **Render Standard ($25/mo)** |

---

## Running Tests

```bash
conda activate agenticai

# Unit tests — no API key needed (~5 seconds)
python -m pytest tests/unit/ -v

# Edit agent unit tests — no API key needed (~3 seconds)
python -m pytest agents/edit_agent/tests/ -v

# Integration tests — requires GROQ_API_KEY (~60 seconds)
python -m pytest tests/integration/ -v

# All tests
python -m pytest tests/ agents/edit_agent/tests/ -v
```

| Test Suite | Count | API Key Required |
|-----------|-------|-----------------|
| Schema validation | 28 | No |
| VideoAgent math/geometry | 17 | No |
| StateManager snapshot/undo | 2 | No |
| Edit agent unit | 29 | No |
| Edit agent integration (live Groq) | 21 | Yes |

### Edit Agent CLI Demo

```bash
python scripts/demo_edit_agent.py
```

Runs 11 scripted edits through the live Groq LLM and prints intent classification, confidence, params, and undo stack depth for each step.

---

## Edit Agent

### Supported Intent Types

| Intent | Example Query | What Changes |
|--------|--------------|-------------|
| `character_visuals` | "Change Alice's hair to bright red" | Character sprite regenerated |
| `background_visuals` | "Make scene 2 a rainy night city" | Scene background regenerated |
| `audio_emotion` | "Make Sam's voice sound angry" | TTS emotion/rate adjusted |
| `script_dialogue` | "Change scene 1 dialogue to 'We need to leave'" | Story spec dialogue updated |
| `speed` | "Make the speech rate slower" | TTS rate reduced |
| `scene_duration` | "Make scene 3 about 20 seconds" | Scene duration updated |
| `subtitle` | "Add bold white subtitles at the bottom" | Subtitle flag set |
| `music` | "Add dramatic orchestral background music" | Music override stored |
| `regenerate` | "Regenerate the entire video" | Full pipeline re-run |
| `undo` | "undo" | Reverts last edit + restores assets |
| `unknown` | Unrecognized requests | No state change, returns note |

### Classification Pipeline

```
User query (string)
    ↓
EditAgent.classify() — Groq LLM + ClassifiedIntent schema
    ↓
EditIntent { intent, target, confidence, params }
    ↓
EditExecutor.execute() — mutates job_state.json
    ↓
StateManager.snapshot() — persists v_N.json + asset copy
    ↓
If rerender_required → background Audio + Video rerun
    ↓
WebSocket progress broadcast → frontend updates
```

### Example Edit Session

```
1. "Change Alice's hair colour to bright red"     → character_visuals
2. "Make scene 2 background a rainy night city"   → background_visuals
3. "Make Sam's voice sound angry and intense"     → audio_emotion
4. "Make the speech rate slower"                  → speed
5. "undo"                                         → reverts edit 4
6. "undo"                                         → reverts edit 3
```

---

## Generated Output Files

After a successful generation, artifacts are saved to `data/outputs/{job_id}/`:

```
data/outputs/{job_id}/
├── job_state.json                    # Job metadata + edit history
├── story_spec.json                   # Generated story structure
├── audio/
│   ├── scene1_00_Alice.wav           # Per-line TTS audio
│   ├── scene1_01_Bob.wav
│   ├── master_dialogue.wav           # All lines concatenated
│   └── timing_manifest.json          # Millisecond-accurate timing per line
└── video/
    ├── char_Alice_raw.png            # Raw generated character sprite
    ├── char_Alice.png                # Background-removed transparent sprite
    ├── char_Bob_raw.png
    ├── char_Bob.png
    ├── scene1_bg.png                 # Generated scene backgrounds
    ├── scene2_bg.png
    ├── mouth_overlay.png             # Lip-sync mouth ellipse overlay
    └── final_output.mp4              # Final rendered video
```

Edit version history at `data/state_versions/{job_id}/`:

```
data/state_versions/{job_id}/
├── history.json                      # Ordered version list
├── v_1.json                          # State after edit 1
├── assets_v_1/                       # Full asset snapshot at edit 1
├── v_2.json
└── assets_v_2/
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GROQ_API_KEY` | **Yes** | — | Groq API key (free at console.groq.com) |
| `GROQ_MODEL` | No | `llama-3.3-70b-versatile` | Groq model for story + edit classification |
| `FRONTEND_ORIGIN` | No | `http://localhost:8080` | CORS allowed origin |
| `TTS_ENGINE` | No | `edge_tts` | TTS engine: `edge_tts` (Docker/Linux) or `pyttsx3` (Windows) |
| `VITE_API_BASE` | No | `http://localhost:8001` | Frontend → backend URL (local dev only; empty in Docker) |
| `DATA_ROOT` | No | `data` | Root directory for outputs and state |
| `BACKEND_HOST` | No | `0.0.0.0` | Backend bind host |
| `BACKEND_PORT` | No | `8001` | Backend port |

> Pollinations.ai image generation requires **no API key**.

---

## Example Prompts

The story agent enforces exactly 2 characters and 50–70 second runtime. Prompts with clear settings, conflict, and two distinct characters work best.

**Mystery:**
> A journalist meets a mysterious informant in a foggy alley at midnight. The informant claims to have proof of a government cover-up, but demands something in return.

**Drama:**
> Two childhood friends reunite at a cozy café after ten years. One has big news, the other is afraid to hear it.

**Sci-fi:**
> Two astronauts on a damaged space station argue about whether to abandon the mission or stay and fix it.

**Fantasy:**
> A wizard apprentice and their master argue in a candlelit tower library about using forbidden magic to save a village.

**Slice of life:**
> Two coworkers share lunch on a rooftop during sunset. One is about to quit; the other tries to convince them to stay.

**What makes a good prompt:**
- Two distinct characters with roles or personalities
- A clear visual setting (rainy street, café, space station, forest)
- Emotional tension or conflict to drive dialogue
- Visual atmosphere keywords for better image generation

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `GroqError: The api_key client option must be set` | Set `GROQ_API_KEY` in `.env` |
| `rembg` slow on first run | Normal — downloads ~170 MB u2net model once |
| `ffmpeg` not found | Install ffmpeg and add to system PATH |
| Port 8001 already in use | `netstat -ano \| findstr 8001` then `taskkill /PID <pid> /F` |
| Frontend CORS errors | Ensure backend is on 8001 and `FRONTEND_ORIGIN=http://localhost:5173` in `.env` |
| No TTS audio / silent video | Docker: ensure `TTS_ENGINE=edge_tts`. Windows local: uses SAPI5 voices |
| Docker build fails on rembg | Ensure Docker has enough disk space (~2 GB free) |
| `docker compose` can't reach backend | Run `docker compose logs backend` and check `GROQ_API_KEY` is set |
| `UnicodeEncodeError` in terminal | Windows codepage issue — demo script handles with UTF-8 reconfigure |
| Video not appearing after generation | Wait for WebSocket `done` event; asset may take a moment to write |

---

## License

MIT
