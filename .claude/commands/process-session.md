---
description: End-to-end pipeline for any study session: transcribe a YouTube video, format into MkDocs Material notes, save to the correct sessions folder, and create or update atomic topic notes. Works for any series (Surah Al-Hujuraat, Surah Yusuf, etc.).
argument-hint: <youtube-url> <sessions-folder> <session-number> [title]
---

# Process Study Session

You are running a full pipeline to process a YouTube study session video into structured study notes. The user has invoked this command with:

```
$ARGUMENTS
```

Parse the arguments:
- **Token 1** — YouTube URL (required)
- **Token 2** — Sessions folder name, e.g. `surah-alhujuraat-sessions` or `surah-yusuf-sessions` (required)
- **Token 3** — Session number, e.g. `4` (required)
- **Remaining tokens** — optional human-readable session title/subtitle, e.g. `"Story of Yusuf Part 3"`. If omitted, derive it from the transcript content after reading.

**Base paths** (fixed across all series):
- Sessions root: `C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\docs\`
- Topics folder: `C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\docs\topics\`
- Topics index: `C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\docs\topics\index.md`
- Raw transcripts: `C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\raw-md-files\`
- Pytranscribe dir: `C:\Users\navee\Documents\projects\pytranscribe\`

Derived paths:
- Sessions folder: `{Sessions root}{sessions-folder}\`
- Sessions index: `{Sessions folder}index.md`

---

## Step 1 — Extract the Video ID

Parse the YouTube URL and extract the 11-character video ID. Handle all formats:
- `https://www.youtube.com/watch?v=VIDEO_ID`
- `https://youtu.be/VIDEO_ID`
- `https://www.youtube.com/embed/VIDEO_ID`
- `https://www.youtube.com/shorts/VIDEO_ID`

---

## Step 2 — Determine Series Metadata

Read `{Sessions folder}index.md`. From its H1 heading, extract the **series display name** (e.g. `"Surah Al-Hujuraat"`, `"Surah Yusuf"`).

The **output filename** is always `session-{N}.md`.

Derive a **short series tag** from the sessions folder name by stripping `-sessions` and converting to lowercase kebab, e.g.:
- `surah-alhujuraat-sessions` → `surah-hujuraat`
- `surah-yusuf-sessions` → `surah-yusuf`

And derive a **raw file title** for the transcript: `"{Series Display Name} Study Session {N}"`, e.g. `"Surah Hujuraat Study Session 4"`.

---

## Step 3 — Update the Transcription Script

Edit `C:\Users\navee\Documents\projects\pytranscribe\fetch_transcript.py`.

Update exactly these three variables and nothing else:
- `VIDEO_ID` → extracted video ID
- `OUTPUT_PATH` → `r"C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\raw-md-files\{raw file title}.md"`
- `TITLE` → `"{raw file title}"`

Also create or overwrite `C:\Users\navee\Documents\projects\pytranscribe\current-session.md`:

```markdown
# Current Session

- **Video ID**: {VIDEO_ID}
- **Series**: {Series Display Name}
- **Session**: {N}
- **URL**: {full YouTube URL}
- **Output**: raw-md-files/{raw file title}.md
```

---

## Step 4 — Run the Transcription Script

Run the script inside the pipenv environment:
```
cd C:\Users\navee\Documents\projects\pytranscribe && pipenv run python fetch_transcript.py
```

Confirm the output file was created. If the fetch fails:
- `TranscriptsDisabled` / `NoTranscriptFound` with a language restriction → retry without the `languages=` argument
- `TranscriptsDisabled` with no transcript at all → report to user and stop
- Any other error → report the error message to the user and stop

---

## Step 5 — Format the Transcript into Structured Notes

Read the raw transcript from `raw-md-files\{raw file title}.md`.

The raw file is auto-generated captions: continuous text, no punctuation, no structure. Reconstruct it into a fully formatted, publication-ready study note.

### How to format

Apply this structure (adapt section count and titles to what was actually taught):

