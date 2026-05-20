<div align="center">

<img src=".github/banner.png" alt="Sparkpod" width="800"/>

# Sparkpod

**Turn long-form podcasts into short-form clips. Locally. With your taste.**

[![License: MIT](https://img.shields.io/badge/license-MIT-FF5500.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-FF8C00.svg)](https://www.python.org/downloads/)
[![ffmpeg](https://img.shields.io/badge/ffmpeg-required-0A0A0A.svg)](https://ffmpeg.org/)
[![Status](https://img.shields.io/badge/status-public%20preview-FF8C00.svg)](#status)

[Quick start](#quick-start) · [How it works](#how-it-works) · [Roadmap](#roadmap) · [Known issues](#known-issues) · [Contributing](CONTRIBUTING.md)

</div>

---

Drop in a podcast YouTube URL. Sparkpod gives you back a folder of vertical clips with burned-in word-highlighted captions, ready to upload. No cloud, no monthly fee, no UI to log into.

It does this by chaining together best-in-class open source pieces — yt-dlp, faster-whisper, Gemini, OpenCV, ffmpeg — into an opinionated pipeline that handles the parts that usually break: silence trimming, speaker-aware crops, and choosing actually-clippable moments instead of arbitrary 60-second windows.

## Quick start

```bash
git clone https://github.com/<you>/sparkpod.git
cd sparkpod
pip install -r requirements.txt
cp .env.example .env       # then edit: add GEMINI_API_KEY
python ignite_clip.py run --url "https://www.youtube.com/watch?v=..."
```

Output lands in `data/clips/<id>/clip_NN_final.mp4` with a sibling `clip_NN.json` for metadata. A consolidated `upload_queue.json` ranks everything by virality score.

System requirements: `ffmpeg` on PATH and a Gemini API key (free tier is plenty for testing). See [Setup](#setup) below for the full path including YouTube cookies and font installation.

## What you get

For a 2-hour podcast, Sparkpod produces ~12 short-form clips with:

- **Speaker-aware 9:16 crop** — face detection drives a smooth pan that follows whoever's talking. No more chopping host or guest out of frame.
- **Word-highlighted captions** — bold display font, active word highlighted in accent color, sized for mobile. Karaoke-style.
- **Silence-trimmed audio** — uses two strategies (whisper word-timestamps OR waveform energy, switchable) to remove dead air without sounding clipped.
- **Loudness-normalized** to broadcast standard (-14 LUFS) so volume's consistent across platforms.
- **Hook detection** — clips start on the actual hook line, not arbitrary timestamps.
- **Per-platform metadata** — generates platform-tuned titles, captions, and hashtag sets for YouTube Shorts, TikTok, Instagram Reels, and Facebook Reels.

Each clip ships with a JSON sidecar carrying the virality score breakdown, source attribution, and the raw transcript — the same data the ranker used, available for your own tooling.

## How it works

```
podcast URL
   │
   ▼
[ ingest ]      yt-dlp pulls audio only (~60-120 MB for a 2hr show), gates on duration
   │
   ▼
[ transcribe ]  faster-whisper with word-level timestamps
   │
   ▼
[ evaluate ]    sentence-window chunks → Gemini Flash scores on 6 dimensions
   │            (hook, clarity, emotional payload, surprise, completeness, quotability)
   ▼
[ cut ]         yt-dlp --download-sections grabs video for ONLY the winners
   │            precise ffmpeg trim with hook-line offset
   ▼
[ polish ]      silence trim (transcript or waveform mode) + loudness normalize
   │
   ▼
[ crop ]        Haar face detection + EMA smoothing + dwell guard → 9:16 pan
   │
   ▼
[ caption ]     captacity with word-highlighted ASS subtitles
   │            (patched to honor position, fix descender clipping)
   ▼
[ frame ]       attribution top bar (PIL-rendered, ffmpeg overlay)
   │
   ▼
[ metadata ]    Gemini call per clip → YT/TT/IG/FB titles + hashtags
   │
   ▼
upload_queue.json + final mp4s
```

Each stage is independently re-runnable. Re-render clips with a new font without re-downloading. Re-evaluate with a tuned rubric in seconds. Stages cache to disk.

The scoring rubric lives in `config/rubric.json` — tune weights or add dimensions without touching code.

## Setup

### 1. System dependencies

```bash
# Linux / macOS
sudo apt install ffmpeg            # or: brew install ffmpeg

# Windows
# Download from https://www.gyan.dev/ffmpeg/builds/ and add to PATH
```

### 2. Python

Python 3.10 or newer. `pip install -r requirements.txt` from the repo root.

### 3. API key

Get a free Gemini API key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey). Paste into `.env`:

```env
GEMINI_API_KEY=...
```

### 4. YouTube cookies (if running on a server)

YouTube challenges datacenter IPs as bots. Solution: export cookies from a browser on a residential connection.

- Install the "Get cookies.txt LOCALLY" browser extension.
- Visit youtube.com (logged in to a *throwaway* Google account — not your main one).
- Export → `cookies.txt`. Place at the repo root.
- Set `YOUTUBE_COOKIES=cookies.txt` in `.env`.

Skip this if running from a home/residential IP.

### 5. Fonts

The repo includes three SIL OFL display fonts (Bangers, Knewave, Poetsen One) in `fonts/`. Switch between them via `CAPTION_FONT` in `.env`.

## Usage

### One-shot CLI

```bash
python ignite_clip.py run --url "<URL>" --top 12
```

### Interactive shell

```bash
python app.py
sparkpod> process https://www.youtube.com/watch?v=...
sparkpod> list
sparkpod> show 1               # inspect ranked clips for source #1
sparkpod> render 1             # re-render with new caption / crop config
```

### Individual stages

Re-run any stage without re-running the whole pipeline:

```bash
python ignite_clip.py evaluate --source <sid> --top 20
python ignite_clip.py render   --source <sid>
```

## Configuration

All config lives in `.env`. The interesting knobs:

| Variable | What it does |
|---|---|
| `CANVAS_PRESET` | `vertical` (9:16), `square` (1:1), or `wide` (16:9) |
| `SILENCE_TRIM_MODE` | `transcript` (smart, uses whisper words) or `waveform` (cuts more) |
| `MAX_PAUSE_SECONDS` | Pauses longer than this get trimmed (transcript mode) |
| `CAPTION_FONT` | `Bangers`, `Knewave`, `Poetsen One`, or a path |
| `CAPTION_POSITION` | `("center", 0.78)` puts captions at lower-third |
| `CROP_SMOOTH_FRAMES` | Higher = smoother speaker-tracking pans |
| `DEFAULT_TOP_N` | How many clips to keep per source |
| `WHISPER_MODEL` | `small` for CPU, `large-v3` if you have a GPU |
| `WHISPER_COMPUTE_TYPE` | `int8` on CPU, `float16` on GPU |

See `.env.example` for the full list with defaults and inline comments.

## Status

**What works today**

- End-to-end pipeline from URL to final captioned clip
- Speaker-aware 9:16 crop with smooth panning
- Two silence-trim strategies (transcript-mode and waveform-mode)
- Word-highlighted captions with three bundled display fonts
- Per-platform metadata generation (YouTube / TikTok / Instagram / Facebook)
- Scheduled `upload_queue.json` output ready for manual or tool-assisted posting
- Interactive shell, one-shot CLI, and optional FastAPI server

**Not built yet**

- Direct posting integrations to YouTube / TikTok / IG / Facebook (use Postiz, Metricool, or post manually — `upload_queue.json` carries everything you need)
- 1:1 square crop variant fully tested across platforms
- Caption A/B variants
- Custom brand frame templates beyond the included IGNITE-style attribution bar
- Background music / dynamic ducking

## Known issues

- **YouTube datacenter IP challenge.** If running on a cloud VM, you'll hit "Sign in to confirm you're not a bot" without cookies. Solved by following step 4 above.
- **Caption block spacing.** With `line_count=2` captions, a small visual gap can appear between lines on certain fonts. Workaround: `line_count=1` in `caption.py`.
- **Whisper on CPU is slow.** Transcribing a 2-hour podcast takes 15-25 minutes on a modern desktop CPU. Use `small` model and `int8` compute type for the best CPU performance.
- **Auto-editor cuts more aggressively than transcript mode.** If waveform-mode trim feels jumpy, raise `SILENCE_THRESHOLD_DB` (less aggressive) or switch to `transcript` mode.

## Roadmap

- **v0.3** — Crop A/B variant batching (9:16 vs 1:1 in one pass), background music layer with ducking
- **v0.4** — Direct platform posting (YouTube first, then IG/FB via Graph API), Web UI for queue management
- **v0.5** — Multi-source batch processing with smart deduplication across episodes
- **v1.0** — Stable API, plugin architecture for custom frame templates and scoring rubrics

## Contributing

Yes please. See [CONTRIBUTING.md](CONTRIBUTING.md). Issues, PRs, and clip-ranking-rubric tweaks all welcome.

## Acknowledgements

Sparkpod stands on the shoulders of [yt-dlp](https://github.com/yt-dlp/yt-dlp), [faster-whisper](https://github.com/SYSTRAN/faster-whisper), [captacity](https://github.com/unconv/captacity), [auto-editor](https://github.com/WyattBlue/auto-editor), [moviepy](https://github.com/Zulko/moviepy), and [Google Gemini](https://ai.google.dev/). Fonts are licensed under [SIL OFL](https://openfontlicense.org/).

Built by [IGNITE Co.](https://github.com/<you>) as an open-source companion to a broader content automation toolkit.

## License

MIT — see [LICENSE](LICENSE). Free for any use, including commercial. If you ship a fork, attribution is appreciated but not required.

---

<div align="center">

⭐ **If Sparkpod saves you time, drop a star.** It's the only way I know if this is useful to anyone but me.

</div>
