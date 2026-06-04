# AudioToText

Splits a multi-track audio recording (e.g. an OBS `.mkv` where each speaker has their own track) into clean, speaker-labeled transcripts. The interesting bit is a per-chunk speaker-resolution step that fixes a sharp diarization failure mode, [see CHANGELOG](CHANGELOG.md) for the algorithm.

## Features

- **Multi-track audio split**: ffmpeg pulls each track out into ~9-minute WAV chunks (auto-synthesizes a combined mix if the recording doesn't have one).
- **Speaker-labeled timeline**: diarizer output on the combined track is stitched to "Speaker A" / "Speaker B" by cross-referencing the solo tracks per chunk.
- **Cache-based reprocess**: re-render the output transcripts from cached responses without re-spending API credits.
- **AI cleanup bundle**: one file (cleanup prompt + all transcripts) ready to paste into any LLM for final wording polish.

## Tech stack

Python 3.11+, PyQt6, OpenAI Audio Transcription API, ffmpeg/ffprobe.

## Install + run

```powershell
pip install -r requirements.txt
python AudioToText.py
```

Needs `ffmpeg`/`ffprobe` on `PATH` (or dropped into `DATA/TOOLS/`) and `DATA/AudioToText/.env` with `OPENAI_API_KEY=...`.

GUI flow: scan a recording's audio tracks → assign roles (Combined / Mixed, Speaker A, Speaker B, Ignore) → extract → transcribe. "Run Full Pipeline" does both steps end-to-end; "Reprocess From Raw JSON" rebuilds outputs from cached responses.

Outputs land under `DATA/AudioToText/Extracts/<session>/transcripts/`: per-speaker solos, a chronological combined timeline, and an AI cleanup bundle.

See [CHANGELOG.md](CHANGELOG.md) for the design notes.
