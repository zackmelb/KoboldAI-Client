# CLAUDE.md — KoboldAI-Client Codebase Guide

This file provides essential context for AI assistants working on the KoboldAI-Client repository. Read this before making changes.

---

## Project Overview

KoboldAI-Client is a browser-based AI writing assistant with multiple gameplay modes (Novel, Adventure, Chat). It supports local Hugging Face models, TPU backends, and external AI APIs. The server is a monolithic Flask/SocketIO application with a rich frontend and Lua scripting support.

**License:** AGPL v3
**Python target:** 3.8+ (tested with conda environments)
**Main entry point:** `aiserver.py`

---

## Repository Structure

```
KoboldAI-Client/
├── aiserver.py              # Main server — Flask, SocketIO, API, game logic (~10,300 lines)
├── breakmodel.py            # Model layer-splitting for low-VRAM scenarios
├── fileops.py               # File I/O helpers (stories, soft prompts, userscripts)
├── gensettings.py           # Generation settings definitions for the UI
├── logger.py                # Loguru-based logging with custom levels
├── prompt_tuner.py          # Soft prompt loading and management
├── structures.py            # KoboldStoryRegister (auto-increment ordered dict)
├── torch_lazy_loader.py     # Lazy loading for large model weights
├── tpu_mtj_backend.py       # Mesh Transformer JAX / TPU backend
├── utils.py                 # Utilities: debounce, sentence trimming, model download
├── warpers.py               # Custom logits warpers (TFS, Typical, Top-A, RepPen)
├── test_aiserver.py         # PyTest test suite
├── bridge.lua               # Lua ↔ Python bridge (~70KB, Lua 5.4)
├── requirements.txt         # Core pip dependencies
├── requirements_mtj.txt     # TPU/MTJ-specific dependencies
├── customsettings_template.json  # CLI args configuration template
├── templates/
│   ├── index.html           # Main web UI (Bootstrap-based)
│   └── swagger-ui.html      # API docs page
├── static/
│   ├── application.js       # Frontend logic (~3,900 lines)
│   ├── custom.css           # UI styling
│   └── ...                  # Bootstrap, jQuery, Socket.IO, icons
├── userscripts/             # Lua userscripts and examples
│   └── api_documentation.md # Userscript API reference
├── colab/                   # Google Colab notebooks (TPU & GPU)
├── models/                  # Local model storage (not committed)
├── stories/                 # Saved story JSON files
├── maps/                    # Token maps for model families
├── environments/            # Conda YAML definitions
└── docker-*/                # Docker deployment scripts
```

---

## Architecture

### Backend (Python)

The entire server lives in **`aiserver.py`**. It is large (~10,300 lines) and intentionally monolithic. Key responsibilities:

- **Web server** — Flask with Flask-SocketIO (eventlet) for real-time communication
- **Global state** — `vars` object holds all runtime state: model info, settings, story chunks, flags
- **Game loop** — `actionsubmit()` → `generate()` → SocketIO push to frontend
- **AI backends** — local HuggingFace models, OpenAI, GooseAI, InferKit, KoboldAI Horde
- **REST API v1** — endpoints under `/api/v1/` with OpenAPI/Swagger docs at `/api`
- **Lua scripting** — Lupa embeds Lua 5.4; `bridge.lua` exposes Python state to scripts
- **World Info** — `checkworldinfo()` activates entries based on keywords in context
- **Soft prompts** — trainable embeddings managed via `prompt_tuner.py` + mkultra

### Frontend (JavaScript)

**`static/application.js`** is the entire client:
- SocketIO connection/event handling
- Story text rendering in a `contenteditable` div
- Settings UI auto-generated from `gensettings.py` definitions
- Adventure mode, chat mode, novel mode branching logic

**`templates/index.html`** provides the shell — Bootstrap layout, menus, buttons.

### Lua Scripting Layer

