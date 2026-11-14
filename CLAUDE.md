# Clip Agent

Python pipeline that turns long livestream VODs into short vertical clips. Plain
code finds and renders candidate moments, an LLM ranks them and writes hooks,
and a human approves each clip in a Streamlit dashboard before it is posted.

The phased plan and current progress live in `ROADMAP.md`. Read only the phase
being worked on (search for its heading), and tick its checkboxes when a step is
finished.

## Owner and goals

- Solo project by a software engineering student. Strong in Java Spring Boot,
  testing, and system design. Comfortable with Python, new to video tooling.
- Goal 1: side hustle through paid clipping campaigns (Whop Content Rewards),
  starting with GTA VI streamer content.
- Goal 2: portfolio project for internships. Clean code, tests, and measurable
  results matter more than features.
- This is a learning project. See "How to work with me" below.

## Stack (do not add dependencies without asking)

- Python 3.11+, virtual environment in `.venv/`
- `yt-dlp` and TwitchDownloaderCLI for VOD and chat download
- FFmpeg, called through `subprocess.run` with argument lists (never `shell=True`)
- `faster-whisper` for transcription (model `small`, CPU, `int8`)
- Anthropic Python SDK for the LLM step. Model name comes from the `LLM_MODEL`
  env var. Default to a small, cheap model. Check https://docs.claude.com for
  current model IDs before changing it.
- SQLite via the standard `sqlite3` module. No ORM.
- Streamlit for the GUI. No JavaScript frontend.
- `pytest` for tests, `python-dotenv` for secrets, `tomllib` for config

Not used on purpose: n8n, LangChain or other agent frameworks, FastAPI, Docker,
Postgres, Celery. Suggest them only if the roadmap reaches a point that needs them.

## Architecture

```
download -> detect -> transcribe -> score (LLM) -> render -> review (human) -> post -> track
```

- Deterministic steps are a plain workflow. The LLM is called only in `score.py`.
- Each step is a pure-ish function: reads from disk/DB, writes to disk/DB, can
  be re-run safely. Skip work whose output already exists.
- Steps communicate through the SQLite database and files under `data/`, not
  through in-memory state.
- Posting happens only for clips with `status = 'approved'`, and only as a
  direct result of the user pressing Approve.

## Repo layout

```
clipagent/
  download.py     yt-dlp + chat download
  detect.py       chat spikes and audio peaks -> candidate windows
  transcribe.py   faster-whisper on candidate windows only
  score.py        LLM scoring, hooks, captions (structured JSON output)
  render.py       FFmpeg: cut, 9:16 layout, burned-in captions
  db.py           SQLite schema and queries
  pipeline.py     runs the steps in order
  post/           instagram.py, youtube.py (later phases)
app.py            Streamlit review dashboard
config/streamers.toml   per-streamer crop coordinates and campaign rules
tests/
data/             gitignored: raw/<vod_id>/, clips/, clipagent.db
```

## Commands

```bash
source .venv/bin/activate              # Windows: .venv\Scripts\activate
python -m clipagent.pipeline <VOD_URL> # run the full pipeline on one VOD
streamlit run app.py                   # review dashboard
pytest -q                              # tests
```

## Data model (SQLite)

- `vods(id, url, streamer, downloaded_at)`
- `candidates(id, vod_id, start_s, end_s, spike_score, transcript)`
- `clips(id, candidate_id, llm_score, llm_reason, hook_title, caption, path,
  status, decided_at)` with `status` in `pending | approved | rejected | posted`
- `posts(id, clip_id, platform, url, posted_at, views, views_updated_at)`
- `llm_calls(id, candidate_id, model, input_tokens, output_tokens, cost_usd, created_at)`

## Conventions

- Type hints on all functions. Small functions. `pathlib.Path` for paths.
- Use `logging`, not `print`, inside `clipagent/`.
- All tunable numbers (bucket size, spike threshold, padding, clip length
  limits) live in one config module, not scattered as magic numbers.
- Times are floats in seconds from the start of the VOD. Caption files use
  times relative to the clip start.
- Every new pure function in `detect.py`, the caption writer, and `db.py` gets
  a pytest test. Tests must not hit the network, the LLM API, or real video
  files larger than a few seconds.
