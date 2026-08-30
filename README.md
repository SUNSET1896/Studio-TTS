# Studio TTS

**Text-to-speech, voice cloning, voice design, and speech-to-text — built for Studio worlds.**

One command turns text into anyone's voice, or audio/video into timestamped text, and drops the
result into your world as a playable, publishable audio work.

Studio TTS was made with reference to [tts-skill](https://github.com/boommanpro/tts-skill)
(MIT, built on OmniVoice + faster-whisper) — a heartfelt thank-you to its author for sharing
the project. It keeps the CLI generation/transcription core, drops
the web server and frontend, adds **multi-engine support** (Qwen3-TTS / OmniVoice / Spark-TTS /
F5-TTS / CosyVoice 2 / IndexTTS2 / Fish Speech) with isolated virtual environments, and lands
outputs in `works/audio/<slug>/` as publishable Studio Works.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Command Reference](#command-reference)
- [Voice Generation Modes](#voice-generation-modes)
- [Multi-Engine Support](#multi-engine-support)
- [Reproducibility](#reproducibility)
- [Output Layout](#output-layout)
- [Publishing as a Studio Work](#publishing-as-a-studio-work)
- [World-Adaptive Workflow](#world-adaptive-workflow)
- [Transcription](#transcription)
- [Model Download Sources & Networking](#model-download-sources--networking)
- [Dependency Manifests](#dependency-manifests)
- [Voice Design Reference](#voice-design-reference)
- [Troubleshooting](#troubleshooting)
- [Project Layout](#project-layout)
- [License & Credits](#license--credits)

---

## Overview

Studio TTS is a **functional methodology**, not a product: it contains no world-specific content.
Text, voice attributes, and reference audio are always supplied by the caller, so the same pipeline
works for any character, any language, any material.

The skill is agent-driven: the host agent (e.g. in Neta Studio) reads the skill, runs the CLI, and
delivers the results. Everything the CLI produces lives under the workspace's `works/audio/`
directory, so audio works are ordinary world assets that can be published like any other Studio Work.

### What it can do

| Capability | Description |
|---|---|
| **Text-to-speech** | Synthesize speech from text (`auto` mode picks a voice for you) |
| **Voice design** | Create a voice from attributes — gender, age, pitch, accent, dialect (`design` mode) |
| **Voice cloning** | Clone a voice from a 3–10 s reference clip (`clone` mode) |
| **Batch exploration** | Same text, multiple seeds, pick the best one (seeds are reproducible) |
| **Transcription** | Convert audio/video to text with word-level timestamps (faster-whisper) |
| **World adaptation** | Read the world, ask which line/scene/narration to voice, bind voices back to characters |

---

## Key Features

- **One entry point** — `studio_tts.py` with subcommands: `setup`, `voice`, `transcribe`,
  `models`, `world`, `bind`, `history`, `doctor`.
- **Self-bootstrapping environment** — a private venv (`scripts/.venv`) is created and reused on
  first run; the system Python is never polluted (PEP 668 safe).
- **Install-time initialization** — `install.sh` activates the main environment and runs `setup`
  once, so the skill is ready immediately after installation (idempotent; re-runnable).
- **7 engines, isolated venvs** — each external engine runs in its own venv
  (`.venv-spark`, `.venv-qwen`, `.venv-f5`), so dependency conflicts are impossible
  (e.g. `qwen-tts` downgrades `transformers` and would break OmniVoice — it never touches it).
- **3 generation modes** — `auto` / `design` / `clone`.
- **Fully reproducible** — fixed `--seed` + identical parameters = identical audio. Every
  generation is recorded in a SQLite history with its exact reproducible command.
- **World-aware workflow** — `world` inventories the world's characters/places/events and existing
  voices; `bind` writes a character's voice back into their material so future lines reuse the
  same voice.
- **Multi-source model downloads** — HuggingFace direct, ModelScope (魔搭, China-friendly), or
  hf-mirror (`HF_ENDPOINT`), all honored automatically.
- **Transcription with word timestamps** — JSON + TXT output; video handled via bundled ffmpeg.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Host agent (Neta Studio / Codex / agentskills-compatible)      │
│    reads SKILL.md → runs CLI → publishes results                │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                    python3 scripts/studio_tts.py <command>
                                │
        ┌───────────────────────┼───────────────────────────────┐
        │                       │                               │
   Main venv (.venv)      External engine venvs            Pure-stdlib commands
   OmniVoice engine       .venv-spark  (Spark-TTS)          world   (read-only world scan)
   + faster-whisper ASR   .venv-qwen   (Qwen3-TTS)          bind    (writes via neta CLI)
                          .venv-f5     (F5-TTS)
                                │
                    works/audio/<slug>/
                    ├── audio/*.wav + *.mp3
                    ├── index.html        (player page)
                    └── tracks.json       (machine-readable manifest)
                                │
                    node /mods/neta/neta work init/publish web  →  Studio Work
```

### Data flow

1. **Plan** — read the world (world-adaptive workflow), settle text / mode / engine (`--model`) /
   seed / batch count; design-mode attributes from `references/voice-design.md`, engines compared
   in `references/models.md`.
2. **Generate** — `voice`; same seed + same parameters = same audio; batches use independent seeds.
   External engines run in their own venvs via backend runners (`scripts/backends/`), never
   touching the main environment.
3. **Land** — WAV (original) + MP3 (web playback) written to `works/audio/<slug>/audio/`; a player
   page `index.html` lists every track with its reproducible command in a collapsible block;
   `tracks.json` serves programmatic consumers (AVG dubbing, character-card voices, …) and carries
   the `engine` field.
4. **Publish** — `neta work init web` + `neta work publish web` turns the folder into a Studio Work.
5. **Deliver** — hand over the Work link or file path, together with the reproducible command.
6. **Transcribe** — independent path: `transcribe` outputs JSON/TXT directly.

### Reproducibility contract (most important invariant)

- Fixed `--seed` + all generation parameters = identical audio.
- Every generation is written to a SQLite history; the reproducible command is always recoverable
  (player page / `history --show_cmd`).
- When you give a character a voice, record the `instruct` (or reference audio) **and** the seed in
  the character's material — that is what makes the voice stable and reusable.

---

## Requirements

| Item | Requirement |
|---|---|
| Python | ≥ 3.10 (3.11/3.12 recommended) |
| Disk | ~2–4 GB for the main environment + model weights (see per-engine sizes) |
| Network | HuggingFace or ModelScope reachable (China-friendly mirrors supported, see [Networking](#model-download-sources--networking)) |
| Hardware | **CPU works out of the box** for OmniVoice, Spark-TTS, Qwen3-TTS (slow but functional); GPU recommended for CosyVoice 2 / IndexTTS2 / Fish Speech and for faster Qwen3-TTS |
| System | Linux / macOS / Windows; `git` for source-based engine installs |

---

## Installation

### Method 1 — copy to a skills directory (recommended)

```bash
# REPO scope (project/world, shared by the team)
cp -r studio-tts /workspace/.agents/skills/

# USER scope (personal, across projects)
cp -r studio-tts ~/.agents/skills/

# ADMIN scope (machine-level, needs permissions)
cp -r studio-tts /etc/codex/skills/
```

### Method 2 — unzip

```bash
unzip studio-tts-package.zip -d /workspace/.agents/skills/
```

### Post-install initialization (ready-to-use)

After copying, **activate the main environment and run `setup` once** (first run ≈ 10–20 min):

```bash
cd <skills-dir>/studio-tts
bash install.sh --with_asr    # recommended: includes speech-to-text (faster-whisper + ffmpeg)
# TTS-only main environment: bash install.sh
```

`install.sh` creates/activates `scripts/.venv` and installs torch + OmniVoice (+ optional ASR).
It is **idempotent** — if the environment is already ready it exits immediately, so it can be
re-run any time. Pass `--force` to force a full reinstall.

If you skip `install.sh`, the CLI auto-installs on first use (same 10–20 min), so installation is
never a hard blocker.

Verify the environment:

```bash
python3 scripts/studio_tts.py doctor         # environment diagnostics
python3 scripts/studio_tts.py models status  # per-engine install state
```

---

## Quick Start

```bash
# 0. Read the world (world-adaptive workflow; no TTS environment needed)
python3 scripts/studio_tts.py world

# 1. Initialize the main environment (skipped automatically if install.sh already ran)
python3 scripts/studio_tts.py setup --with_asr

# 2. Generate a voice — design mode (attributes) — default engine qwen3-tts
python3 scripts/studio_tts.py voice \
  --text "Hello, welcome." \
  --mode design --instruct "warm, gentle female voice" \
  --seed 1 --slug hello

# CPU-friendly engines are one flag away
python3 scripts/studio_tts.py voice --model omnivoice --text "Hello, welcome."
python3 scripts/studio_tts.py voice --model spark-tts  --text "Hello, welcome." \
  --mode design --instruct "male, low pitch"

# 3. Publish as a Studio Work
node /mods/neta/neta work init web works/audio/hello --title "Hello"
node /mods/neta/neta work publish web works/audio/hello

# 4. Bind the voice back to a character material (reuse the same voice later)
python3 scripts/studio_tts.py bind --slug hello --character "<character name>"

# 5. Transcribe a clip
python3 scripts/studio_tts.py transcribe --input meeting.mp4 --language zh
```

---

## Command Reference

### `setup`

Initialize the main environment (torch + OmniVoice + optional ASR).

| Flag | Description |
|---|---|
| `--with_asr` | Also install speech-to-text (faster-whisper + imageio-ffmpeg) |
| `--force` | Force full reinstall, ignoring the "already set up" marker |

### `voice` — generate speech

| Parameter | Description | Default |
|---|---|---|
| `--text` | Text to synthesize (**required**) | — |
| `--mode` | `auto` (automatic voice) / `design` (attribute-based) / `clone` (reference audio) | `auto` |
| `--ref_audio` | Reference audio path (required for `clone`) | — |
| `--ref_text` | Transcript of the reference audio (clone; auto-transcribed if empty) | — |
| `--instruct` | Voice attribute description for `design` (e.g. `"male, middle-aged, low pitch"`) | — |
| `--language` | Language (e.g. `Chinese`, `en`); auto-detected if empty | auto |
| `--seed` | Random seed; same seed + same parameters = same audio | random |
| `--batch_count` | Batch size (1–5, each with an independent seed) | 1 |
| `--num_step` | Diffusion decoding steps (lower = faster) | 32 |
| `--speed` | Speaking rate (>1 is faster) | 1.0 |
| `--duration` | Fixed duration in seconds (overrides `speed` when set) | — |
| `--guidance_scale` | Guidance scale | 2.0 |
| `--t_shift` | Time-step shift | 0.1 |
| `--denoise` | Denoise tokens | `true` |
| `--postprocess_output` | Output post-processing | `true` |
| `--normalize_text` | Text normalization (numbers/symbols read aloud) | `false` |
| `--model` | Engine ID: `qwen3-tts` (default) / `omnivoice` / `spark-tts` / `f5-tts` / `cosyvoice2` / `index-tts2` / `fish-speech`; for omnivoice you may also pass an HF repo name | `qwen3-tts` |
| `--device` | Inference device (cuda/mps/xpu/cpu, auto-detected) | auto |
| `--slug` | Output folder name → `works/audio/<slug>/` | `voice_<timestamp>` |
| `--title` | Work title | record name |
| `--desc` | Work description | auto summary |
| `--name` | History record name | auto |

### `transcribe` — audio/video to text

| Parameter | Description | Default |
|---|---|---|
| `--input` | Input audio/video file (**required**) | — |
| `--model` | Whisper model: tiny/base/small/medium/large-v3 | `large-v3` |
| `--language` | Language code (zh/en/ja…); auto-detected if empty | auto |
| `--task` | `transcribe` (as-is) / `translate` (to English) | `transcribe` |
| `--device` | cuda/cpu (faster-whisper has no MPS; macOS falls back to cpu) | auto |
| `--compute_type` | `float16` / `int8` | cuda→`float16`, cpu→`int8` |
| `--beam_size` | Beam-search size | 5 |
| `--word_timestamps` | Word-level timestamps | `true` |
| `--vad_filter` | VAD silence filtering | `true` |
| `--output_dir` | Output directory | `works/audio/transcribe_<timestamp>/` |
| `--name` | Record name | auto |

Output: JSON (word timestamps + segment metadata) + TXT (plain text). Video input is handled via
the bundled ffmpeg.

### `models` — engine registry

```bash
python3 studio_tts.py models                        # list all engines
python3 studio_tts.py models status                 # install state per engine
python3 studio_tts.py models install <ID>           # create venv + install deps + download weights
python3 studio_tts.py models install <ID> --source modelscope   # download from ModelScope
```

`models install` performs: create venv → install torch (CPU index) → install dependencies from the
engine's `requirements-<id>.txt` → download weights. Manual engines (CosyVoice 2 / IndexTTS2 /
Fish Speech) print exact manual steps instead.

### `world` — world voice inventory (no TTS environment needed)

Reads `manifest.json`, all materials (characters/locations/events/lore), and existing audio works;
prints a report of the world and every voice binding state. `--json` emits the structured report.

```bash
python3 scripts/studio_tts.py world          # readable report
python3 scripts/studio_tts.py world --json   # structured JSON
```

### `bind` — write a character's voice back into their material

Writes the voice binding into the character material's `content` key `声线` (via
`neta atom update`, merged by key so other facts are untouched). Future generations can then reuse
the exact same voice.

```bash
# From an existing audio work (reads engine/mode/instruct/seed from tracks.json)
python3 scripts/studio_tts.py bind --slug <slug> --character <name>
# Or specify explicitly
python3 scripts/studio_tts.py bind --character <name> --engine spark-tts --mode design \
  --instruct "male, low pitch" --seed 7
# Preview without writing
python3 scripts/studio_tts.py bind --slug <slug> --character <name> --dry-run
```

Binding string format: `engine=<ID>; mode=<auto|design|clone>; instruct="…"; seed=<N>`
(`clone` uses `ref_audio="<path>"` instead of `instruct`).

### `history` / `doctor`

```bash
python3 scripts/studio_tts.py history [--type tts|asr|all] [--show_cmd]
python3 scripts/studio_tts.py doctor
```

- `history` — browse the SQLite generation history; `--show_cmd` prints reproducible commands.
- `doctor` — environment diagnostics (Python, torch, device, network, model cache).

---

## Voice Generation Modes

### `auto` — fastest validation

Give text only; the model picks a voice automatically.

```bash
python3 scripts/studio_tts.py voice --text "Hello, welcome." --mode auto
```

### `design` — create a voice from attributes

Describe the voice with natural-language attributes (Chinese or English, comma-separated,
3–8 attributes recommended):

```bash
python3 scripts/studio_tts.py voice \
  --text "Hello, welcome." \
  --mode design --instruct "male, middle-aged, low pitch" \
  --seed 42 --slug demo
```

Attribute categories are documented in [Voice Design Reference](#voice-design-reference) and
`references/voice-design.md`. To give a character a stable voice, record the same `--instruct`
and `--seed` in their material (see [World-Adaptive Workflow](#world-adaptive-workflow)).

### `clone` — clone a reference voice

Provide a 3–10 second reference clip; the model replicates that voice speaking new text.

```bash
python3 scripts/studio_tts.py voice \
  --text "Hello, this is a cloned voice." \
  --mode clone --ref_audio ref.wav --ref_text "Transcript of ref.wav" \
  --seed 42 --slug clone-demo
```

If `--ref_text` is omitted it is auto-transcribed (requires the ASR deps). Clone requires a
reference audio — with none, use `design` instead.

---

## Multi-Engine Support

`voice --model <ID>` selects the engine. The default is **`qwen3-tts`** (all public variants,
no token). Every engine runs in its own venv; dependency conflicts are impossible.

| ID | Model | License | Install | Hardware | Verified |
|---|---|---|---|---|---|
| `omnivoice` | OmniVoice | Apache-2.0 | with `setup` | CPU/GPU | ✅ |
| `spark-tts` | Spark-TTS 0.5B | Apache-2.0 | `models install spark-tts` | CPU-friendly | ✅ |
| `qwen3-tts` | **Qwen3-TTS 0.6B/1.7B (default)** | Apache-2.0 | `models install qwen3-tts` | GPU recommended | weights verified |
| `f5-tts` | F5-TTS 0.3B | MIT | `models install f5-tts` | CPU/GPU | — |
| `cosyvoice2` | CosyVoice 2 0.5B | Apache-2.0 | manual (conda env) | GPU ~4 GB | documented |
| `index-tts2` | IndexTTS2 1.5B | Apache-2.0 | manual (uv env) | GPU ~8 GB | documented |
| `fish-speech` | Fish Speech 1.5 | Apache-2.0 | manual (conda 3.12) | GPU ≥4 GB | documented |

### Engine comparison

| Capability | omnivoice | spark-tts | qwen3-tts | f5-tts | cosyvoice2 | index-tts2 | fish-speech |
|---|---|---|---|---|---|---|---|
| Design (attributes) | ✅ all | gender/pitch/speed | ✅ natural language | ❌ | ✅ NLC | ❌ | partial |
| Zero-shot clone | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Emotion control | ❌ | ❌ | partial | partial | ✅ | ✅ 7 types | ✅ |
| Chinese dialects | ✅ | ❌ | ❌ | ❌ | ✅ (Cantonese/Sichuan…) | ❌ | ❌ |
| Runs on CPU | ✅ slow | ✅ fast | slow | slow | ❌ | ❌ | ❌ |
| Environment | built-in | isolated venv | isolated venv | isolated venv | conda | uv | conda 3.12 |

Selection guidance (based on the *2026 Open-Source TTS Selection Guide*): **CosyVoice 2** for
Chinese-first and engineering maturity, **Qwen3-TTS** for low latency, **IndexTTS2** for emotion
control, **Spark-TTS** when you have no GPU.

### Per-engine notes

- **omnivoice** — installed with `setup`; weights `k2-fsa/OmniVoice` (~5 GB, auto-downloaded to the
  HF cache); 600+ languages; three modes; CPU-ready.
- **spark-tts** — `models install spark-tts` clones the source (or uses the vendored
  `models/Spark-TTS-src`) and downloads `SparkAudio/Spark-TTS-0.5B` (~3.7 GB: BiCodec + LLM +
  wav2vec2). Design supports gender/pitch/speed only; missing attributes default to
  male/moderate/moderate. Bilingual zh/en.
- **qwen3-tts (default)** — `models install qwen3-tts`; weights are **public (no token)**:
  - design → `Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice` (speaker + natural-language tone for design)
  - clone → `Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice`
  - auto → `Qwen/Qwen3-TTS-12Hz-0.6B-Base`
  - The `0.6B-VoiceDesign` variant is **restricted** (login + token on HF and ModelScope) and has
    been removed from the default mapping.
  - ~1.5 GB per variant, downloaded on first use. Note: speed adjustment is not supported; missing
    `sox` only prints a warning.
- **f5-tts** — `models install f5-tts`; pure zero-shot clone (MIT, most permissive); weights
  `SWivid/F5-TTS` (~1.4 GB, auto-downloaded on first use). Clone mode only — reference audio
  required.
- **cosyvoice2** — weights are public on ModelScope (`iic/CosyVoice2-0.5B`, ~2 GB, verified
  anonymous clone+LFS); engine requires a manual conda environment (`python=3.10`,
  `pip install -r requirements.txt`, `pynini` via conda-forge). GPU ~4 GB.
- **index-tts2** — weights public on ModelScope (`IndexTeam/IndexTTS-2`, ~3 GB, verified); engine
  requires `uv run webui.py` (uv-managed deps). GPU FP16 ~7.8 GB. 7 emotions + token-level
  duration control.
- **fish-speech** — weights public (`fishaudio/fish-speech-1.5`, ~1 GB); engine requires a manual
  conda env with Python **3.12** (3.13/3.14 break dependencies), `pip install torch==2.8.0
  torchaudio==2.8.0 --index-url https://download.pytorch.org/whl/cu126`, checkout `v1.5.1`.

> **All 7 engines' weights can be downloaded anonymously** (verified 2026-08) — none requires
> login. "Downloadable" ≠ "runnable": CosyVoice 2 / IndexTTS2 / Fish Speech still need their
> manual conda/uv environments + GPU; the CPU sandbox can run OmniVoice, Spark-TTS and Qwen3-TTS
> directly.

---

## Reproducibility

- All engines accept `--seed`; the same seed + the same parameters reproduce the same audio
  (samplers differ per engine, so quality varies).
- `tracks.json` and the player page carry reproducible commands that always include `--model <ID>`
  — switching engines is just a flag change.
- Changing devices (cuda ↔ cpu) may change results; external engines use their own venv's torch.

---

## Output Layout

```
works/audio/<slug>/
├── audio/
│   ├── <engine>_<timestamp>_seed<N>.wav      # original generation
│   └── <engine>_<timestamp>_seed<N>.mp3      # web-playback copy
├── index.html                                 # player page (no external deps)
└── tracks.json                                # machine-readable manifest
```

`tracks.json` schema (excerpt):

```json
{
  "work_dir": "/workspace/works/audio/demo",
  "title": "Demo",
  "mode": "design",
  "text": "Hello, welcome.",
  "seed": 42,
  "instruct": "male, middle-aged, low pitch",
  "engine": "Spark-TTS",
  "command": "python3 …/studio_tts.py voice --text … --model spark-tts --slug demo",
  "tracks": [ { "name": "Demo · 1", "file": "audio/…_seed42.mp3", "seed": 42 } ]
}
```

Transcription output: `works/audio/transcribe_<timestamp>/` → `.json` (word timestamps + segments)
+ `.txt`.

History: `scripts/tts_skill.db` (SQLite — TTS/ASR records + reproducible commands).

---

## Publishing as a Studio Work

```bash
# 1. Generate (writes the folder + player page under works/audio/<slug>/)
python3 scripts/studio_tts.py voice --text "…" --slug my-voice --title "…"

# 2. Initialize the Work (creates meta.json)
node /mods/neta/neta work init web works/audio/my-voice --title "…"

# 3. Publish (registers the Work, generates a preview URL)
node /mods/neta/neta work publish web works/audio/my-voice
```

- `init` and `generate` can be swapped; `init` must simply precede `publish`.
- Transcription output is not published by default — hand over the JSON/TXT paths directly; create
  a page only if the user asks.

---

## World-Adaptive Workflow

The skill defaults to a **world-aware flow** in Studio worlds: read the world, ask which
part to voice, generate, and bind the voice back so it becomes a reusable world asset. The full
methodology lives in `references/world-voice.md`; the two CLI commands are `world` and `bind`.

1. **Read the world** — read `manifest.json` and every material (characters/locations/events/lore)
   with full facts; run `studio_tts.py world` for the voice inventory.
2. **Sort the story** — order events/scenes into a mainline, note branches, and list every
   voiceable segment (character lines, scene narration, mainline/branch narration).
3. **Inventory voices** — who already has a voice (material `content` key `声线`/`voice`), who has a
   work but no binding (`world` marks them "待绑定"), and whether a narrator archive exists.
4. **Ask the user** — always ask: which part / which character / which segment's narration. Use
   `neta questionnaire` with options built from the world's real names, or ask in chat. If the user
   names a character but no text, draft the line from the material and confirm before generating.
5. **Generate** — reuse the bound voice when it exists (engine + mode + instruct/ref + seed); design
   from the material or clone from a reference clip otherwise; narrations are designed from the
   world's tone (e.g. a calm low male voice for a road-trip story).
6. **Bind back** — write the character's voice into their material (`bind` subcommand →
   `content` key `声线`) and archive narrator voices in `works/audio/narrator/binding.json`, so the
   same voice is reused automatically next time.

Narrator archive shape (`works/audio/narrator/binding.json`):

```json
{
  "engine": "spark-tts",
  "mode": "design",
  "instruct": "male, middle-aged, low pitch, calm",
  "seed": 11,
  "desc": "World narrator voice (designed from the world's tone)",
  "updatedAt": "2026-08-29T00:00:00"
}
```

The skill itself is **world-agnostic**: all examples are generic placeholders, and every
world-specific value is read at runtime.

---

## Transcription

```bash
python3 scripts/studio_tts.py transcribe --input meeting.mp4 --language zh
```

- Uses faster-whisper (CTranslate2) — fast on CPU, no torch dependency for ASR.
- Word-level timestamps + segment metadata in JSON; plain text in TXT.
- `--task translate` converts to English.
- Video inputs are handled via the bundled ffmpeg (`imageio-ffmpeg`), no system ffmpeg needed.
- Model default `large-v3`; choose `tiny/base/small/medium` for speed.

---

## Model Download Sources & Networking

### Sources (three, in order of preference)

| Source | When to use | How |
|---|---|---|
| **ModelScope (魔搭)** | China networks — recommended | `models install <ID> --source modelscope`, or `export MODELSCOPE_HUB=1` |
| **HuggingFace** | Direct/reliable access | default |
| **hf-mirror** | China + HF-only repos | `export HF_ENDPOINT=https://hf-mirror.com` |

Verified ModelScope repositories (all engines, 2026-08):

| Engine | ModelScope repo |
|---|---|
| omnivoice | `k2-fsa/OmniVoice` |
| spark-tts | `SparkAudio/Spark-TTS-0.5B` |
| qwen3-tts | `Qwen/Qwen3-TTS-12Hz-0.6B-{CustomVoice,Base}` (VoiceDesign restricted) |
| f5-tts | `SWivid/F5-TTS` |
| cosyvoice2 | `iic/CosyVoice2-0.5B` (official `iic` org; clone+LFS verified) |
| index-tts2 | `IndexTeam/IndexTTS-2` (with hyphen; clone+LFS verified) |
| fish-speech | `fishaudio/fish-speech-1.5` |
| ASR | `Systran/faster-whisper-large-v3` (and tiny~medium) |

### Environment variables

| Variable | Effect |
|---|---|
| `HF_ENDPOINT` | Use hf-mirror (or any HF endpoint) for downloads; unset → direct huggingface.co |
| `MODELSCOPE_HUB=1` | Default all downloads to ModelScope |
| `HF_TOKEN` | For restricted repos (only Qwen `0.6B-VoiceDesign` remains restricted; removed from defaults) |
| `MODELSCOPE_API_TOKEN` | Token for ModelScope restricted repos |
| `WORKSPACE_DIR` | Override the workspace root (otherwise auto-detected via `manifest.json`) |
| `NETA_CLI` | Override the `neta` CLI path (default `/mods/neta/neta`) |

> Mirror note: hf-mirror only accelerates **public** repos — it does not bypass restricted-repo
> authorization. Overseas/direct environments may see 308 redirects back to the origin; direct
> access is faster there — prefer ModelScope or direct HF.

---

## Dependency Manifests

Each environment has its own requirements file; `models install` installs from the file
(falling back to the built-in registry list if missing). torch/torchaudio are installed
separately by the installer (CPU index in sandboxes), never listed here, to avoid CUDA/CPU
conflicts.

| File | Environment | Contents |
|---|---|---|
| `scripts/requirements.txt` | Main `.venv` (OmniVoice + ASR + ModelScope) | `omnivoice` / `soundfile` / `numpy` / `transformers>=5.16` (locked) / `accelerate>=1.14` / `modelscope>=1.39` / `faster-whisper` / `imageio-ffmpeg` / `num2words` |
| `scripts/requirements-spark.txt` | `.venv-spark` | `einops` / `einx` / `omegaconf` / `safetensors` / `soxr` / `tqdm` / `soundfile` / `numpy` / `transformers==4.46.2` |
| `scripts/requirements-qwen.txt` | `.venv-qwen` | `qwen-tts==0.1.1` / `transformers==4.57.3` (locked) / `accelerate==1.12.0` / `librosa` / `soundfile` / `onnxruntime` / `einops` / `sox` |
| `scripts/requirements-f5.txt` | `.venv-f5` | `f5-tts` |

---

## Voice Design Reference

`--mode design --instruct "<attributes>"` — natural-language attributes, comma-separated,
Chinese or English, 3–8 recommended. Categories:

| Category | Options |
|---|---|
| Gender | male / female (男 / 女) |
| Age | child / teenager / young adult / middle-aged / elderly |
| Pitch | very low / low / moderate / high / very high |
| Style | whisper |
| English accents | American / British / Australian / Canadian / Indian / Chinese / Japanese / Korean / Portuguese / Russian |
| Chinese dialects | 河南话 陕西话 四川话 贵州话 云南话 桂林话 济南话 石家庄话 甘肃话 宁夏话 青岛话 东北话 |

Examples:

```bash
--instruct "male, middle-aged, low pitch"        # steady older male (elder, shopkeeper)
--instruct "female, teenager, high pitch"        # bright young female (protagonist, little sister)
--instruct "male, middle-aged, whisper"          # hushed mystery NPC
--instruct "male, young adult, british accent"   # foreign visitor
--instruct "男, 中年, 东北话"                     # Northeastern Chinese dialect, middle-aged male
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| First run of any command | Auto-creates the venv and installs (10–20 min), or run `bash install.sh` first |
| `voice` reports model download failure | Mirror configured? Retry once; `doctor` shows device/network state |
| `transcribe` reports missing faster-whisper | Run `setup --with_asr` (the wrapper auto-triggers it) |
| All generations fail | Check text length (≤ 5000 chars) and that the reference audio exists; `history` shows the error |
| Slow generation | Normal on CPU — a short line (≤ 100 chars) takes ~1–3 min per track |
| Reproducibility drift | `--seed` must be fixed **together with all parameters**; changing device (cuda ↔ cpu) changes results |
| Engine missing | `models status` shows what's missing; `models install <ID>` sets it up; manual engines print exact steps |
| `bind` can't find the character | Run `world` to list character names; use `--atom <id>` for exact targeting |
| pip install inside venv breaks | The wrapper strips injected `PIP_USER`; keep using the wrapper |

---

## Project Layout

```
studio-tts/
├── SKILL.md                       # skill definition (when to invoke, workflow, checklist)
├── install.sh                     # post-install init: activate main env + run setup once
├── README.md                      # this document
├── references/
│   ├── params.md                  # voice/transcribe full parameter reference
│   ├── voice-design.md            # design-mode attribute categories & combinations
│   ├── models.md                  # engine registry, download methods, selection guide
│   ├── studio-integration.md      # output layout, publishing, delivery, error handling
│   └── world-voice.md             # world-adaptive workflow (read/plan/ask/generate/bind)
├── models/
│   └── Spark-TTS-src/             # vendored Spark-TTS source (offline install support)
└── scripts/
    ├── studio_tts.py              # single entry point (setup/voice/transcribe/models/world/bind/history/doctor)
    ├── backends/                  # external-engine runners (run_spark / run_qwen / run_f5)
    ├── requirements*.txt          # per-environment dependency manifests
    └── tts_skill/                 # vendored upstream package (slimmed: no web server)
```

---

## License & Credits

- **This skill** was made with reference to [tts-skill](https://github.com/boommanpro/tts-skill)
  (MIT) — many thanks to its author for making the project open. It is slimmed to CLI-only and
  adapted for Studio worlds. Upstream fixes carried in the vendored code: pip-mirror `--index-url`/`-i` conflicts, PEP 668 isolation, `PIP_USER`
  injection, CPU float16 unavailability, hf-mirror redirect failures (real network probing +
  `HF_ENDPOINT` respected), reproducible commands rewritten to Studio-native `studio_tts.py`
  form.
- **Engines** (each under its own license): OmniVoice (k2-fsa, Apache-2.0), Spark-TTS
  (SparkAudio, Apache-2.0), Qwen3-TTS (Qwen, Apache-2.0), F5-TTS (SWivid, MIT), CosyVoice 2
  (iic/FunAudioLLM, Apache-2.0), IndexTTS2 (IndexTeam, Apache-2.0), Fish Speech (fishaudio,
  Apache-2.0); ASR via faster-whisper (Systran, MIT).
- The skill contains **no world-specific content**: text, voice attributes, and reference audio
  are always supplied by the caller.
