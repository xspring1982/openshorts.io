# OpenShorts.io — open source AI video clipper

Turn long videos (podcasts, talks, interviews, streams) into vertical short
clips with face tracking, animated captions and a hook line: a free,
MIT-licensed alternative to OpusClip.

**Two ways to use it:**

- **Online, no install:** [openshorts.io](https://openshorts.io). Paste a
  YouTube link and get captioned 9:16 Shorts. The free plan covers
  15 minutes of video a month, no card needed; paid plans start at
  $11.90 a month ([pricing](https://openshorts.io/pricing)). There is also a
  hosted [API and MCP server](https://openshorts.io/api) for Claude, Cursor
  and other agents.
- **Self-hosted, free:** this repository, with Docker. No watermark, no
  minute cap; you bring a Gemini API key (see [Quick start](#quick-start-docker)).

This is the OpenShorts.io open source repository: the self-hostable
clipping pipeline behind [OpenShorts.io](https://openshorts.io), maintained
as a fork of
[OpenShorts](https://github.com/mutonby/openshorts) by mutonby, which is
MIT-licensed. It keeps the whole upstream application:
the clip generator, the dashboard, AI Shorts, YouTube Studio, the MCP server
and the API. On top of that it adds the generic improvements we built while
running the pipeline in production at [OpenShorts.io](https://openshorts.io):

| Change | What it does | Turn it on |
|---|---|---|
| **Aspect ratios** | 4:5 and 16:9 (or any W:H) besides 9:16 and 1:1, with captions and the hook sized for each frame | `OUTPUT_ASPECT=4:5` or `main.py --aspect 4:5` |
| **Caption look per run** | Any of upstream's caption presets, your active-word colour, your font | `CAPTION_PRESET=hormozi`, `CAPTION_HIGHLIGHT=#00E0FF`, `CAPTION_FONT=...` |
| **CJK and right-to-left captions** | No spaces between Chinese or Japanese words, CJK counted as double width, bidi for Arabic and Hebrew, a matching Noto font per language | automatic |
| **Audio-first download** | YouTube links fetch only the audio, pick the moments, then fetch only those ranges of the video | `AUDIO_FIRST=1` or `main.py --audio-first` |
| **Groq transcription** | Hosted Whisper large-v3-turbo in seconds, with local faster-whisper as the fallback | `TRANSCRIBE_BACKEND=groq`, `GROQ_API_KEY` |
| **Gemini resilience** | About three minutes of retries during "503 high demand", alternating with a fallback model | on by default (`GEMINI_MAX_ATTEMPTS`, `GEMINI_FALLBACK_MODELS`) |
| **LLM cost meter** | One JSON line per Gemini call with tokens and cost | `GEMINI_USAGE_FILE=/path/usage.jsonl` |
| **Reuse picks** | Re-render the same moments (for example at another ratio) without a new LLM call | `main.py --clips old_metadata.json` |
| **Modal GPU backend** | Optional serverless rendering on L4/A10, billed per second, with a CPU fallback | [`gpu_modal/`](gpu_modal/README.md) |

Each change is a separate commit on top of upstream, with tests. See
[CHANGELOG.md](CHANGELOG.md).

> **Hosted version:** if you would rather not run it yourself,
> [OpenShorts.io](https://openshorts.io) runs this pipeline on GPUs. Paste a
> YouTube, TikTok, Instagram or Facebook link, or use its API and MCP server.

## Quick start (Docker)

You need Docker with Docker Compose and a
[Gemini API key](https://aistudio.google.com/app/apikey). You can instead point
the moment picker at a local OpenAI-compatible LLM (see below).

```bash
git clone https://github.com/xspring1982/openshorts.io.git openshorts && cd openshorts
cp .env.example .env          # optional: S3 backup, GPU, the options above
docker compose up --build
```

Open <http://localhost:5175>, enter your Gemini key in **Settings**, and upload
a video or paste a link in the **Clip Generator**. AI Shorts, YouTube Studio
and social posting need their own keys (fal.ai, ElevenLabs, Upload-Post), as
upstream documents.

### From the command line

The pipeline is `main.py`, and you can run it directly inside the backend
container:

```bash
docker exec -it openshorts-backend python main.py -i /app/uploads/talk.mp4 -o /app/output/talk \
  --aspect 4:5
docker exec -it -e AUDIO_FIRST=1 -e CAPTION_PRESET=pill openshorts-backend \
  python main.py -u "https://www.youtube.com/watch?v=..." -o /app/output/yt
```

Each clip lands in the output folder as `<title>_clip_N.mp4`, together with
`<title>_metadata.json`. The metadata holds each clip's range, title, hook,
TikTok and Instagram descriptions, a predicted score and the reason for it.

### GPU

On an NVIDIA machine, build with `GPU=1` and give the container the card, as
described in upstream's
[GPU section](https://github.com/mutonby/openshorts#5-gpu-acceleration-optional-nvidia).
An 8-minute video then clips in about a minute instead of 5 to 8. Without a GPU
of your own, use the Modal backend:

```bash
pip install modal && modal token new
modal secret create openshorts GEMINI_API_KEY=<key>
modal deploy -m gpu_modal.app
python -m gpu_modal render talk.mp4 -o clips/ --aspect 4:5
```

Measured on the hosted service: a 10-minute source took 111 s on an L4 for 6
clips, which cost $0.034. One production hour averaged **$0.022 per job**.
Costs, limits and the A10 numbers are in
[gpu_modal/README.md](gpu_modal/README.md).

### Local LLM instead of Gemini

Set `LLM_BASE_URL` (Ollama, vLLM, LM Studio...) and `LLM_MODEL`. The
transcript then never leaves your machine. Gemini is then needed only for
layout picking and for silent videos.

## Configuration

Everything upstream documents in `.env.example` still applies. The fork adds:

| Variable | Default | |
|---|---|---|
| `OUTPUT_ASPECT` | `9:16` | output frame; `CAPTION_SCALE`, `CAPTION_CHARS`, `CAPTION_MARGIN_V`, `HOOK_WIDTH` override the per-ratio caption sizing |
| `CAPTION_PRESET` | `default` | `hormozi`, `pill`, `lime`, `oneword`, `cine`, `clean` |
| `CAPTION_HIGHLIGHT` | preset colour | `#RRGGBB` |
| `CAPTION_FONT` | per language | any installed font |
| `AUDIO_FIRST` | off | `1` for YouTube links; `SECTION_PAD` (2 s) and `SECTION_PARALLEL` (6) tune it |
| `TRANSCRIBE_BACKEND` | `whisper` | `groq` (needs `GROQ_API_KEY`), or upstream's `parakeet` |
| `GEMINI_MAX_ATTEMPTS` | 8 | tries per Gemini stage call |
| `GEMINI_FALLBACK_MODELS` | `gemini-flash-lite-latest` | comma-separated, `none` to disable |
| `GEMINI_USAGE_FILE` | unset | path of the per-call cost log |

The dashboard does not show the new options yet. Set them in `.env` for the
backend, or per run on the command line.

## Limits and differences from upstream

- **No `cloud/` directory.** Upstream's billing, accounts and hosted-service
  code (`cloud/`) is under a separate commercial license, so this fork does
  not include it, in any commit. The rest runs without it: two imports in
  `app.py` became optional, `queue_mode_off.py` stands in for the queue
  policy, and the tests that need `cloud/` are skipped. `BILLING_ENABLED`
  mode does not work here.
- **Audio-first jobs keep only the picked ranges.** The clip editor's
  re-render of other parts of the video reports the source as gone for those
  jobs.
- **The new options are not in the dashboard.** They are environment
  variables and command-line flags only.
- **YouTube downloads** depend on your IP, cookies and yt-dlp's current
  state, as upstream. Datacenter IPs, Modal's included, are often blocked.

## Staying close to upstream

This fork tracks upstream's `main`. To make updates easy to merge, the fork's
changes are small, marked with `Fork:` comments, and kept in new modules where
possible (`output_aspect.py`, `audio_first.py`, `groq_asr.py`,
`usage_meter.py`, `queue_mode_off.py`, `gpu_modal/`). See
[CONTRIBUTING.md](CONTRIBUTING.md) for the sync procedure.

## Development

```bash
pip install pytest pillow httpx pydantic sqlalchemy greenlet numpy opencv-python-headless \
  python-dotenv google-genai fastapi boto3 PyJWT email-validator python-multipart asyncpg
pytest tests/ -q
```

With the full CPU stack installed (`requirements-cpu.lock` plus torch), the
tests that import `main.py` run as well.

## License and credit

MIT, see [LICENSE](LICENSE) and [NOTICE](NOTICE). OpenShorts is by mutonby
and its contributors; the original project, its documentation and its hosted
service live at [github.com/mutonby/openshorts](https://github.com/mutonby/openshorts).
The full upstream history is kept in this repository, minus `cloud/`. This
fork is maintained by the OpenShorts.io team. It is not affiliated with or
endorsed by the upstream project.
