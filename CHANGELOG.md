# Changelog (fork)

Changes this fork makes on top of [OpenShorts](https://github.com/mutonby/openshorts).
Upstream's own changes are in its history, which this repository keeps.
Versions follow [Semantic Versioning](https://semver.org/); dates are UTC.

## [Unreleased]

### Fixed
- Dockerfile: the Deno `COPY --from` uses a digest-only reference, which
  Modal's image builder accepts (it refuses tag plus digest). Same image.
- gpu_modal: default Modal names are `openshorts-oss-*`, so a deploy cannot
  replace an existing app called `openshorts-gpu` in the same workspace.

### Added
- gpu_modal: `MAX_SOURCE_MINUTES` as a run setting.
- gpu_modal/README.md: a run measured with this repository (A10, GPU
  transcription included): 3 clips in 111 s, about $0.027 of Modal.

## [0.1.0] - 2026-11-19

Based on upstream `main` at "feat(billing): connecting social accounts is a
paid feature" (2026-11-18).

### Added
- Output aspect ratios 4:5, 16:9 and any W:H, with caption and hook sizing
  per ratio (`OUTPUT_ASPECT`, `main.py --aspect`; `output_aspect.py`).
- Per-run caption look: `CAPTION_PRESET`, `CAPTION_HIGHLIGHT`, `CAPTION_FONT`,
  and a script font chosen from the transcript's language.
- Audio-first partial download for YouTube links (`AUDIO_FIRST=1`,
  `main.py --audio-first`; `audio_first.py`).
- Groq transcription backend with local faster-whisper fallback
  (`TRANSCRIBE_BACKEND=groq`; `groq_asr.py`).
- Per-call Gemini cost meter (`GEMINI_USAGE_FILE`; `usage_meter.py`).
- `main.py --clips FILE` to reuse an earlier run's picks.
- Optional Modal GPU backend with CPU fallback (`gpu_modal/`).
- Tests for every addition (`tests/test_fork_*.py`).

### Changed
- Captions: no spaces between Chinese or Japanese words, CJK counted as
  double width in line budgets, ASS Encoding -1 so libass applies bidi to
  whole Arabic and Hebrew lines; the image installs `fonts-noto-cjk`.
- Gemini stage calls retry up to 8 times (waits capped at 40 s) and alternate
  with `GEMINI_FALLBACK_MODELS` on transient errors.

### Removed
- Upstream's `cloud/` directory (OpenShorts Commercial License), from the
  whole history. `app.py` imports from it are optional, `queue_mode_off.py`
  stands in for the queue policy, and tests that need it are skipped.
- The workflow that publishes upstream's CLI to PyPI.

[0.1.0]: https://github.com/xspring1982/openshorts.io/releases/tag/v0.1.0
