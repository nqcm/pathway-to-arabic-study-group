---
description: Format all pre-transcribed session files for a series into MkDocs Material notes and create or update atomic topic notes. Works for any series (Surah Al-Hujuraat, Surah Yusuf, etc.).
argument-hint: <sessions-folder> [start-from-session-number]
---

# Process Study Session

You are processing all pre-transcribed study sessions for a series into structured study notes. The user has invoked this command with:

```
$ARGUMENTS
```

Parse the arguments:
- **Token 1** — Sessions folder name, e.g. `surah-alhujuraat-sessions` or `surah-yusuf-sessions` (required)
- **Token 2** — Optional starting session number. If given, skip all transcript files whose session number is below this value.

---

**Base paths** (fixed across all series):
- Sessions root: `C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\docs\`
- Topics folder: `C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\docs\topics\`
- Topics index: `C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\docs\topics\index.md`
- Raw transcripts: `C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\raw-md-files\`

Derived paths:
- Sessions folder: `{Sessions root}{sessions-folder}\`
- Sessions index: `{Sessions folder}index.md`

---

## Step 1 — Determine Series Metadata and Discover Transcripts

Read `{Sessions folder}index.md`. From its H1 heading, extract the **series display name** (e.g. `"Surah Al-Hujuraat"`, `"Surah Yusuf"`).

Derive a **short series tag** from the sessions folder name by stripping `-sessions` and converting to lowercase kebab, e.g.:
- `surah-alhujuraat-sessions` → `surah-hujuraat`
- `surah-yusuf-sessions` → `surah-yusuf`

**Discover transcript files:** Glob `{Raw transcripts}*.md` and filter to files whose name contains a significant word from the series display name (e.g. `Hujuraat`, `Yusuf`) and also contains `Session` or `Part`. These are the transcripts to process.

**Sort** the discovered files by their session number in ascending order. For multi-part sessions (e.g. `Session 14a`, `Session 14b`), sort part `a` before part `b`.

**Apply the start-from filter:** If Token 2 was given, drop all files whose base session number is below that value.

Print the ordered list of transcript files that will be processed, then proceed.

---

## Per-Transcript Pipeline

For **each transcript file** in the sorted list, run Steps 2–6 in order. After finishing all steps for one file, move to the next — do not batch across files.

The sessions index and MkDocs navigation are **not** updated inside this loop. They are written once after all session files exist (see Post-Pipeline steps below).

### Step 2 — Parse Session Metadata from Filename

From the transcript filename:

- Extract the **base session number** (digits only, e.g. `14` from `Session 14a`).
- Extract the **part letter** if present (e.g. `a` from `Session 14a`). If absent, there is no part suffix.
- Extract the **session subtitle** — descriptive text beyond the series name and session marker. If none found, derive it from the transcript content after reading.

**Multi-part sessions (e.g. Session 14a and Session 14b):**
- Both parts share the same **base session number** (14).
- The **output filename** is always `session-{base_number}.md` — the part letter is **never** in the filename.
- A part letter signals there may already be a file from an earlier part. Check for it before writing.

**Skip already-processed sessions:** If `session-{base_number}.md` already exists **and** the current file has no part letter (i.e. it is not a continuation of a multi-part session), skip this transcript and note it as `SKIPPED` in the final report.

### Step 3 — Format the Transcript into Structured Notes

Read the raw transcript from `{Raw transcripts}{transcript-filename}`.

The raw file is auto-generated captions: continuous text, no punctuation, no structure. Reconstruct it into a fully formatted, publication-ready study note.

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

| Arabic | Transliteration | Meaning |
|---|---|---|
| **word** | transliteration | meaning |

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

### Step 4 — Save the Formatted Session File

Check whether `{Sessions folder}session-{base_number}.md` already exists:

- **File does not exist** — write the full formatted note as-is.
- **File exists** (processing a later part of a multi-part session):
  - Read the existing file.
  - Do **not** overwrite the frontmatter, H1 heading, or Overview section.
  - Append the new part's content as additional numbered sections continuing from the last existing section number.
  - Add a divider before the new sections:
    ```markdown
    ---

    ## Part {part_letter} — {part subtitle if different}
    ```
  - Merge vocabulary tables into a single combined **Vocabulary Summary** at the end.
  - Update the **Key Lessons** section to include lessons from both parts.
  - Save, overwriting the existing file.

### Step 5 — Create or Update Atomic Topic Notes

Re-read the formatted session file. Identify every distinct **grammar rule, morphology pattern, vocabulary root, Quranic concept, or learning methodology** discussed.

For each topic:

1. Check if a matching file already exists in the topics folder. Use the topics index as a guide, and also Glob for related filenames.

2. **Existing file** — read it and add any new information, examples, or Quranic references. Integrate naturally, or add a clearly labelled sub-section (e.g. `### From Session {N}`) when the new content is self-contained.

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

