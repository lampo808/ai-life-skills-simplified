---
name: summarize
description: Summarize any content (YouTube video, article, whitepaper/PDF, podcast episode, book chapter, etc.) into a clean Obsidian note with section-by-section breakdowns and a Further Reading list. Use when the user provides a URL, file, or content to summarize and document in the vault.
user_invocable: true
---

# Summarize

Universal content summarizer. Takes any input — YouTube video, web article, whitepaper/PDF, epub book, podcast, lecture — and produces a clean Obsidian summary note.

## Requirements

**Vault structure** — the skill expects these folders inside your Obsidian vault. Folder names are the defaults; override them in the Configuration block below if your vault uses different names.

| Folder | Purpose |
|---|---|
| `summaries/` | Where summary notes land (flat — no subfolders) |
| `references/` | Concept / company / product / place stubs (optional) |
| `daily/YYYY/` | Daily notes, named `MM-DD-YY ddd.md` (e.g. `03-29-26 Sun.md`) |

**CLI tools** — install these before first use, or let Step 0 walk you through it:

| Tool | Purpose | Install |
|---|---|---|
| `yt-dlp` | YouTube/podcast download + metadata + subs | `brew install yt-dlp` or `pip install yt-dlp` |
| `defuddle` | Web article extraction | `npm install -g defuddle` |
| `pdftotext` | PDF text extraction | `brew install poppler` |
| `pandoc` | EPUB / DOCX to markdown | `brew install pandoc` |

## Configuration

The skill reads these variables at runtime. Override any of them via environment variables, or edit the defaults here:

```
VAULT_ROOT     = $VAULT_ROOT        # auto-detected if not set (see Step 0a)
SUMMARIES_DIR  = summaries
REFERENCES_DIR = references
DAILY_DIR      = daily
```

All paths below are relative to `$VAULT_ROOT`.

## Trigger

When the user provides content to summarize: a URL (YouTube, article, blog), a PDF/file path, pasted text, a reference to content already in the vault, or a text file containing a list of URLs to batch-summarize.

## Inputs

- **Source**: URL, file path, pasted text, or a `.txt`/`.md` file containing one URL per line
- **Audience** (optional): defaults to "general reader." User may specify (e.g. "high school student", "expert", "5-year-old")
- **`--parallel`** (default): when processing a batch file, launch all summaries as parallel background agents
- **`--serial`**: when processing a batch file, process one URL at a time in order

## Batch mode (file input)

When the argument looks like a local file path (not starting with `http`) and the file exists, treat it as a **batch file**: read each non-empty, non-comment line as a URL and summarize them all.

**Detection:**
- Argument does not start with `http`
- File exists on disk (use Bash `test -f` or Read tool to confirm)
- Lines that start with `#` are treated as comments and skipped

**Execution:**

```
if --serial (or -s):
    for each URL in file:
        run the full summarize workflow (Steps 1-7) for that URL
        wait for completion before proceeding to next

if --parallel (default):
    for each URL in file:
        launch a background Agent (general-purpose) with the full summarize prompt for that URL
    collect results as agents complete
    update daily note once at the end with all entries
```

**Daily note in batch mode:** to avoid write conflicts when running parallel, each agent should **return** its note title and one-line description instead of writing the daily note itself. The orchestrating skill invocation collects all results and writes a single `## content summary` block with all entries.

**Reporting:** after all items complete, report a summary table:
```
| # | Title | Status |
|---|-------|--------|
| 1 | Note Title | done |
| 2 | Note Title | done |
...
```

## Step 0: Bootstrap check (first run)

Before doing any work, verify the environment is ready. **Skip any check that already passes** — only prompt the user when something is actually missing. Do not re-run Step 0 on subsequent invocations if the initial setup succeeded; you can tell it already ran if `$VAULT_ROOT` resolves and the required folders + tools are present.

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
for d in "$SUMMARIES_DIR" "$DAILY_DIR"; do
  [ -d "$VAULT_ROOT/$d" ] || echo "MISSING: $d"
done
```

For each missing folder, ask the user: **"Create `<folder>` in your vault? [y/N]"** — if yes, `mkdir -p "$VAULT_ROOT/<folder>"`.

### 0c. Check required CLI tools

```bash
for tool in yt-dlp defuddle pdftotext pandoc; do
  command -v "$tool" >/dev/null 2>&1 || echo "MISSING: $tool"
