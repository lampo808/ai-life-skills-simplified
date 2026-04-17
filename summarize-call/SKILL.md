---
name: summarize-call
description: Transcribe a call recording with speaker diarization, summarize it, and create Obsidian vault notes (call note + transcript). Works with video or audio files. Supports local transcription (auto-detects mlx_whisper on Apple Silicon or faster-whisper elsewhere) or ElevenLabs Scribe.
user_invocable: true
---

# Summarize Call

Takes a call recording (video or audio), transcribes it with speaker labels, summarizes it, and writes structured notes into your Obsidian vault.

## Requirements

**Vault structure** — the skill expects these folders inside your Obsidian vault. Folder names are defaults; override them in the Configuration block below if your vault uses different names.

| Folder | Purpose |
|---|---|
| `meetings/` | Where call notes and transcripts land |
| `daily/YYYY/` | Daily notes, named `MM-DD-YY ddd.md` |

**CLI tools** — install these before first use, or let Step 0 walk you through it:

| Tool | Purpose | Install |
|---|---|---|
| `ffmpeg` | Extract audio from video files | macOS: `brew install ffmpeg` · Linux: `apt install ffmpeg` or `dnf install ffmpeg` · Windows: ffmpeg.org/download.html |
| `faster-whisper` (non-Apple) | Local transcription on Windows/Linux | `pip install faster-whisper` |
| `mlx_whisper` (Apple Silicon) | Local transcription on M-series Macs | `pip install mlx-whisper` |
| `pyannote.audio` | Speaker diarization | see Step 3 walkthrough |

**Alternative to local**: set `ELEVENLABS_API_KEY` to use ElevenLabs Scribe for transcription + diarization in one call (paid, faster, no setup).

## Configuration

The skill reads these variables at runtime. Override any of them via environment variables, or edit the defaults here:

```
VAULT_ROOT    = $VAULT_ROOT   # auto-detected if not set (see Step 0a)
MEETINGS_DIR  = meetings
DAILY_DIR     = daily
```

All paths below are relative to `$VAULT_ROOT`.

## Trigger

When the user provides a call recording (MP4, MOV, WAV, MP3, M4A, etc.) and wants it transcribed, summarized, and documented.

## Inputs

- **Recording**: file path to the recording
- **Participants**: names of the people on the call
- **Date/time**: extract from filename if possible, otherwise ask
- **Speaker count**: default 2, ask if ambiguous

## Step 0: Bootstrap check (first run)

Before doing any work, verify the environment is ready. **Skip any check that already passes** — only prompt the user when something is actually missing.

### 0a. Resolve the vault root

```bash
vault=""
if [ -n "$VAULT_ROOT" ]; then
  vault="$VAULT_ROOT"
else
  dir="$PWD"
  while [ "$dir" != "/" ]; do
    if [ -d "$dir/.obsidian" ]; then vault="$dir"; break; fi
    dir="$(dirname "$dir")"
  done
fi
echo "Vault: ${vault:-NOT FOUND}"
```

If no vault is found, ask the user:

> **What's the absolute path to your Obsidian vault?**
> Recommended: use a **new, dedicated Obsidian vault** for this skill — not your existing personal vault. The skill creates and modifies many notes and folders, and a clean vault avoids polluting your existing notes. If you don't have one yet, create an empty folder, open it in Obsidian (File -> Open vault as folder), and paste that path here.

After they answer, validate that `<answer>/.obsidian/` exists before using it — if not, warn that the path doesn't look like an Obsidian vault (they may need to open it in Obsidian first) and ask them to confirm or re-enter. Use the validated answer as `$VAULT_ROOT` for the session (and suggest they set it permanently in their shell profile).

### 0b. Check required folders

```bash
for d in "$MEETINGS_DIR" "$DAILY_DIR"; do
  [ -d "$VAULT_ROOT/$d" ] || echo "MISSING: $d"
done
```

For each missing folder, ask the user: **"Create `<folder>` in your vault? [y/N]"** — if yes, `mkdir -p "$VAULT_ROOT/<folder>"`.

### 0c. Check required CLI tools

```bash
command -v ffmpeg >/dev/null 2>&1 || echo "MISSING: ffmpeg"
```

If ffmpeg is missing, ask the user before installing. Install command depends on the platform:
- macOS: `brew install ffmpeg`
- Linux (Debian/Ubuntu): `sudo apt install ffmpeg`
- Linux (Fedora/RHEL): `sudo dnf install ffmpeg`
- Windows: download from https://ffmpeg.org/download.html