User scripts run inside the Python process via **Lupa** (Lua 5.4). The `bridge.lua` file exposes the Python `vars` object and utility functions to scripts. Scripts can hook into generation events to modify prompts, inject text, and manipulate logits.

---

## Key Classes & Data Structures

| Name | File | Purpose |
|------|------|---------|
| `vars` | `aiserver.py` | Global state singleton (all runtime data) |
| `KoboldStoryRegister` | `structures.py` | Ordered dict with auto-increment IDs for story chunks |
| `TokenStreamQueue` | `aiserver.py` | Queue for streaming token output via SocketIO |
| `Send_to_socketio` | `aiserver.py` | Redirects tqdm progress bars to WebSocket |
| `KoboldSchema` | `aiserver.py` | Marshmallow schema base for API validation |

---

## Key Functions

| Function | File | Description |
|----------|------|-------------|
| `general_startup()` | `aiserver.py` | App initialization, CLI arg parsing |
| `load_model()` | `aiserver.py` | Load any supported AI model/backend |
| `actionsubmit()` | `aiserver.py` | Handle user action, build prompt, call generate |
| `generate()` | `aiserver.py` | Core HuggingFace generation with custom samplers |
| `sendtoapi()` | `aiserver.py` | Dispatch to external API backends |
| `tpumtjgenerate()` | `aiserver.py` | TPU-specific generation path |
| `checkworldinfo()` | `aiserver.py` | Scan context for World Info keyword matches |
| `post_generate()` | `aiserver.py` | REST API generation endpoint handler |

---

## Custom Sampling (warpers.py)

KoboldAI implements several sampling strategies on top of HuggingFace Transformers:

- **TailFreeLogitsWarper** — Tail-free sampling (TFS)
- **TypicalLogitsWarper** — Typical decoding
- **TopALogitsWarper** — Top-A sampling
- **AdvancedRepetitionPenaltyLogitsProcessor** — Enhanced repetition penalty

When modifying generation behavior, changes to these warpers affect all local model inference.

---

## API Endpoints (v1)

All under `/api/v1/`. OpenAPI spec available at `/api`.

| Endpoint | Methods | Purpose |
|----------|---------|---------|
| `/info/version` | GET | API version info |
| `/generate` | POST | Text generation |
| `/model` | GET, PUT | Active model info |
| `/story/` | GET, POST, DELETE | Story management |
| `/story/end` | GET, PUT, POST, DELETE | Story end chunk |
| `/story/end/num_actions` | GET | Action count |
| `/world_info/` | GET, POST | World info CRUD |
| `/memory` | GET, PUT | Memory text |
| `/authors_note` | GET, PUT | Author's note text |
| `/config/...` | GET, PUT | Generation config |

---

## Dependencies

### Core (requirements.txt)

```
transformers==4.24.0
Flask==2.2.3
Flask-SocketIO==5.3.2
torch>=1.9,<1.13
huggingface_hub==0.12.1
lupa==1.10           # Lua 5.4 embedding
accelerate
eventlet
marshmallow
loguru
safetensors
mkultra              # Soft prompt library
```

### Frontend (CDN / static)

- jQuery 3.6.0
- Bootstrap (JS + CSS)
- Socket.IO (client)
- jQuery UI Sortable
- Rangy (text selection)
- Open Iconic (icons)

### Optional

- `requirements_mtj.txt` — JAX, Mesh Transformer for TPU support

---

## Development Workflow

### Running Locally

```bash
# Standard launch (installs deps if missing)
./play.sh           # Linux/macOS
play.bat            # Windows

# Manual launch
python aiserver.py

# With a specific model
python aiserver.py --model path/to/model

# Remote access
python aiserver.py --host 0.0.0.0
```

### Installing Dependencies

```bash
./install_requirements.sh   # Linux
install_requirements.bat    # Windows
```

### Running Tests

```bash
pytest test_aiserver.py
```