done
```

For each missing tool, tell the user what's missing and **ask before installing** — installs touch the user's system. Use the install commands from the Requirements table above. If the user declines, note which tools are missing and warn that the corresponding content types (YouTube, web articles, PDFs, EPUBs) will fail until installed.

## Step 1: Detect content type and extract text

### YouTube video
```bash
# Get metadata
yt-dlp --print "%(id)s|%(title)s|%(duration)s|%(upload_date)s|%(view_count)s|%(channel)s|%(channel_id)s" \
  --no-download "<URL>"

# Try auto-subtitles first (fastest, free)
yt-dlp --write-auto-sub --sub-lang en --sub-format json3 \
  --skip-download -o "/tmp/summarize/%(id)s" "<URL>"
```

**yt-dlp error handling:** if yt-dlp fails, diagnose the error before retrying. Common fixes include adding `--extractor-args`, `--cookies-from-browser`, or other flags. Once you find a fix that works, **save it to memory** (`memory/feedback_ytdlp.md` in the vault) so it is applied automatically in future sessions. Do not hard-code environment-specific flags here.

If auto-subs exist, extract text from the JSON3 file. If not, or if quality is poor, download audio and transcribe locally (ask user: run `yt-dlp` to download, then use `whisper` or `faster-whisper` for transcription).

### Web article / blog post
```bash
defuddle parse "<URL>" --md -o /tmp/summarize/article.md
```

Extract title, author, date, domain from defuddle metadata:
```bash
defuddle parse "<URL>" -p title
defuddle parse "<URL>" -p domain
```

### PDF
```bash
pdftotext "<path>" /tmp/summarize/paper.txt
```

### EPUB (books)
```bash
# Extract full text as markdown (preserves chapter structure)
pandoc "<path>" -t markdown --wrap=none -o /tmp/summarize/book.md

# If you need chapter boundaries, extract the TOC:
pandoc "<path>" -t json | python3 -c "
import json, sys
doc = json.load(sys.stdin)
for block in doc['blocks']:
    if block['t'] == 'Header':
        level = block['c'][0]
        text = ''.join(
            item['c'] if item['t'] == 'Str' else ' ' if item['t'] == 'Space' else ''
            for item in block['c'][2]
        )
        print(f'L{level}: {text}')
"
```

**Chapter splitting strategy for books:**
1. Extract full text with `pandoc` -> markdown
2. Identify chapter boundaries from headers (epubs have built-in TOC structure that pandoc preserves as `#`/`##` headers)
3. Split into one chunk per chapter
4. Dispatch parallel Sonnet subagents — one per chapter — same as any other long content
5. A typical book (60-100k words, 15-30 chapters) produces chapters of ~3-5k words each — well within subagent context limits

**For very long books (>30 chapters):** batch chapters into groups of ~5 per subagent to keep the number of parallel agents manageable. Each subagent summarizes its batch and returns section summaries.

**CRITICAL — Book summary depth requirement:**
- Each chapter MUST get its own dedicated `## Chapter N: Title` section with a **substantial** summary (300-600 words per chapter depending on chapter length)
- Do NOT batch multiple chapters into a single brief paragraph — every chapter gets its own detailed treatment
- Include key arguments, data points, examples, and quotes from each chapter
- A 10-chapter book should produce ~3000-6000 words of summary content (excluding frontmatter/tldr)
- A 30-chapter book should produce ~5000-10000 words
- Think of each chapter summary as a standalone mini-essay that captures the chapter's core contribution
- The goal is that someone reading the summary should understand what each chapter argues, not just what the book is "about" at a high level

**Output structure for books:**
- Location: `summaries/<Book Title>.md`
- Frontmatter tag: `book`

### Other files (txt, docx, etc.)

For `.docx`: `pandoc "<path>" -t markdown --wrap=none -o /tmp/summarize/doc.md`

For plain text: read directly.

### Pasted text / vault note
Read directly from user message or vault path.

## Step 2: Determine output structure

All content types land in `summaries/<Title>.md` (flat — no channel or show subfolders).

Frontmatter for all notes:

```yaml
---
created: YYYY-MM-DDTHH:MM
date: YYYY-MM-DD
tags: [<content type>]
status: unread
link: "<source URL or file path>"
summary: "one-line description"
---
```