## Step 1: Choose transcription method

Ask the user:

> **Transcription method?**
> 1. **Local** (free, private, slower, requires setup)
> 2. **ElevenLabs Scribe** (paid, faster, handles transcription + diarization in one call)

### Option A: Local

**Auto-detect transcription engine:**
```bash
arch=$(uname -m)
if [ "$arch" = "arm64" ]; then
  engine="mlx_whisper"
else
  engine="faster-whisper"
fi
echo "Engine: $engine"
```

- **Apple Silicon (arm64)**: use `mlx_whisper` — optimized for M-series GPUs
- **Windows / Linux / Intel Mac**: use `faster-whisper` — runs on CPU or CUDA

Check for the detected engine:
```bash
command -v "$engine" >/dev/null 2>&1 || echo "MISSING: $engine"
```

If missing, ask before installing:
- mlx_whisper: `pip install mlx-whisper`
- faster-whisper: `pip install faster-whisper`

**Platform notes (Windows):**
If running on Windows and you see `OMP: Error #15: Initializing libiomp5md.dll`, set before running:
```bash
export KMP_DUPLICATE_LIB_OK=TRUE
```
This is a runtime workaround for a known OpenMP conflict on Windows — do not set it permanently in your profile unless needed.

If `python` is not on PATH, use the full path to your Python installation (e.g. `C:/Users/<you>/miniconda3/python.exe`).

**pyannote.audio environment:**

Check for the venv at `~/.local/share/summarize-call/pyannote-env` (persists across reboots, XDG-compliant):
```bash
PYANNOTE_ENV="${XDG_DATA_HOME:-$HOME/.local/share}/summarize-call/pyannote-env"
[ -d "$PYANNOTE_ENV" ] || echo "MISSING: pyannote venv"
```

If missing, walk through setup:
```bash
mkdir -p "$(dirname "$PYANNOTE_ENV")"
uv venv "$PYANNOTE_ENV"
source "$PYANNOTE_ENV/bin/activate"
uv pip install pyannote.audio torch torchaudio
```

**HuggingFace token:**

Check for `HF_TOKEN`:
```bash
[ -n "$HF_TOKEN" ] || echo "MISSING: HF_TOKEN"
```

If missing, tell the user:
> You need a HuggingFace token with access to the pyannote gated repos.
> 1. Create a token at https://huggingface.co/settings/tokens (choose "Read" scope)
> 2. Accept the terms for all three repos while logged in:
>    - https://huggingface.co/pyannote/speaker-diarization-3.1
>    - https://huggingface.co/pyannote/segmentation-3.0
>    - https://huggingface.co/pyannote/speaker-diarization-community-1
> 3. Export for this session: `export HF_TOKEN="hf_..."`
> 4. To persist, add that line to your `~/.zshrc` (or `~/.bashrc`)

Wait for the user to confirm before continuing.

### Option B: ElevenLabs Scribe

Check for the API key:
```bash
[ -n "$ELEVENLABS_API_KEY" ] || echo "MISSING: ELEVENLABS_API_KEY"
```

If missing, tell the user:
> You need an ElevenLabs API key.
> 1. Grab one at https://elevenlabs.io/app/settings/api-keys
> 2. Export for this session: `export ELEVENLABS_API_KEY="..."`
> 3. To persist, add that line to your `~/.zshrc` (or `~/.bashrc`)

Wait for the user to confirm before continuing.

**Before calling Scribe**, always print the recording duration and a pricing heads-up so the user can confirm:

> This recording is `<HH:MM:SS>` (`<minutes>` min). Check ElevenLabs pricing at https://elevenlabs.io/pricing for the current per-minute rate on the Scribe model. Continue? [y/N]

## Step 2: Extract audio

```bash
ffmpeg -v quiet -i "<input>" -vn -acodec pcm_s16le -ar 16000 -ac 1 /tmp/<name>.wav -y
```

## Step 3: Transcribe + diarize

### Option A: Local

**Transcribe with auto-detected engine:**

mlx_whisper (Apple Silicon):
```bash
mlx_whisper --model mlx-community/whisper-large-v3-turbo --language en \
  --output-dir /tmp --output-format json \
  --condition-on-previous-text False /tmp/<name>.wav
```

