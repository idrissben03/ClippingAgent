# Clip Agent — Roadmap

A pipeline that turns long livestream VODs into short vertical clips, with an LLM
picking the moments and a human (you) approving them before they are posted.

**Two goals:** a side hustle through paid clipping campaigns, and a portfolio
project for internship applications.

**How to use this file:** work top to bottom. One "evening" is about 2 hours.
Every phase ends with a *Done when* check. Tick the boxes as you go.

---

## The system in one picture

```
VOD link
  -> 1. Download video + chat          (plain code)
  -> 2. Find candidate moments         (plain code: chat spikes, audio peaks)
  -> 3. Transcribe the candidates      (plain code: Whisper)
  -> 4. Score moments, write hooks     (LLM  <- the "agent" part)
  -> 5. Render 9:16 clips + captions   (plain code: FFmpeg)
  -> 6. Review dashboard               (you: approve / reject / edit caption)
  -> 7. Post                           (manual first, then automated)
  -> 8. Track views and metrics        (plain code)
```

Design rule to remember for interviews: fixed steps are a **workflow** (cheap,
predictable, testable). The LLM is used only where **judgment** is needed.

---

## Read this first: rules that protect the accounts

- [ ] **Only clip footage you are allowed to use.** That means streamers whose
      campaign explicitly invites clipping, ideally from the source links the
      campaign gives you.
- [ ] **Rockstar's video policy.** Its permission for fan footage is written for
      non-commercial use, and paid clips are commercial. Treat takedowns as a
      real risk. Streamer-reaction clips (face cam + commentary) are safer than
      raw gameplay.
- [ ] **No spoilers.** Rockstar removes endings, isolated cutscenes, cutscene
      compilations, big story reveals, and in-game TV shows posted on their own.
      Never use leaked or pre-release footage.
- [ ] **No fake engagement.** No view bots, no networks of fake accounts.
- [ ] **Disclose paid content** with the platform's paid-partnership setting
      when the campaign or platform requires it.
- [ ] **You approve every clip.** No fully automatic posting.
- [ ] **Never pay to join a campaign.** A "campaign" that asks for money is a scam.

---

## Timeline at a glance

GTA VI launches **Thursday Nov 19, 2026**. GTA VI VODs do not exist before then,
so build and test the pipeline on any existing VOD from a streamer who runs a
clipping campaign.

| When | What |
|---|---|
| Weekend of Nov 14 | Phase 0 (setup) + start Phase 7 account creation |
| Week 1 evenings | Phases 1–3: download, detect, transcribe |
| Launch week | Clip 3–5 by hand to get your baseline numbers |
| Week 2 evenings | Phases 4–6: LLM scoring, rendering, dashboard |
| Week 3 | Phase 7 finished, first pipeline clips posted manually |
| Week 4+ | Phase 8 (auto-posting), Phase 9 (metrics), Phase 10 (portfolio) |

---

## Phase 0 — Setup (1 hour)

- [ ] Install **Python 3.11+**, **Git**, and **FFmpeg**
      (Windows: `winget install ffmpeg` · Mac: `brew install ffmpeg`).
- [ ] Download **TwitchDownloaderCLI** from its GitHub releases page.
- [ ] Create the repo and virtual environment:

```bash
mkdir clip-agent && cd clip-agent && git init
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install yt-dlp faster-whisper anthropic streamlit python-dotenv requests pytest
pip freeze > requirements.txt
```

- [ ] Put `CLAUDE.md` and this file in the repo root.
- [ ] Create `.gitignore` containing `.venv/`, `.env`, `data/`.
- [ ] Get an LLM API key, save it in `.env` as `ANTHROPIC_API_KEY=...`, and set
      a monthly spend limit in the provider console.

**Done when:** `ffmpeg -version` and `yt-dlp --version` both print a version,
and a 5-line Python script gets a reply from the LLM API.

---

## Phase 1 — Download a VOD and its chat (evening 1)

- [ ] Pick one VOD. Start with a 20-minute slice so tests are fast:

```bash
yt-dlp --download-sections "*00:30:00-00:50:00" -o "data/raw/%(id)s/video.%(ext)s" <VOD_URL>
```

- [ ] Download the chat replay as JSON:

```bash
./TwitchDownloaderCLI chatdownload --id <VOD_ID> -o data/raw/<VOD_ID>/chat.json
```

- [ ] Wrap both in `clipagent/download.py` using `subprocess.run`.

Note: Twitch's official API does not give you VOD chat. TwitchDownloader uses
the same endpoints the Twitch website uses, so it can break when Twitch changes
things. Phase 2 has a backup signal for that case.

**Done when:** `data/raw/<vod_id>/` contains `video.mp4` and `chat.json`.

---

## Phase 2 — Find candidate moments (evening 2)

Idea: when something great happens, chat explodes a few seconds later.