```markdown
---
title: {Series Display Name} — Study Session {N}
tags:
  - quran
  - arabic-grammar
  - {series-tag}
  - {2–4 specific topic tags, e.g. munada, tafseer, vocabulary, morphology}
---

# {Series Display Name} — Study Session {N}

---

## Overview

The main topics covered in this session are:

- {bullet 1}
- {bullet 2}
- ...

---

## 1. {First Major Topic}

### 1.1 {Sub-section if needed}

{Prose explanation. Bold all Arabic terms: **كَلِمَة (kalimah)**. Transliterate in italics or parentheses.}

{Tables for paradigms, root derivatives, or vocabulary sets:}

| Arabic | Transliteration | Meaning |
|---|---|---|
| **word** | transliteration | meaning |

{Admonitions — use the right type:}

!!! note "..."
    Grammatical notes or reminders.

!!! example "..."
    Quranic examples or applied illustrations.

!!! info "..."
    Background or context.

!!! important "..."
    Key rules the student must memorise.

!!! tip "..."
    Learning strategies or mnemonics.

!!! question "..."
    Rhetorical or student questions raised in the session.

!!! summary "..."
    Quick-reference rule tables at the end of a section.

## 2. {Second Topic}
...

## {N}. Vocabulary Summary

| Arabic | Root | Pattern / Form | Meaning |
|---|---|---|---|
| **verb / present** | root | Form I / II / etc. | meaning |

## {N+1}. Key Lessons from This Session

!!! tip "Summary of Lessons"
    1. ...
    2. ...

---

*{Closing sentence: what was announced for next session, or any homework}*
```

**Formatting rules:**
- Bold Arabic terms; transliterate in italics or parentheses inline
- Quranic verses in a blockquote: `> *"…"*`
- Preserve ALL content — student questions, tangents, cross-references, word-history stories
- Do not invent content that was not in the transcript
- Section numbering starts at 1 (after Overview)

---

## Step 6 — Save the Formatted Session File

Save the formatted content to:
```
{Sessions folder}{output filename}
```

---

## Step 7 — Update the Sessions Index

Read `{Sessions index}`.

Add a new entry in the **Study Sessions** list:
```markdown
- [Session {N}]({output filename})
```

Insert in ascending session-number order. If the entry already exists, leave it unchanged.

---

## Step 8 — Create or Update Atomic Topic Notes

Re-read the formatted session file. Identify every distinct **grammar rule, morphology pattern, vocabulary root, Quranic concept, or learning methodology** discussed.

For each topic:

1. Check if a matching file already exists in the topics folder. Use the topics index as a guide, and also Glob for related filenames.

2. **Existing file** — read it and add any new information, examples, or Quranic references from this session. Integrate naturally, or add a clearly labelled sub-section (e.g. `### From Session {N}`) when the new content is self-contained and distinct.

3. **New topic** — create a new file in `docs/topics/` using this structure:

```markdown
---
title: {Topic Name}
tags:
  - {arabic-grammar / quran / morphology / vocabulary — whichever fits}
  - {series-tag}
---

# {Topic Name}

{1–3 sentence definition or introduction.}

---

## {First section from this session}

{Content}

## Examples from the Quran

{Any Quranic occurrences mentioned}

## Session References

- {Series Display Name} Session {N}: {one-line summary}
```

Choose a kebab-case filename:
- Grammar topics: `munada-rules.md`, `idgham.md`
- Root studies: `root-nadaa.md`
- Patterns: `faala-pattern.md`
- Vocabulary / verbs: `verb-nadaa.md`

Check the topics folder and index before creating — never duplicate an existing note.

---

## Step 9 — Update the Topics Index

Read `docs/topics/index.md`.

For each **new** topic file, add an entry under the appropriate section:
- `## Grammar`
- `## Vocabulary and Morphology`
- `## Roots and Etymology`
- `## Quranic Concepts`
- `## Methodology`
- `## Context and History`
- `## Recitations`

Format: `- [Display Name](filename.md)`

Insert alphabetically within the section. Do not modify existing entries. If a topic does not fit any existing section, add a new section heading at a sensible location.

---

## Final Report

Output a brief summary when done:

```
Done. Processed {Series Display Name} Session {N}: "{title}"

Files created/updated:
  NEW   docs/{sessions-folder}/{output filename}
  UPD   docs/{sessions-folder}/index.md
  NEW   docs/topics/{new-topic}.md      (repeat for each new file)
  UPD   docs/topics/{existing-topic}.md (repeat for each updated file)
  UPD   docs/topics/index.md
```