- `tags`: one of `youtube`, `article`, `podcast`, `pdf`, `book`, `lecture`
- `date`: publication or upload date of the source content (not today's date)
- `link`: source URL, or file path for local files
- `summary`: one-line description used as the Obsidian note description

## Step 3: Analyze structure, determine depth, and plan sections

Read the full extracted text. Identify the natural sections/chapters/topics.

### 3a. Determine summary depth from source length

Summary length must be **proportional** to the source material. A 10-minute video and a 3-hour documentary should not produce the same size summary. Use the source word count to determine the target summary word count:

| Source word count | Source examples | Target summary words | Sections | TLDR |
|---|---|---|---|---|
| <1,500 | 5-min video, short article | 200-400 | 1-2 | 2 sentences |
| 1,500-5,000 | 10-20 min video, blog post, short paper | 500-1,200 | 3-5 | 3 sentences |
| 5,000-15,000 | 30-60 min video, long article, whitepaper | 1,500-3,000 | 5-8 | 3-4 sentences |
| 15,000-40,000 | 1-3 hr video/podcast, long paper | 3,000-6,000 | 8-15 | 4-5 sentences |
| 40,000-80,000 | Short book, multi-hour series | 5,000-10,000 | 15-25 | 5 sentences |
| 80,000+ | Full book (200+ pages) | 8,000-15,000 | 20-40 | 5 sentences |

**The ratio is roughly 1:5 to 1:10** — a 10,000-word source should produce ~1,500-2,500 words of summary. Denser/more technical content skews toward the higher end; conversational/repetitive content skews lower.

**For videos/podcasts**, estimate source words from duration: ~150 words/minute for conversational, ~120 words/minute for interviews with pauses, ~170 words/minute for scripted/narrated content. Or just use the actual transcript word count.

**Per-section depth**: each section's word budget should be proportional to its share of the source material. A section covering 20% of the transcript gets ~20% of the summary word budget. Adjust up for particularly dense/important sections, down for filler/repetitive ones.

### 3b. Plan sections and dispatch

**For long content (>3000 source words):** dispatch parallel Sonnet subagents — one per section — to summarize simultaneously. Each subagent gets:
- The section text
- The audience level
- A **specific word count target** (calculated from 3a above)

**For short content (<3000 source words):** summarize directly without subagents.

## Step 4: Assemble the summary note

### Structure

```markdown
---
[frontmatter per Step 2]
---

> [!tldr]
> [Overview — sentence count per Step 3a depth table. What is it about, who made it, what are the key takeaways?]

## [Section 1 Title]

[Summary paragraphs]

## [Section 2 Title]

[...]

## Further Reading

- Person or concept name -- brief one-line note on who/what they are
- Book or paper title -- brief one-line context
```

### Formatting rules

1. **No `# Title` heading** — filename is the title
2. **Never repeat frontmatter in the body** — if it's in metadata, don't write it again
3. **`> [!tldr]`** for the overview, not `## Summary`
4. **No wikilinks** unless the user explicitly asks for them
5. **Use actual Japanese/Chinese characters** for non-English words, not romanization
6. **Timestamps** on topic headings and inline when available (YouTube, podcasts)

### `## Further Reading` section

Always include at the end. A flat list of people, books, papers, and concepts mentioned in the content — one item per line, one-line note on each. No wikilinks. Examples:

```markdown
## Further Reading

- Benjamin Bloom -- educational psychologist, author of the Two Sigma Problem (1984)
- Angela Duckworth -- Grit: The Power of Passion and Perseverance (2016)
- Math Academy -- mastery-based math app, publishes learning-hours data per grade level
- Bloom's Two Sigma Problem -- landmark paper on tutoring vs. classroom instruction
```

### Audience adaptation

- **High school / college student**: plain language, analogies, explain jargon inline
- **General reader**: balanced — explain key terms but don't over-simplify
- **Expert**: technical language fine, focus on novel contributions and critiques

## Step 7: Update daily note

Update `$VAULT_ROOT/$DAILY_DIR/YYYY/MM-DD-YY ddd.md` (e.g. `daily/2026/04-11-26 Sat.md`). Create the `YYYY/` subdirectory if it doesn't exist. No `# Title` heading — the filename is the title.

```markdown
## content summary
- summarized [[Note Title]] -- [1-line description of what it is]
```

## Key rules

1. **No wikilinks** unless explicitly requested
2. **No `# Title` headings** — Obsidian shows filename as title
3. **Never repeat frontmatter in body** — frontmatter is metadata, body is content
4. **Always use Sonnet** — never Haiku
5. **Parallel Sonnet subagents** for long content — one per section
6. **Audience-appropriate language** — match the user's requested level
7. **Always embed/link the source** — source URL or file path in the `link` frontmatter field
8. **`> [!tldr]`** is mandatory — every summary starts with a concise overview callout
9. **`## Further Reading`** is mandatory — always close with a flat reference list
10. **Batch file input**: if the argument is a local file path, read each line as a URL and summarize all of them; use `--parallel` (default) or `--serial` to control execution order; write daily note once at the end