- Secrets only in `.env`. Never commit `.env`, tokens, or anything in `data/`.

## LLM step rules

- Send text only: transcript, a sample of chat messages, spike stats. Never
  send video or the full VOD transcript.
- Always request structured output with a JSON schema and validate it. On
  invalid output, retry once, then mark the candidate as failed.
- Required fields: `score` (1–10), `reason`, `hook_title`, `caption`,
  `trim_start`, `trim_end`, `flags`.
- If the optional agent loop is built (tools like `get_transcript`,
  `get_chat`), cap it at 5 tool calls per candidate.
- Record tokens and cost for every call in `llm_calls`.
- Keep each call small: at most 20 candidates per VOD (pre-filter by spike
  score), transcript trimmed to about 1,500 characters, at most 30 chat
  messages, and a low `max_tokens` (about 400).

## Hard rules (never break, even if asked casually)

- A human approves every clip. Do not build fully automatic posting, and do not
  add a setting that bypasses the Approve step.
- No fake engagement: no view bots, no multi-account automation, no scraping
  or posting through unofficial or reverse-engineered platform endpoints. Use
  official posting APIs or an audited third-party scheduler.
- Only process footage from sources listed in `config/streamers.toml` (streamers
  whose campaigns permit clipping).
- Drop candidates the LLM flags as `spoiler`. No endings, isolated cutscenes,
  or story reveals. No leaked or pre-release footage.
- Do not add watermarks or branding to clips.

## How to work with me

- Follow `ROADMAP.md` phase order. Do not jump ahead to later phases.
- Build one small step at a time, then stop so I can run it and read it. Do not
  generate the whole project in one go.
- When you use a library or FFmpeg flag for the first time, explain it in 2–3
  lines: what it does and why it was chosen.
- When there is a design choice, give me the trade-off in a sentence or two and
  a recommendation. I need to be able to explain these decisions in interviews.
- Prefer the simplest thing that works. Flag over-engineering instead of doing it.
- If a command fails or a tool is missing, say so plainly and show the error.

## Token budget

Keep context small. Brevity applies to reading and wording, never to
correctness: still run the tests and verify the change works.

**Reading**

- Never read or print files under `data/`, `.venv/`, or `__pycache__/`. Chat
  JSON, transcripts, and logs are huge. To inspect one, print counts and a
  small sample (`head -c 2000`, `wc -l`, or a 5-line Python snippet).
- Never open media or binary files (`.mp4`, `.wav`, `.db`). Query the database
  with `sqlite3 data/clipagent.db "SELECT ... LIMIT 5"`.
- Search first, then read only the function or line range needed. Do not read
  whole files to "get oriented"; the repo layout above is accurate.
- Do not re-read a file already read in this session unless it changed.
- Open only the files the current task touches.

**Commands**

- Run quietly: `pytest -q --tb=short` (add `-x` when debugging),
  `ffmpeg -hide_banner -loglevel error`, `yt-dlp --quiet --no-progress`,
  `pip install -q`.
- Pipe any long output through `tail -n 30`.
- Test on the 20-minute sample VOD or tiny fixtures in `tests/fixtures/`. Never
  run the pipeline on a full VOD just to check a change.
- Do not run Whisper or call the LLM API unless the task is about that step.

**Writing**

- Make targeted edits. Do not rewrite a whole file for a small change.
- Do not paste code back into chat after writing it. Report in 5 lines or
  fewer: what changed, how to run it, what to check.
- No preamble, no restating the request, no recap of things that did not change.
- Explain a new library or flag once, in 2–3 lines. Do not repeat it later.

**Scope**

- One roadmap step per request. If a request spans several, do the first and stop.
- If a task needs more than about 5 files or a refactor, give a 3-line plan and
  wait for a go-ahead.
- After two failed attempts at the same fix, stop and report what was tried.
- No subagents or parallel agents unless asked.

## Metrics to keep working

These are the portfolio results. Do not remove the data that feeds them.

- Approval rate: approved clips / clips proposed by the LLM
- Minutes of human time per published clip (vs manual baseline)
- LLM cost per clip
- Views per clip, earnings per campaign
