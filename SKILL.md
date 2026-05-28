---
name: video-frame-review
description: >
  Use when a screen recording (`.mov` / `.mp4` / `.webm`) appears in the
  conversation or the user references a video path. The skill probes the
  video first, then presents scoped options via `AskUserQuestion` (full
  vs specific window, default vs higher detail) tailored to the clip's
  duration — never burns tokens silently and never assumes scope or
  cadence. On user pick, extracts frames, dispatches a fast multimodal
  describer subagent (Haiku) to narrate each frame verbatim, iteratively
  re-extracts denser sub-ranges where evidence is ambiguous, zooms on
  small UI elements for OCR, and synthesizes findings into a diagnosis.
  Triggers on phrases like "revisa este video", "I recorded the bug",
  "trace what I did in the screencast", or when a video file path
  appears in the conversation.
---

# Video Frame Review

A discipline for turning a screen recording into a precise textual narration plus a synthesized diagnosis. The orchestrating model decides cadence and zones; a fast describer subagent (Haiku) does dense observational reads; the orchestrator iterates (zoom / densify) until evidence is verbatim, then synthesizes.

**Core separation of duties**:

| Role | Model | Job |
|---|---|---|
| Describer | **Haiku** (default) | Observation only. Verbatim text, exact colors, frame-to-frame deltas. **No inference, no opinions, no diagnosis.** |
| Orchestrator | Caller's model | Pick cadence, choose zones, iterate on ambiguities, synthesize the answer. |

When the describer says `[unreadable]` or hedges with "appears to be" / "approximately", the orchestrator's job is to **zoom or densify and re-dispatch** — never accept hedged evidence as the answer.

## When to invoke

The skill **auto-suggests itself** when a video appears in the conversation, but it **must probe the video first, then present scoped options via `AskUserQuestion`** — never burns tokens silently on a video the user didn't actually want analyzed, and never assumes scope/cadence without asking.

Flow on every invocation:

