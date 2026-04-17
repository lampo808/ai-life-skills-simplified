# ai-life-skills

A collection of skills I use with Claude Code to improve my life in various ways. Designed to pair with an AI-managed Obsidian vault — the skills read from and write to the vault.

## Skills

- [`summarize/`](./summarize) — drop in a YouTube video, article, PDF, EPUB, or podcast and it writes a clean summary note into your vault. Summary length is proportional to source length. Closes with a `## Further Reading` list of people and references mentioned.
- [`summarize-call/`](./summarize-call) — drop in a call recording (video or audio) and it transcribes with speaker labels, summarizes, and writes a call note + transcript. No person notes, no wikilinks — just the essentials.

(more coming) (daily brief, daily news)

## Install — easy mode

Open Claude Code in any directory and paste this:

> Install the ai-life-skills pack from https://github.com/reysu/ai-life-skills. Clone the repo to `~/src/ai-life-skills`, ask me where I want the new Obsidian vault to live, create the vault folder with the full folder structure the skills expect, and symlink every skill in the repo into `~/.claude/skills/`.

Claude will:
1. Clone the repo
2. Ask where to put the vault (default: `~/ai-vault`)
3. Create the vault folder with the expected structure (see below)
4. Symlink `summarize` and `summarize-call` into `~/.claude/skills/`

Restart Claude Code so it picks up the new skills. Then run `/summarize` or `/summarize-call`.

> **Recommended: use a new, dedicated Obsidian vault** for these skills rather than your existing personal vault. The skills create and modify many notes/folders automatically, and keeping it separate avoids polluting notes you've written yourself.

### If you already have an Obsidian vault

You can point the skills at an existing vault if you want — tell Claude the path instead of creating a new one and it'll only create any missing folders. Just note the recommendation above about a dedicated vault.

## Install — individual skill only

If you just want one skill and already have a vault:

```bash
mkdir -p ~/src
git clone https://github.com/reysu/ai-life-skills ~/src/ai-life-skills
ln -s ~/src/ai-life-skills/summarize ~/.claude/skills/summarize
# or:
ln -s ~/src/ai-life-skills/summarize-call ~/.claude/skills/summarize-call
```

## Usage

### Summarize a video, article, PDF, or book

```
/summarize https://youtube.com/watch?v=...
```

Summary length is proportional to source length — a 10-minute video gets a short summary, a 3-hour podcast gets a long one, a 600-page book gets chapter-by-chapter treatment.

For a book or PDF, drop the file into your vault first (I use `_Attachments/`) then call the skill with the filename:

```
/summarize The Singularity Is Near.epub
```

### Summarize a call

Drop the recording anywhere and call the skill with the path:

```
/summarize-call ~/Downloads/call-with-alex.mp4
```

It'll ask whether to transcribe locally (free, private, slower — auto-detects `mlx_whisper` on Apple Silicon or `faster-whisper` on Windows/Linux) or with ElevenLabs Scribe (paid, faster, one API call). Then it writes a call note and the full transcript.

## Vault structure

```
your vault/
├── daily/YYYY/         # daily notes, named MM-DD-YY ddd.md
├── meetings/           # call notes + transcripts
├── summaries/          # content summary notes (flat, no subfolders)
├── references/         # concept / company / product stubs
├── research/           # longer-form research (subfolders per topic)
└── _Attachments/       # drop ebooks/PDFs here before telling claude to summarize them
```

Easy-mode install creates this structure for you. If you're using an existing vault, the skills prompt before creating any missing folders on first run. You can also rename any of them in the Configuration block at the top of each SKILL.md.

If you run Claude Code from outside your vault, set `VAULT_ROOT`:

```bash
export VAULT_ROOT="/path/to/vault"
```

Otherwise the skills walk up from your current directory looking for `.obsidian/`.

## Requirements

- `summarize` uses `yt-dlp`, `defuddle`, `pdftotext`, and `pandoc`
- `summarize-call` uses `ffmpeg` (always), plus either `mlx_whisper` (Apple Silicon) or `faster-whisper` (Windows/Linux) + `pyannote.audio`, or an `ELEVENLABS_API_KEY` (cloud path)

Each skill checks what's missing on first run and asks before installing anything.

## Tested on

- macOS 15 (Darwin 25) on Apple Silicon, Python 3.11+, Claude Code CLI
- Windows 11, Python 3.11+, faster-whisper on CPU
- ElevenLabs Scribe path works on any OS with Python + `requests`

## License

MIT.
