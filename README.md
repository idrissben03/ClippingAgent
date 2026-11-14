# Clip Agent

Turn multi-hour livestreams into short vertical clips. Plain code finds and
renders the moments, an LLM ranks them and writes the hooks, and **you approve
every clip** in a small Streamlit dashboard before anything is posted.

> **Status:** work in progress. See [ROADMAP.md](ROADMAP.md) for the phase plan
> and what is built so far.

<!-- Add a demo GIF of the dashboard here once Phase 6 is done:
![Dashboard demo](docs/demo.gif) -->

## Why this exists

Clipping campaigns (for example on Whop Content Rewards) pay per 1,000 views on
approved clips. Finding good moments in a 6-hour VOD, cutting, cropping and
captioning takes 20–40 minutes per clip by hand. This project automates the
mechanical work and leaves one human decision: *is this clip good enough to post?*

It is also a portfolio project on LLM integration and system design.

## How it works

```
VOD link
  1. Download video + chat          yt-dlp, TwitchDownloaderCLI
  2. Find candidate moments         chat-activity spikes, audio peaks
  3. Transcribe the candidates      faster-whisper
  4. Score moments, write hooks     LLM (structured JSON output)
  5. Render 9:16 clips + captions   FFmpeg
  6. Review                         Streamlit dashboard: approve / reject / edit
  7. Post                           Instagram Reels API (YouTube, TikTok later)
  8. Track views and metrics        SQLite
```

**Design choice:** steps 1–3, 5, 7 and 8 are a deterministic workflow, which is
cheaper, more predictable and easier to test. The LLM is used only in step 4,
where judgment is needed (is this funny? what is the hook? is it a spoiler?).

## Tech stack

| Area | Tool |
|---|---|
| Language | Python 3.11+ |
| Download | `yt-dlp`, TwitchDownloaderCLI |
| Video | FFmpeg |
| Transcription | `faster-whisper` |
| LLM | Anthropic API (small model, structured output) |
| Storage | SQLite |
| GUI | Streamlit |
| Tests | pytest |

## Getting started

### Prerequisites

- Python 3.11+
- [FFmpeg](https://ffmpeg.org/download.html) on your `PATH`
- [TwitchDownloaderCLI](https://github.com/lay295/TwitchDownloader/releases)
- An LLM API key

### Install

```bash
git clone <your-repo-url> clip-agent
cd clip-agent
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### Configure

Create a `.env` file in the project root (it is gitignored):

```
ANTHROPIC_API_KEY=your-key-here
LLM_MODEL=your-model-id
```

Add the streamers you are allowed to clip, with their face-cam crop
coordinates, to `config/streamers.toml`.

### Run

```bash
python -m clipagent.pipeline <VOD_URL>   # find, score and render clips
streamlit run app.py                     # review dashboard
pytest -q                                # tests
```

## Project layout

```
clipagent/
  download.py     VOD and chat download
  detect.py       chat spikes and audio peaks -> candidate windows
  transcribe.py   faster-whisper on candidate windows only
  score.py        LLM scoring, hooks, captions
  render.py       FFmpeg cut, 9:16 layout, burned-in captions
  db.py           SQLite schema and queries
  pipeline.py     runs the steps in order
  post/           platform posting (Instagram, YouTube)
app.py            Streamlit review dashboard
config/           streamer and campaign settings
tests/
data/             gitignored: raw downloads, clips, database
```

## Rules this project follows

- **A human approves every clip.** There is no fully automatic posting mode.
- **Only clip sources that allow it.** Footage comes from streamers listed in
  `config/streamers.toml` whose campaigns invite clipping.
- **No spoilers.** The LLM flags story reveals, endings and isolated cutscenes,
  and flagged candidates are dropped. Rockstar's
  [video policy](https://support.rockstargames.com/articles/200153756/rockstar-games-video-policy)
  allows takedowns of this material, and its fan-footage permission is for
  non-commercial use, so takedowns are a real risk for paid clips.
- **No fake engagement.** Official platform APIs or audited schedulers only.
- **Disclose paid content** with the platform's paid-partnership setting where
  required.

## Metrics

Logged from day one so the results are measurable:

| Metric | What it shows |
|---|---|
| Approval rate | How often the LLM's picks are good enough to post |
| Minutes per published clip | Time saved vs the manual baseline |
| LLM cost per clip | Whether the pipeline is cheap to run |
| Views per clip / earnings per campaign | Whether the side hustle pays |

Results will be added here as they come in.

## Design decisions

- **Workflow first, agent second.** Fixed steps are plain code. The LLM handles
  only judgment calls.
- **Chat spikes as the main signal.** Viewers react a few seconds after a great
  moment, so each window starts about 25 seconds before the spike.
- **Transcribe only candidates.** Never the whole VOD, to keep it fast and cheap.
- **Human in the loop.** Protects clip quality and approval rate, and platform
  posting rules (such as TikTok's) require the user to see a preview and consent.
- **SQLite, not Postgres.** One file is enough at this scale. A job queue,
  Postgres and object storage are the next step at 100x volume.

## Roadmap

Phases 0–10 are tracked in [ROADMAP.md](ROADMAP.md): setup, download, moment
detection, transcription, LLM scoring, rendering, dashboard, account setup and
posting, metrics, and polish.

## License

TBD. Add a license before making the repository public.

## Disclaimer

Not affiliated with Rockstar Games, Take-Two, Twitch, Whop, or any social
platform. Clip only content you have permission to use, and follow each
platform's and campaign's rules.