1. **Detect** a video reference (path drop, file attachment, trigger phrase like "revisa este video" / "I recorded the bug" / "trace what I did").
2. **Probe FIRST** — run `ffprobe` to read duration. The duration determines which options to present (a 9-second clip and a 5-minute clip get different choices).
3. **Present options via `AskUserQuestion`** (use the harness's structured-question tool, not a verbatim text prompt). Pick the option set from the table below based on duration. Each option has a short label + description so the user can choose visually.
4. **On user pick**: if they chose a "Full" or "Quick scan" preset, proceed to Step 1 with that cadence. If they chose "Specific window", ask a one-line follow-up: "Which window? Reply with `start-end` seconds (e.g. `15-25`)." If they chose Other and typed a custom cadence / scope, parse it.
5. **On skip / no reply**: no-op.

### Option sets by duration

The skill picks ONE of these four sets based on the probed duration:

**≤10s** (every second matters; "specific window" rarely useful):

| Label | Description | Cadence |
|---|---|---|
| Full · default detail | Every second of the clip, 1 frame per second | `fps=1` |
| Full · max detail | Two frames per second across the full clip | `fps=2` |
| (Other) | Lets the user type a custom cadence | user-defined |

**10–60s** (typical UI demos):

| Label | Description | Cadence |
|---|---|---|
| Full · default detail | One frame every 2 seconds | `fps=1/2` |
| Full · more detail | One frame per second | `fps=1` |
| Specific window | Focus on a chosen time range at higher density | user-picked window |
| (Other) | Custom cadence or other shape | user-defined |

**60–180s** (longer demos / debug recordings):

| Label | Description | Cadence |
|---|---|---|
| Full · default detail | One frame every 3 seconds | `fps=1/3` |
| Specific window (recommended) | Focus on the part that actually matters | user-picked window |
| Quick scan | One frame every 5 seconds, full clip | `fps=1/5` |
| (Other) | Custom | user-defined |

**>180s** (long sessions; the full-clip option is usually too coarse to be useful):

| Label | Description | Cadence |
|---|---|---|
| Specific window (recommended) | Pick the segment that matters; analyze it densely | user-picked window |
| Quick scan | One frame every 10 seconds, full clip | `fps=1/10` |
| Walk-through | One frame every 5 seconds, full clip | `fps=1/5` |
| (Other) | Custom | user-defined |

If the user is mid-action recording when you notice, **wait for the file to arrive** before triggering the question. Don't analyze a stream.

## Step 0 — Pre-flight (run this BEFORE anything else)

Every invocation starts here. Don't skip and don't defer.

```bash
command -v ffmpeg >/dev/null 2>&1 && command -v ffprobe >/dev/null 2>&1 && echo "ok" || echo "missing"
```

**If `ok`**: proceed to step 1.

**If `missing`**:

1. Detect platform: `uname -s` → `Darwin` (macOS) or `Linux`.
2. **macOS + Homebrew available** (`command -v brew` succeeds): tell the user, verbatim:
   > "`ffmpeg` is not installed and this skill needs it to extract frames from your video. Want me to install it now with `brew install ffmpeg`? (~30s)"

   On affirmative reply, run `brew install ffmpeg` and re-check. On negative or no reply, stop and tell the user the skill cannot proceed.
3. **Linux**: tell the user the distro-appropriate command (e.g. `sudo apt-get install ffmpeg` for Debian/Ubuntu) and ask them to run it themselves. The skill must not `sudo`. After they confirm install, re-check and proceed.
4. **macOS without Homebrew, or other**: surface the situation to the user and ask how they want to install `ffmpeg`. Don't guess.

Other requirements (built-in to Claude Code; skip the check if you know the harness):
- Multimodal `Read` tool that accepts PNG/JPG paths and returns image content.
- `Agent` tool to dispatch subagents with `model="haiku"` (Sonnet as fallback).

If those aren't present, the skill can't run — say so to the user and stop.

## Inputs

- **Video path** (required): absolute path. If from Slack, use `slack_read_file` to download; the returned blob path is your input.
- **Question or focus** (required): "Why does the deploy fail?", "Did the badge update after threshold change?", "Walk me through the steps". This shapes cadence and zones.
- **Optional zones of interest**: regions you already suspect matter (header badge, modal CTA, error toast). If unknown, start full-frame and refine after pass 1.

## Process

### 1. Probe

```bash
ffprobe -v error -show_entries format=duration:stream=width,height,r_frame_rate \
  -of default=noprint_wrappers=1 "$VIDEO"
```

Capture `duration`, `width`, `height`. If the path has non-ASCII chars (accents, ñ, spaces) and `ffprobe` returns "No such file", copy to `/tmp/clip.mov` first — macOS NFD/NFC mismatch bites here.

### 2. Apply the user's chosen cadence and scope

By this point the user already picked an option in the invocation gate (full vs specific window, default vs higher detail). Use those choices directly — don't re-decide. The duration-bucket table from the option set already mapped the user's pick to an explicit `fps` value.

Default scale: cap width at **1960px**. Higher resolutions burn tokens for marginal gain at modal-body-text scale. If the user reports unreadable text after pass 1, bump to `scale=2940:-1`.

The user can override at any later point ("actually re-extract at 1 fps over the first 20 seconds") — re-run this step with their new parameters and discard the previous frames before continuing.

```bash
OUT=/tmp/<clip-name>-frames
mkdir -p "$OUT" && rm -f "$OUT"/*.png
ffmpeg -hide_banner -loglevel error -i "$VIDEO" \
  -vf "fps=1/2,scale=1960:-1" "$OUT/f_%02d.png"
```

`fps=1/N` = one frame every N seconds. `fps=N` = N frames per second.

### 3. Sanity-check ONE frame yourself

Before dispatching the subagent, `Read` `f_01.png` yourself. Confirm:

- Resolution gives readable text in the regions you care about. If modal-body text is blurry, re-extract at `scale=2940:-1`.
- Layout — where the header is, where modals open, where toasts land. This informs your zone-of-interest hints in the dispatch prompt.

**Skipping this step is the #1 way to burn tokens on a 50-frame run that turns out unreadable.**

### 4. Dispatch the describer (Haiku)

Dispatch via `Agent` with **`model="haiku"`**, `subagent_type="general-purpose"`. Haiku is faster, cheaper, and equally precise on OCR/description tasks. Sonnet is only worth it when you've already tried Haiku on a critical crop and the verbatim read is wrong or fuzzy.

The describer's job is **observation only**. Make the prompt enforce this:

```
You are an observational describer. Your only job is to report what is
visible in each frame — verbatim text, exact colors, layout positions,
and changes between consecutive frames. You do NOT diagnose, interpret,
or speculate about why something is happening. If you find yourself
writing "this is a bug" / "should be" / "appears to indicate" — stop
and rewrite as a pure observation.

I need a detailed chronological narration of a [duration]-second screen
recording. The video is split into [N] frames (one every [interval]s),
located at:

  /tmp/<clip>-frames/f_01.png through f_<N>.png

Context for the recording: [brief — what app, what screen, what the user
appears to be doing. Do NOT include hypotheses about what might fail.]

For each frame, describe in detail (1-3 sentences each):

1. **Top header / right side**: [name the specific elements you expect
   here — badge, version selector, action buttons]. Quote any visible
   labels verbatim.
2. **Modals open**: which modal is visible (title verbatim), main
   content sections, action buttons (label + color).
3. **Modal content**:
   - [list the sections you expect: e.g. Target, Test Summary, error
     banners, footer buttons]. Quote any visible values verbatim
     (version numbers, percentages, IDs).
4. **State change vs previous frame**: name exactly what differs.
5. **Any error toast / banner / notification**: location on screen,
   background color, icon, verbatim text inside.

Specifically watch for and quote verbatim:
- [list 3-6 specific things the orchestrator cares about for this run]
- UUIDs in any modal field (format XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX)
- Any inconsistency between modal content and global UI state.

Discipline:
- If text is too small/blurry to read confidently, write `[unreadable]`
  — do NOT guess. Describe what's roughly there ("amber pill, ~10
  characters, can't make out") so the orchestrator can decide to zoom.
- If a region's text is unreadable, do NOT commit to its color/icon
  either — say `[visual attributes unclear at this resolution]`.
- Quote text in "double quotes" verbatim, including punctuation and
  ellipses. Do not paraphrase.
- Don't interpret. "The button is green and labeled Deploy Now" is
  correct. "The button looks enabled, so deploy should work" is not.
- **Compress identical frames.** If a frame is visually identical to
  the previous (same modal, same cursor, same labels, same button
  states) — DO NOT re-describe every section. Output one line only:
  "Identical to frame N — no observable change." Save the orchestrator
  tokens; the rich detail you wrote for frame N still applies.

Output format: one block per frame numbered 01-<N>, each in:

  ## Frame 01 (00:00s)
  - Header: ...
  - Modal: ...
  - Modal content: ...
  - Change from previous: ...
  - Notes: ...

Final section: **Summary of key state transitions** (e.g. "frame 03 →
Test Runner opened", "frame 17 → red banner appeared") — observational,
not interpretive.
```

### 5. Triage the narration — when to zoom, when to densify

Read the narration end-to-end before deciding next steps. Three categories of ambiguity to look for:

**Trigger ZOOM (targeted crop + OCR re-dispatch) when:**

- Describer wrote `[unreadable]` for any text.
- Describer hedged: "appears to be", "approximately", "something like", "possibly".
- Critical evidence (error banner text, modal title, version number, UUID, button label) is described but not verbatim-quoted.
- Color is reported but exact shade matters (e.g. "red vs amber" distinguishes failure vs warning).
- An icon is named generically ("warning icon", "check icon") and the exact icon matters for state classification.

Crop with ffmpeg, scaled if needed:

```bash
cd "$OUT" && ffmpeg -hide_banner -loglevel error -i f_17.png \
  -vf "crop=W:H:X:Y" -y zoom17_topbar.png
```

`crop=W:H:X:Y` is `width:height:x_offset:y_offset`. Eyeball the rectangle from the sanity-check frame.

Re-dispatch a **short, focused** OCR task — still Haiku, still observation-only:

```
Read /tmp/<clip>-frames/zoom17_topbar.png. Give me the EXACT verbatim
text of every visible word/letter — no paraphrasing, no guessing.
Include:
- Banner background color
- Icon (if any) — describe it shape-by-shape rather than naming it
- Verbatim text (in "double quotes")
- Button text inside the banner (verbatim)
- If something is genuinely unreadable, say `[unreadable]` and describe
  what's there (color, approximate length).
```

**When to dispatch the crop to Sonnet instead of Haiku** (rare but worth knowing):

- **Haiku already failed** on this same crop — returned `[unreadable]` or hedged after the zoom. Escalate that one crop only, not the whole video.
- **Non-English / non-Spanish text** in the crop (CJK, Arabic, Devanagari, Cyrillic, accented Latin like Vietnamese). Haiku's OCR is noticeably less precise on these scripts; if the critical evidence is text in one of them, go to Sonnet from the start of pass 2 rather than waiting for Haiku to hedge.
- **Stylized / handwritten / decorative fonts** that don't look like standard system fonts. Haiku can mis-read curly script or hand-drawn UI mockups; Sonnet is more reliable there.
- **Heavily degraded compression** — screenshots that have been re-encoded multiple times (e.g. screenshot → Slack image compression → re-saved as JPEG → re-uploaded), where Haiku starts confusing similar glyphs (e.g. `0` vs `O`, `1` vs `l` vs `I`).
- **Sub-pixel rendering issues** (HiDPI source rendered at the wrong scale, anti-aliasing artifacts). Bump to Sonnet only on the affected crop.

Outside these cases, stay on Haiku. It's ~5× cheaper and equally precise for standard Latin-script UI text at reasonable resolution, which is 95%+ of real screencasts.

**Trigger DENSIFY (re-extract a sub-range at higher fps) when:**

- The describer's "change from previous" gap is too big — a state changed between two frames but neither frame catches the actual transition. Example: frame 16 = modal idle, frame 17 = error banner + spinner. The click that caused this lives between them.
- A typed sequence is sampled too sparsely (typing is ~1 char/sec; 0.5 fps misses half the characters).
- A toast appeared and disappeared inside a single sampling window.
- The orchestrator's question is about timing ("how long after X did Y appear?") and the current cadence can't resolve it.

Re-extract the sub-range with a precise time window:

```bash
ffmpeg -hide_banner -loglevel error -ss 30 -to 36 \
  -i "$VIDEO" -vf "fps=2,scale=1960:-1" \
  "$OUT/dense_30s_%02d.png"
```

`-ss START -to END` (seconds) picks the window; `fps=2` is 2 frames per second. Re-dispatch the describer **only on the new frames**, with a tight prompt focused on what changed in that window.

**Iteration discipline**: each pass should reduce ambiguity. If after zoom + densify you still don't have verbatim evidence for the question, ask the user to re-record at higher resolution or share the underlying request payload / logs — don't make a fourth pass on the same low-quality frame.

### 6. Synthesize

You now have:
- A baseline narration (step 4).
- Verbatim evidence on critical moments (step 5 zooms).
- Optionally, dense sampling of key sub-ranges (step 5 densify).

**Now** the orchestrator interprets. Cross-reference observations against:
- Code paths that could produce the verbatim error string seen.
- Known sources of truth that should agree (e.g. modal field vs header badge).
- State machine transitions that should be cause-and-effect (click → response).

**Output to the user**: one short paragraph describing what actually happened, followed by a diagnosis. **Quote verbatim** for any string driving the diagnosis. Distinguish observation from inference explicitly.

**Cleanup (optional)**: after synthesizing, the extracted PNGs in `/tmp/<clip>-frames/` are no longer needed — the narration and verbatim crops already live in the conversation. If you ran multiple analyses in one session and `/tmp` is getting crowded, `rm -rf /tmp/<clip>-frames` cleans up. Skippable for single-run sessions.

Example output (synthetic — illustrating the shape):

> The video shows: user fills out a form, clicks Save. Server returns
> in a red toast at the top of the screen:
> > `Stale record: expected revision 3, got 4. Reload and try again.`
>
> Diagnosis: this is an **optimistic-concurrency rejection** on the
> Save endpoint, not a validation failure. The form was loaded against
> revision 3, but another tab/session (or a backend job) advanced the
> record to revision 4 in the meantime. The FE doesn't auto-refresh the
> form when this fires — the user has to manually reload to pick up
> the newer revision and re-apply their edits.

## Output discipline (orchestrator)

- **Quote verbatim** anything that drives the diagnosis. Paraphrasing breaks downstream `grep`s.
- **Label observation vs inference**. "The badge says X" is observation; "the badge said X because the atom wasn't updated" is inference — mark it as such.
- **Cite frame numbers** for state transitions ("Frame 17 — error banner appears, modal stays open"). Lets the user replay if they disagree.
- **Don't speculate beyond evidence**. If the critical text remained `[unreadable]` after zoom passes, surface that explicitly and ask the user to re-record at higher fidelity or share the API response directly — don't fill the gap with plausible-sounding guesses.

## Common pitfalls

- **Resolution too low → re-extract at higher scale**. `scale=2940:-1` for native 4K recordings; for 1080p sources, native scale is usually fine.
- **Wrong cadence → densify the relevant sub-range**. Don't re-extract the whole video at 1 fps if only a 5-second window matters.
- **Trusting describer interpretation over describer observation**. If the describer says "looks like an error appeared", re-crop and OCR the actual text before believing what kind of error it is.
- **Skipping the sanity-check (step 3)**. Reading one frame yourself takes 5 seconds and prevents 50-frame token waste.
- **Defaulting to Sonnet for narration**. Haiku is the right default. Save Sonnet for the rare case where Haiku returned hedged or wrong output on a critical crop.
- **Non-ASCII file paths + ffprobe failing**. `cp` to `/tmp/clip.mov` first.

## When to escalate

If even after zoom + densify the diagnosis is ambiguous (e.g. you have the verbatim error string but multiple code paths could emit it), ask the user for permission to:

- Query the relevant DB tables for the user / agent / session shown in the video.
- Pull BE logs around the timestamp the video was recorded.
- Cross-reference deploy / CI events that may have changed runtime behavior.

The video gives the user's perspective; the logs give the system's. Both together are usually decisive — and the video has already done its job by pointing you at the precise moment and verbatim error to grep for.

## Quick reference

Default knobs:

| Knob | Default | Bump up when |
|---|---|---|
| Cadence | 0.5 fps (one every 2s) for 10–60s clips | Fast state changes (typing, transitions); densify the relevant window |
| Scale | 1960px wide | Modal body text unreadable at default → 2940px |
| Describer model | Haiku | Non-Latin script, stylized/handwritten fonts, heavily re-compressed images, or Haiku already returned hedged on the same crop → escalate that one crop to Sonnet |
| Passes | 1 narration + targeted zooms/densifies | If pass 2 still has `[unreadable]` on critical evidence → ask user for higher-fidelity recording or direct payload |
