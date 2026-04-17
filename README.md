# ai-life-skills

Skills for Claude Code that write notes into an Obsidian vault.

## Skills

- [`summarize/`](./summarize) — summarize a YouTube video, article, PDF, EPUB, or podcast into a clean note in `summaries/`. Proportional summary length, closes with a `## Further Reading` list.
- [`summarize-call/`](./summarize-call) — transcribe a call recording with speaker labels, summarize it, and write a call note + transcript into `meetings/`. Auto-detects `mlx_whisper` on Apple Silicon or `faster-whisper` on Windows/Linux; or use ElevenLabs Scribe.

## Install

```bash
git clone https://github.com/lampo808/ai-life-skills-simplified ~/src/ai-life-skills
ln -s ~/src/ai-life-skills/summarize ~/.claude/skills/summarize
ln -s ~/src/ai-life-skills/summarize-call ~/.claude/skills/summarize-call
```

Restart Claude Code, then use `/summarize` or `/summarize-call`.

## Vault structure

```
your vault/
├── daily/YYYY/         # daily notes, named MM-DD-YY ddd.md
├── meetings/           # call notes + transcripts
├── summaries/          # content summary notes (flat)
├── references/         # concept / company / product stubs
└── research/           # longer-form research (subfolders per topic)
```

Skills auto-detect the vault root by walking up from `$PWD` looking for `.obsidian/`. Override with `export VAULT_ROOT="/path/to/vault"`.

## Requirements

- `summarize`: `yt-dlp`, `defuddle`, `pdftotext`, `pandoc`
- `summarize-call`: `ffmpeg`, plus `mlx_whisper` (Apple Silicon) or `faster-whisper` (Windows/Linux) + `pyannote.audio`, or `ELEVENLABS_API_KEY`

Each skill checks what's missing on first run and asks before installing anything.

## License

MIT.
