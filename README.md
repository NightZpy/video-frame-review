# video-frame-review

A Claude Code skill for analyzing screen recordings of UI features, bug repros, or workflow walkthroughs. Extracts frames at the right cadence, dispatches a multimodal subagent to narrate each frame verbatim, and uses targeted crops for OCR on critical moments (error toasts, version transitions, modal CTAs).

The skill turns a video into a precise textual narration plus a synthesized diagnosis. Designed for the case where reading text from a static image is easy but watching a 60-second clip and pulling out the exact verbatim error string is tedious.

## Why this exists

LLMs reason great about images and poorly about video. Breaking a recording into discrete frames + targeted region crops makes the video tractable: each frame is just an image. The orchestrating model decides cadence and zones of interest; a fast subagent (Sonnet/Haiku) does the dense observational read; the orchestrator synthesizes.

## Requirements

- `ffmpeg` + `ffprobe` on `PATH` (Homebrew: `brew install ffmpeg`)
- Claude Code (or any harness with `Read`-image and `Agent`-with-subagent support)

The skill checks for `ffmpeg` at the start of every run and will offer to install it via Homebrew if missing — answer yes to the prompt and it handles the install before continuing.

## Install

Drop the folder under `~/.claude/skills/` (or `~/.claude-<profile>/skills/` if you use a custom config dir):

```bash
cd ~/.claude/skills && git clone https://github.com/NightZpy/video-frame-review.git
```

Then restart your Claude Code session. The skill auto-loads from `SKILL.md`.

## When the skill fires

The skill **auto-suggests itself** when a video shows up in your conversation — but it **probes the video first and then presents structured options** (not a wall of text) before doing any work. Never burns tokens silently.

On detect, the flow is:

1. The skill runs `ffprobe` to read the video's duration.
2. It presents you a **picker** (via the harness's structured-question UI) with 3-4 options tailored to that duration — e.g. for a 9-second clip you get something like:
   - *Full · default detail* (1 frame per second)
   - *Full · max detail* (2 frames per second)
   - *(Other)* — type your own cadence
3. For a 5-minute clip the options skew toward picking a window:
   - *Specific window (recommended)* — you pick `start-end` seconds
   - *Quick scan* (1 frame every 10s)
   - *Walk-through* (1 frame every 5s)
   - *(Other)*
4. On your pick, the analysis proceeds. On skip/no reply, the skill no-ops.

The exact option sets per duration bucket live in [SKILL.md](./SKILL.md). The point is: you get a tight set of choices that already match the clip you're analyzing, rather than a generic "what cadence do you want?" prompt.

## How the frame system works

Videos are hard for LLMs; stills aren't. The skill turns a recording into images, reasons about each in isolation, then assembles a chronological diagnosis. Three passes, each only firing if the prior left ambiguity:

- **Pass 1 — Baseline narration.** Extract at the chosen cadence (default: 1 frame every 2s). A Haiku describer reads all frames in one dispatch and reports observation only — verbatim text, colors, deltas. Unreadable text is marked `[unreadable]`, not guessed.
- **Pass 2 — Targeted densification.** If a state changed between two adjacent frames but neither captures the transition (e.g. between t=15s and t=20s the modal title flipped and a toast appeared), re-extract only that window at higher rate (say, 5 frames between 15–20s) and re-dispatch the describer on those new frames. The rest of the video isn't reprocessed.
- **Pass 3 — Targeted zooms.** For regions the describer marked `[unreadable]` (a 14px badge, an icon-only button), crop the region with ffmpeg and re-dispatch a short OCR read on just the crop.

The orchestrator then synthesizes: **observation cited per frame**, **inference labeled separately**.

Full process in [SKILL.md](./SKILL.md).

## Example output

For an 8-second screen recording showing a generic "Publish article" flow where the user clicks Publish, the skill produces something like:

> **Observation (Frame 01, t=0s)**:
> - Modal "Publish article": title field **"How to brew espresso"**, status pill **"Draft"** (gray), button **"Publish"** blue and enabled.
> - Header pill (top-right): **"3 unsaved changes"** (amber, ~80px wide).
>
> **Observation (Frame 03, t=2s)** — click registered:
> - Button label changed: **"Publish"** → **"Publishing…"** (blue, with spinner).
> - Header pill unchanged: **"3 unsaved changes"**.
>
> **Observation (Frame 06, t=5s)** — response landed:
> - Toast bottom-right (green): **"Article published"**.
> - Modal closed.
> - Header pill still reads: **"3 unsaved changes"** (unchanged — `[verified by zoom on the badge crop]`).
>
> **Inference (labeled separately)**:
> The publish API call succeeded (toast confirms). The header pill counter did not reset after publish — it should have dropped to 0 since the unsaved changes are now persisted. Likely a stale derived state in the header component; the toast and modal handler updated their local stores but didn't invalidate the unsaved-changes counter.

What makes the output useful:
- **Verbatim quoting** of every string that drives the diagnosis (`"3 unsaved changes"`, `"Publishing…"`, `"Article published"`) — no paraphrasing, so you can `grep` directly against your codebase.
- **Frame-numbered citations** (`Frame 03`, `Frame 06`) so you can replay the recording if you disagree.
- **Observation vs inference labeled explicitly** — what's in the pixels vs what it means.

The skill arrives at this through a baseline narration pass (Haiku on all frames in one dispatch), followed by targeted crops on regions the describer marked `[unreadable]` — a header pill is typically ~14px tall and needs a zoom to OCR. The crops resolve the verbatim text in a single extra round-trip.

## License

MIT.