Never duplicate an existing note.

### Step 6 — Update the Topics Index

Read `docs/topics/index.md`.

For each **new** topic file created in Step 5, add an entry under the appropriate section:
- `## Grammar`
- `## Vocabulary and Morphology`
- `## Roots and Etymology`
- `## Quranic Concepts`
- `## Methodology`
- `## Context and History`
- `## Recitations`

Format: `- [Display Name](filename.md)`

**Alphabetical ordering (mandatory):** After inserting, sort all entries in the affected section A–Z by display name and rewrite the section. Do not touch sections that received no new entries. Add a new section heading if no existing section fits.

---

## Post-Pipeline Steps

Run these **once**, after every transcript in the sorted list has been processed through Steps 2–6.

### Step 7 — Write the Sessions Index

Read `{Sessions index}`.

For every session file that was written or updated during this run, read its saved `session-{N}.md` file and extract the **Overview** bullet list (the items under `## Overview`). Condense them into a short comma-separated subtitle (drop filler words; keep Arabic terms and grammar names).

Build the full **Study Sessions** list in ascending session-number order:

```markdown
- [Session {N}](session-{N}.md) — {subtitle derived from Overview bullets}
```

Rules:
- Include **all** sessions that exist in the folder (not just the ones processed in this run) — read any pre-existing entries from the current index and keep them, updating the subtitle only if the session was rewritten in this run.
- If an entry already exists and the session was **not** touched this run, copy its existing subtitle unchanged.
- Do not open any raw transcript files at this step — all content comes from the saved `session-{N}.md` files.

**About/intro paragraph:** If the sessions folder is newly created (the index had no Study Sessions list before this run), write a 2–4 sentence intro paragraph under the H1 before the `## Study Sessions` heading. Describe what the series is, what text is studied, and what skill it builds. Do not overwrite an existing intro paragraph.

Save the updated `{Sessions index}`.

### Step 8 — Update MkDocs Navigation

Read `mkdocs.yml` at the project root (`C:\Users\navee\OneDrive\Documents\prod-projects\study-notes\mkdocs.yml`).

Check whether the series already has an entry under the `nav:` key by looking for `{sessions-folder}` in any nav line.

- **Already present** — leave `mkdocs.yml` unchanged.
- **Not present** — insert a new nav entry immediately **before** the `- Topics:` line, preserving the two-space indent style:
  ```yaml
    - {Series Display Name} Sessions: {sessions-folder}/index.md
  ```

---

## Final Report

After Steps 7–8 complete, output a consolidated report:

```
Done. Processed {total} sessions for {Series Display Name}.

Session {N}: "{subtitle}"
  NEW   docs/{sessions-folder}/session-{N}.md

Session {N+1}: "{subtitle}"
  UPD   docs/{sessions-folder}/session-{N+1}.md   ← part b merged

...

SKIPPED  Session {X}: session file already existed

Topics (all sessions):
  NEW   docs/topics/{new-topic}.md
  UPD   docs/topics/{existing-topic}.md
  UPD   docs/topics/index.md

Index:
  UPD   docs/{sessions-folder}/index.md

Navigation:
  UPD   mkdocs.yml   ← added "{Series Display Name} Sessions"
  (or SKIPPED mkdocs.yml — series already in nav)
```
