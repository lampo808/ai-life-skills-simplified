# ai-life-skills

This vault will be a collection of skills I use with Claude Code to help improve my life in various ways. For best results I would recommend pairing this with an AI managed oObsidian vault. Right now I'm uploading just the summarize skill first and will update with the full vault setup soon. 


## Skills

- [`summarize/`](./summarize) — drop in a YouTube video, article, PDF, EPUB, or podcast and it writes a summary note into your vault with wikilinks to every person and concept mentioned

(more coming) (summarize-meeting, daily brief, daily news)

## Install

Claude Code loads skills from `~/.claude/skills/<name>/`. Clone this repo and symlink the skills you want:

```bash
git clone https://github.com/<you>/ai-life-skills ~/src/ai-life-skills
ln -s ~/src/ai-life-skills/summarize ~/.claude/skills/summarize
```
Then use `/summarize` in Claude Code.

## Usage

For a YouTube video or article, paste the URL:

```
/summarize https://youtube.com/watch?v=...
```

It will summarize the content proportional to the length of the content unless specified otherwise. The purpose of this is that you are able to still get a granular view of the content itself without just generating a generic paragraph summary. Summaries of books will be broken down into chapters, so for instance you can read a 600 page book in about 20 minutes. There will also be quotes directly from the page and link to the direct transcript or file in the frontmatter. 

For a book or PDF, drop the file into your vault first (anywhere works, I usually put it in `_Attachments/`), then call the skill with the filename:

```
/summarize The Singularity Is Near.epub
```

The skill finds the file, extracts the text, and writes the summary note into `08 Summaries/`.

## Vault structure

My vault generally looks like this:
(note this is completely separate from my personal vault)
```
your vault/
├── 01 Updates/
├── 02 Daily/YYYY/MM/   # daily notes, named MM-DD-YY ddd.md
├── 03 Meetings/
├── 04 People/
├── 05 Projects/
├── 06 Research/
├── 07 References/
├── 08 Summaries/
├── _Templates/
├── _Attachments/       # where I put ebooks,pdfs, before telling claude to summarize them 
└── _Bases/             # optional, only if you use Obsidian Bases
```

If you don't have these folders, the summarize skill asks you before creating them on first run. You can also rename them in the Configuration block at the top of `summarize/SKILL.md`.

If you run Claude Code from outside your vault, set `VAULT_ROOT`:

```bash
export VAULT_ROOT="/path/to/vault"
```

Otherwise the skill walks up from your current directory looking for `.obsidian/`.

## Requirements

The summarize skill uses `yt-dlp`, `defuddle`, `pdftotext`, and `pandoc`. It'll check what's missing on first run and ask before installing.

## License

MIT.