- [ ] Load `chat.json` and read each message's time offset in seconds.
- [ ] Count messages in 10-second buckets.
- [ ] Score each bucket: `count / (rolling_median + 1)`. Keep buckets above ~3.
- [ ] Merge spikes that are close together.
- [ ] Turn each spike into a window: start **25 s before** the spike (chat
      reacts late), end 10 s after.
- [ ] Backup signal for VODs without chat: loudness peaks from the audio track.
- [ ] Save the top 20 windows to `candidates.json`.
- [ ] Write `pytest` tests with fake chat data (a flat chat gives no candidates,
      one burst gives one candidate).

**Done when:** you open the top 5 timestamps in a video player and at least 2
are moments you would clip yourself.

---

## Phase 3 — Transcribe the candidates (evening 3)

- [ ] Cut each candidate's audio: `ffmpeg -ss <start> -t <length> -i video.mp4 -vn -ac 1 -ar 16000 cand_01.wav`
- [ ] Transcribe only those windows, never the whole VOD:

```python
from faster_whisper import WhisperModel
model = WhisperModel("small", device="cpu", compute_type="int8")
segments, info = model.transcribe("cand_01.wav", word_timestamps=True)
```

- [ ] Save text plus word timestamps per candidate. The word timestamps become
      your captions in Phase 5.

**Done when:** every candidate has a readable transcript on disk.

---

## Phase 4 — LLM scoring (evening 4)

- [ ] For each candidate, send the LLM: the transcript, 20–30 sample chat
      messages, and the spike score.
- [ ] Force a structured JSON answer (use tool use / a JSON schema):
      `score` 1–10, `reason`, `hook_title`, `caption`, `trim_start`, `trim_end`,
      `flags` (`spoiler`, `needs_context`, `not_funny`).
- [ ] Drop anything flagged `spoiler`. Keep the top 5 by score.
- [ ] Use a small, cheap model. Log tokens and cost per VOD.
- [ ] Save every LLM response to the database.

**Stretch (4b), the real agent loop:** give the LLM tools such as
`get_transcript(start, end)` and `get_chat(start, end)` so it can look around a
moment and choose better clip boundaries itself. Cap it at 5 tool calls per
candidate.

**Done when:** one command prints a ranked top 5 with titles and reasons.

---

## Phase 5 — Render clips (evenings 5–6)

- [ ] Write an `.srt` caption file from the word timestamps. Times must be
      relative to the clip start, not the VOD start.
- [ ] Render the simplest version first (centre crop to 9:16, captions burned in):

```bash
ffmpeg -ss <start> -t <length> -i video.mp4 \
  -vf "crop=ih*9/16:ih,scale=1080:1920,subtitles=clip.srt" \
  -c:v libx264 -crf 20 -preset veryfast -c:a aac out.mp4
```

- [ ] Upgrade: stacked layout with the streamer's face cam on top and gameplay
      below (two `crop` filters joined with `vstack`). Store each streamer's
      crop coordinates in `config/streamers.toml`.
- [ ] Keep clips between 15 and 60 seconds.

Tip for Windows: run FFmpeg from the folder that contains the `.srt` file. Full
Windows paths inside the `subtitles=` filter need awkward escaping.

**Done when:** 5 clips play correctly on your phone, vertical, with captions.

---

## Phase 6 — Review dashboard (evening 7)

- [ ] Create a SQLite database with tables `vods`, `candidates`, `clips`, `posts`.
- [ ] Build `app.py` with Streamlit: for each pending clip show the video
      (`st.video`), the LLM score and reason, an editable caption
      (`st.text_area`), and **Approve** / **Reject** buttons.
- [ ] Save every decision with a timestamp. This is your approval-rate metric.
- [ ] Run it with `streamlit run app.py`.

**Done when:** you can review a day's clips in under 15 minutes.

---

## Phase 7 — Accounts and first posts (start this on day 1, in parallel)

- [ ] Create a free **Whop** account and open **Content Rewards**.
- [ ] Pick 1–2 campaigns. Write down for each: pay per 1,000 views, allowed
      platforms, required tags or disclosure, minimum views, payout terms.
- [ ] Create dedicated accounts: a **YouTube** channel, an **Instagram** account
      switched to Creator or Business (needed for the API later), and **TikTok**.
- [ ] Create a **Google Cloud project**, enable **YouTube Data API v3**, and
      request the compliance audit early. It takes weeks and uploads stay
      private until it passes.
- [ ] Launch week: clip 3–5 by hand. Record minutes per clip and views. This is
      your "before" baseline.
- [ ] Post your first approved pipeline clips **manually** and submit the links
      to the campaign.

**Done when:** one clip is submitted to a campaign and counted.

---

## Phase 8 — Automate posting (week 4+)

Do them in this order.

