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

## Step 1 — Extract the Video ID and Fetch the Video Title

Parse the YouTube URL and extract the 11-character video ID. Handle all formats:
- `https://www.youtube.com/watch?v=VIDEO_ID`
- `https://youtu.be/VIDEO_ID`
- `https://www.youtube.com/embed/VIDEO_ID`
- `https://www.youtube.com/shorts/VIDEO_ID`

Then fetch the video title:
```
cd C:\Users\navee\Documents\projects\pytranscribe && pipenv run yt-dlp --get-title "https://www.youtube.com/watch?v={VIDEO_ID}"
```

Store the raw video title — it is used in Step 2 to derive the session number and session name.

---

## Step 2 — Determine Series Metadata

Read `{Sessions folder}index.md`. From its H1 heading, extract the **series display name** (e.g. `"Surah Al-Hujuraat"`, `"Surah Yusuf"`).

**Parse the session number and name from the video title** (the title fetched in Step 1). The Token 3 argument is only a fallback if no number can be found in the title.

- Look for a session/part number pattern in the title, e.g. `Session 14`, `Session 14a`, `Part 3b`, `#7`, `Ep 5`, etc.
- Extract the **base session number** (digits only, e.g. `14` from `Session 14a`).
- Extract the **part letter** if present (e.g. `a` from `Session 14a`, `b` from `Part 3b`). If no letter is present, there is no part suffix.
- Extract the **session subtitle** — the remaining descriptive text in the title after stripping the series name and session/part marker (e.g. `"Munada and its types"` from `"Surah Al-Hujuraat Session 14 — Munada and its types"`). Use this as the human-readable title for the note. If no subtitle is found, derive one from the transcript content after reading it in Step 4.
- If the user also provided remaining title tokens (Token 4+), prefer those over the video-title-derived subtitle.

**Multi-part sessions (e.g. Session 14a and Session 14b):**
- Both parts share the same **base session number** (14).
- The **output filename** is always `session-{base_number}.md` (e.g. `session-14.md`) — the part letter is **never** in the filename.
- A part letter in the title signals that there may already be a file for this session (from an earlier part). Check whether `session-{base_number}.md` exists before writing.

If no session number can be parsed from the title and no Token 3 was given, stop and ask the user for the session number.

Derive a **short series tag** from the sessions folder name by stripping `-sessions` and converting to lowercase kebab, e.g.:
- `surah-alhujuraat-sessions` → `surah-hujuraat`
- `surah-yusuf-sessions` → `surah-yusuf`

And derive a **raw file title** for the transcript: `"{Series Display Name} Study Session {N}{part}"`, e.g. `"Surah Hujuraat Study Session 14a"` (include the part letter so the raw file is unique per part).

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

Check whether `{Sessions folder}{output filename}` already exists:

- **File does not exist** — write the full formatted note as-is.
- **File already exists** (multi-part scenario, e.g. processing Session 14b after 14a was saved):
  - Read the existing file.
  - Do **not** overwrite the frontmatter, H1 heading, or Overview section from the earlier part.
  - Append the new part's content as additional numbered sections continuing from where the existing sections end (e.g. if the existing file has sections 1–4, the new part starts at section 5).
  - Add a visible divider before the new part's sections:
    ```markdown
    ---

    ## Part {part_letter} — {part subtitle if different}
    ```
  - Merge the vocabulary tables from both parts into a single combined **Vocabulary Summary** section at the end.
  - Update the **Key Lessons** section to include lessons from both parts.
  - Save the merged file, overwriting the existing one.

---

## Step 6 — Update the Sessions Index

Read `{Sessions index}`.

Add a new entry in the **Study Sessions** list:
```markdown
- [Session {N}]({output filename})
```

Insert in ascending session-number order. If the entry already exists (which is expected when processing a later part of a multi-part session), leave it unchanged — do not add a duplicate.

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

**Alphabetical ordering (mandatory):** After inserting a new entry, re-read all entries in that section and sort them A–Z by the display name (the link text, not the filename). Rewrite the entire section list in sorted order. Apply this to every section that received a new entry. Do not modify entries in sections that did not change. If a topic does not fit any existing section, add a new section heading at a sensible location and apply the same alphabetical sorting to its entries.

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