Tests use a Flask test client with SocketIO. They cover model loading, story CRUD, and generation. Note: some tests require actual model files.

### Docker

```bash
./docker-cuda.sh     # NVIDIA GPU
./docker-rocm.sh     # AMD GPU
```

---

## Configuration

**CLI arguments** (`customsettings_template.json` documents all flags):
- `--model` — model path or name
- `--host` — bind address (default `127.0.0.1`)
- `--port` — port (default `5000`)
- `--remote` — enable remote access
- `--nobreakmodel` — disable layer splitting

**In-app settings** are stored in the `vars` object and persisted to story JSON files.

---

## Code Conventions

### Python

- The codebase uses a **flat, functional style** — no class hierarchy beyond the data containers.
- `vars` is a global module-level object; functions read/write it directly rather than passing state.
- SocketIO events are the primary communication channel; REST API is a secondary interface.
- Logging uses **loguru** with custom levels: `GENERATION`, `PROMPT`, `INIT`, `MESSAGE`. Use `logger.info()`, `logger.debug()`, etc.
- Error handling typically logs and falls back gracefully rather than raising.
- String formatting uses f-strings throughout.

### JavaScript

- jQuery-centric; avoid adding modern framework dependencies.
- SocketIO events are handled with `socket.on(...)` listeners.
- UI elements are manipulated via jQuery selectors (`$('#id')`).
- The story text lives in a `contenteditable` div; manipulate it through the provided helper functions, not raw DOM writes.

### Lua Userscripts

- Scripts run in a sandboxed Lua 5.4 environment inside the Python process.
- Use `bridge.lua` APIs only — do not attempt to import arbitrary Python modules.
- Scripts hook into `kobold.callback_*` functions to intercept generation events.
- See `userscripts/api_documentation.md` for the full API reference.

---

## Things to Avoid

- **Do not refactor `aiserver.py` into multiple files** — the monolithic structure is intentional and many external tools depend on it being a single file.
- **Do not upgrade pinned dependencies** without testing — `transformers==4.24.0` and `torch<1.13` are version-locked for compatibility with model architectures.
- **Do not add new global state** outside the `vars` object.
- **Do not add async/await patterns** — the codebase uses eventlet for coroutines; mixing asyncio will break things.
- **Do not modify `bridge.lua`** without understanding the full Lua ↔ Python binding; bugs here affect all userscripts.
- **Do not add inline script tags** to `index.html` — keep JS in `application.js`.

---

## Common Patterns

### Adding a New Generation Setting

1. Add entry to the settings dict in `gensettings.py`
2. Add corresponding attribute to `vars` in `aiserver.py`
3. Wire it into the sampler pipeline in `generate()` or the relevant warper
4. The frontend auto-generates UI from `gensettings.py` — no HTML changes needed

### Adding a New API Endpoint

1. Define a Marshmallow schema class inheriting `KoboldSchema`
2. Create the route handler function with `@app.route('/api/v1/...')`
3. Register it in the APISpec in `general_startup()`
4. Use existing endpoint handlers as patterns — see `post_generate()` for POST, story endpoints for CRUD

### Adding a Lua Script Hook

1. Create a new `.lua` file in `userscripts/`
2. Implement the `kobold.callback_*` function(s) you need
3. Refer to `userscripts/api_documentation.md` for available hooks and the `vars` bindings

---

## Model Support

KoboldAI supports:
- GPT-Neo, GPT-J, GPT-NeoX family
- OPT (Meta)
- BLOOM
- XGLM
- Any HuggingFace CausalLM-compatible model

Token maps in `maps/` define special token handling per model family. When adding a new model family, add a corresponding JSON map.

---

## File Format Notes

- **Stories** are saved as JSON (`stories/*.json`) containing story chunks, settings, memory, world info, and author's note.
- **Soft prompts** are `.zip` files containing embedding tensors.
- **Model weights** live in `models/` and are never committed to git.