- [ ] **8a. Instagram (your own account, no app review needed).**
      Meta developer account -> create an app -> add the Instagram product ->
      add your account as an **Instagram Tester** -> accept the invite in
      Instagram settings -> generate a long-lived token (refresh every ~60 days).
      Publishing is three calls: create a container with `media_type=REELS`,
      poll its status until `FINISHED`, then call `media_publish`.
      Gotcha: Meta fetches the video from a **public URL**, so the file must be
      hosted somewhere reachable (or use the resumable upload flow if your app
      type supports it).
- [ ] **8b. YouTube.** OAuth 2.0 login + `videos.insert`. Works only after the
      audit passes. Until then, upload manually.
- [ ] **8c. TikTok.** Its audit expects apps built for a wide audience, so a
      personal tool may be refused. Keep TikTok manual.
- [ ] **Alternative for all three:** call a paid scheduler API that has already
      passed the audits (for example Blotato, Ayrshare, or Postiz).
- [ ] The Approve button in the dashboard is the only thing that triggers a post.

**Done when:** approving a clip in the dashboard publishes it to Instagram.

---

## Phase 9 — Tracking and metrics

- [ ] Fetch view counts daily and store them in `posts`.
- [ ] Add a stats page to the dashboard with:
  - approval rate (approved / proposed by the LLM)
  - minutes of your time per published clip (vs the manual baseline)
  - LLM cost per clip
  - views per clip, and earnings per campaign
- [ ] Use the numbers to tune the detector and the scoring prompt.

**Done when:** you can state one before/after number, for example "approval
rate went from 30% to 60% after adding chat spikes."

---

## Phase 10 — Portfolio polish

- [ ] `README.md`: what it does, architecture diagram, how to run it, results.
- [ ] A 20-second demo GIF of the dashboard.
- [ ] Tests for the detector, the caption writer, and the database layer.
- [ ] A "Design decisions" section: workflow vs agent, why a human approves,
      what you would change at 100x scale (job queue, Postgres, object storage).
- [ ] CV bullet: *"Built a Python pipeline that detects highlights in multi-hour
      livestreams from chat-activity and transcript signals, uses an LLM to rank
      moments and write hooks, renders vertical clips with FFmpeg, and publishes
      through platform APIs after human approval; cut editing time per clip by X%."*

---

## Tools you are new to

| Tool | What it is | Used for | Learn first |
|---|---|---|---|
| **venv + pip** | Python's isolated environment and package installer (like Maven dependencies per project) | Keeping this project's packages separate | `python -m venv`, `pip install`, `requirements.txt` |
| **yt-dlp** | Command-line video downloader | Getting VODs | `--download-sections`, `-o` output templates |
| **TwitchDownloaderCLI** | Open-source Twitch VOD and chat downloader | Chat replay as JSON | `chatdownload --id` |
| **FFmpeg** | Command-line video and audio editor | Cutting, cropping, captions | `-ss`, `-t`, `-vf`, the `crop`, `scale`, `subtitles` filters |
| **Whisper / faster-whisper** | Speech-to-text model that runs on your laptop; faster-whisper is a quicker implementation | Transcripts and caption timing | model sizes (`tiny`, `small`), `word_timestamps=True` |
| **LLM API + tool use** | Calling a model from code; tool use lets it return structured data or call your functions | Scoring moments, writing hooks | messages format, JSON schema, the agent loop |
| **python-dotenv / `.env`** | Loads secrets from a local file that is never committed | API keys and tokens | `load_dotenv()`, `os.environ` |
| **SQLite** | A database stored in one file, built into Python (`sqlite3`) | Clips, decisions, posts, metrics | `connect`, `execute`, parameterised queries |
| **Streamlit** | Turns a Python script into a web UI, no HTML or JS | Review dashboard | `st.video`, `st.button`, `st.text_area`, `st.session_state` |
| **pytest** | Python's test framework (the JUnit equivalent) | Testing the detector and helpers | `assert`, fixtures, `pytest -q` |
| **OAuth 2.0** | How an app gets permission to act on your account | YouTube and Instagram posting | access vs refresh tokens, scopes |
| **Whop Content Rewards** | Marketplace where streamers and brands pay clippers per view | The side-hustle income | campaign rules, pay per 1,000 views, approval rate |

---

## References

- Rockstar video policy: https://support.rockstargames.com/articles/200153756/rockstar-games-video-policy
- TwitchDownloader: https://github.com/lay295/TwitchDownloader
- Instagram content publishing: https://developers.facebook.com/docs/instagram-platform/content-publishing/
- YouTube `videos.insert` (audit note): https://developers.google.com/youtube/v3/docs/videos/insert
- TikTok content sharing guidelines: https://developers.tiktok.com/doc/content-sharing-guidelines
- Whop Content Rewards docs: https://docs.whop.com/manage-your-business/growth-marketing/content-rewards
- Claude API docs (models, tool use, pricing): https://docs.claude.com/en/api/overview
