---
description: End-to-end pipeline for any study session: transcribe a YouTube video, format into MkDocs Material notes, save to the correct sessions folder, and create or update atomic topic notes. Works for any series (Surah Al-Hujuraat, Surah Yusuf, etc.). Also supports processing an entire playlist one video at a time.
argument-hint: <youtube-url-or-playlist-url> <sessions-folder> <session-number> [title]
---

# Process Study Session

You are running a full pipeline to process a YouTube study session video (or playlist) into structured study notes. The user has invoked this command with:

```
$ARGUMENTS
```

Parse the arguments:
- **Token 1** — YouTube URL or playlist URL (required). Detect playlist mode when the URL contains `list=` **and** does NOT contain `v=` (i.e. it is a pure playlist URL like `https://www.youtube.com/playlist?list=PLxxx`). A URL with both `v=` and `list=` is a single-video URL — process only that one video.
- **Token 2** — Sessions folder name, e.g. `surah-alhujuraat-sessions` or `surah-yusuf-sessions` (required)
- **Token 3** — Session number (required). For a playlist this is the **starting** session number; subsequent videos in the playlist are numbered N+1, N+2, etc.
- **Remaining tokens** — optional human-readable session title/subtitle (single video only). If omitted, derive it from the transcript content after reading.

---

## Playlist Mode vs Single-Video Mode

If the URL is a **playlist URL** (contains `list=` but not `v=`):

1. Run the following command to extract the ordered list of video IDs **without downloading anything**:
   ```
   cd C:\Users\navee\Documents\projects\pytranscribe && pipenv run yt-dlp --flat-playlist --print id "PLAYLIST_URL"
   ```
   This prints one video ID per line. Collect them into an ordered list.

2. For each video ID in order, run the **full single-video pipeline** (Steps 1–8 below), using session number = starting_N + index (0-based index).

3. **Rate-limit between videos**: after completing all steps for one video and before starting the next, wait **30 seconds**. Use:
   ```
   python -c "import time; time.sleep(30)"
   ```
   Run this inside the pipenv environment so no extra install is needed.

4. After finishing all videos, print a combined final report listing every file created or updated across all sessions.

If the URL is a **single-video URL**, skip directly to Step 1 below and process just that one video.

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

## Step 3 — Run the Transcription Script

Run the script inside the pipenv environment, passing the video ID, output path, and title as arguments:
```
cd C:\Users\navee\Documents\projects\pytranscribe && pipenv run python fetch_transcript.py {VIDEO_ID} "C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\raw-md-files\{raw file title}.md" "{raw file title}"
```

The script automatically retries without a language filter if the preferred languages are unavailable. Confirm the output file was created. If the fetch fails:
- `TranscriptsDisabled` with no transcript at all → report to user and stop
- Any other error → report the error message to the user and stop

---

## Step 4 — Format the Transcript into Structured Notes

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

## Step 5 — Save the Formatted Session File

Save the formatted content to:
```
{Sessions folder}{output filename}
```

---

## Step 6 — Update the Sessions Index

Read `{Sessions index}`.

Add a new entry in the **Study Sessions** list:
```markdown
- [Session {N}]({output filename})
```

Insert in ascending session-number order. If the entry already exists, leave it unchanged.

---

## Step 7 — Create or Update Atomic Topic Notes

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

## Step 8 — Update the Topics Index

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

**Single-video mode** — output a brief summary when done:

```
Done. Processed {Series Display Name} Session {N}: "{title}"

Files created/updated:
  NEW   docs/{sessions-folder}/{output filename}
  UPD   docs/{sessions-folder}/index.md
  NEW   docs/topics/{new-topic}.md      (repeat for each new file)
  UPD   docs/topics/{existing-topic}.md (repeat for each updated file)
  UPD   docs/topics/index.md
```

**Playlist mode** — after all videos are done, output one consolidated report:

```
Done. Processed {total} sessions from playlist.

{Series Display Name} Session {N}: "{title}"
  NEW   docs/{sessions-folder}/session-{N}.md
  ...

{Series Display Name} Session {N+1}: "{title}"
  NEW   docs/{sessions-folder}/session-{N+1}.md
  ...

Topics (all sessions):
  NEW   docs/topics/{new-topic}.md
  UPD   docs/topics/{existing-topic}.md
  UPD   docs/topics/index.md
```

If any video in the playlist fails to transcribe, log the failure inline (`FAILED session-{N}: {reason}`) and continue to the next video rather than stopping the entire run.