faster-whisper (Windows / Linux):
```python
from faster_whisper import WhisperModel

model = WhisperModel("large-v3", device="cpu", compute_type="int8")
segments, info = model.transcribe("/tmp/<name>.wav", language="en",
                                   condition_on_previous_text=False)
# save segments to JSON matching the mlx_whisper output format
```

Notes:
- Use `--language en` / `language="en"` for English calls; for Japanese use `ja`
- `--condition-on-previous-text False` / `condition_on_previous_text=False` prevents hallucination loops
- Start processing partial results while transcription is still running — don't block

**Diarize with pyannote:**
```python
from pyannote.audio import Pipeline
import torch, os

pipeline = Pipeline.from_pretrained(
    "pyannote/speaker-diarization-3.1",
    use_auth_token=os.environ["HF_TOKEN"]
)
device = "cuda" if torch.cuda.is_available() else ("mps" if torch.backends.mps.is_available() else "cpu")
pipeline.to(torch.device(device))

output = pipeline("/tmp/<name>.wav", num_speakers=<N>)
annotation = output.speaker_diarization
for turn, _, speaker in annotation.itertracks(yield_label=True):
    # save turn.start, turn.end, speaker
```

Note: use `output.speaker_diarization.itertracks(yield_label=True)` (not `output.itertracks`).

**Merge transcript + diarization:**
- Map whisper segments to speaker turns by matching timestamps
- Merge consecutive same-speaker segments into paragraphs
- Format: `[H:MM:SS] **Speaker**: text`

### Option B: ElevenLabs Scribe

```python
import requests, os

url = "https://api.elevenlabs.io/v1/speech-to-text"
headers = {"xi-api-key": os.environ["ELEVENLABS_API_KEY"]}

with open("/tmp/<name>.wav", "rb") as f:
    response = requests.post(
        url,
        headers=headers,
        files={"file": f},
        data={
            "model_id": "scribe_v1",
            "language_code": "<lang>",  # e.g. "eng", "jpn"
            "diarize": "true",
            "timestamps_granularity": "word",
            "num_speakers": <N>
        }
    )

result = response.json()
```

Scribe handles both transcription AND diarization in one call — no pyannote needed. Format the result the same way: `[H:MM:SS] **Speaker**: text`.

## Step 4: Summarize

- Use Sonnet for all calls
- For long transcripts (>3000 words), split into chunks and summarize each in parallel, then combine
- Extract: key topics, decisions, action items

## Step 5: Create vault notes

### Transcript file
- Location: `$MEETINGS_DIR/<MM-DD-YY Day Participant1 x Participant2> Transcript.md`
- Content: the merged, speaker-labeled transcript with timestamps
- Frontmatter:
  ```yaml
  ---
  date: YYYY-MM-DD
  duration: <seconds>
  meeting: "<Call Note Title>"
  ---
  ```

### Call note
- Location: `$MEETINGS_DIR/<MM-DD-YY Day Participant1 x Participant2>.md`
- Frontmatter:
  ```yaml
  ---
  created: YYYY-MM-DDTHH:MM
  date: YYYY-MM-DD
  tags: [call]
  participants: ["Name 1", "Name 2"]
  summary: "1-line description of call topics"
  ---
  ```
- No `# Title` heading — filename is the title
- Body structure:
  ```markdown
  > [!tldr]
  > [2-3 sentence overview]

  ## Key Topics
  - ...

  ## Decisions
  - ...

  ## Action Items
  - [ ] ...

  ## Further Reading
  - Person name -- brief context
  - Book or resource -- brief context
  ```
- `summary` frontmatter field is **mandatory** — never omit it
- No wikilinks

### Daily note
- Update `$VAULT_ROOT/$DAILY_DIR/YYYY/MM-DD-YY ddd.md` (create `YYYY/` if missing)
- No `# Title` heading — filename is the title
- Add under a `## calls/meetings` section:
  ```markdown
  - <Call Note Title> -- brief description
  ```

## Key rules

1. **No wikilinks** in call notes or transcripts
2. **No `# Title` headings** — Obsidian shows filename as title
3. **Never repeat frontmatter in body**
4. **`summary` frontmatter is mandatory** on call notes
5. **Always use Sonnet** — never Haiku
6. **Always `--condition-on-previous-text False`** on mlx_whisper / `condition_on_previous_text=False` on faster-whisper to prevent hallucination loops
7. **Auto-detect transcription engine** based on architecture (arm64 -> mlx_whisper, else faster-whisper)
8. **Auto-detect device** for pyannote (CUDA -> MPS -> CPU) so it works on any platform
