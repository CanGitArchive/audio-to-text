# Changelog

## Initial release

Single-file PyQt6 app for splitting multi-track recordings into speaker-labeled transcripts. Pipeline: ffmpeg extracts each role's audio into 16 kHz mono WAV chunks (default 540 s); OpenAI transcribes the combined track with diarization and the solo tracks with prompt-supported transcription; per-chunk speaker resolution stitches the diarized output back to "Speaker A" / "Speaker B" labels by cross-referencing against the solo tracks; backchannel filtering and same-speaker turn merging produce the final readable timeline.

### The per-chunk speaker-resolution algorithm

The headline engineering bit, and the one whose design notes are worth keeping.

OpenAI's `gpt-4o-transcribe-diarize` model assigns A/B/C speaker labels **independently per audio chunk**, starting fresh at "A" each time. A natural first implementation, give every raw label a global ID and use it across the whole session, mis-attributes turns at every chunk boundary, because "A" in chunk 1 may correspond to a different person than "A" in chunk 2.

The fix is to resolve labels **per chunk** by cross-referencing each raw label's text against the solo Speaker A and Speaker B transcripts for the same chunk. Group the diarized segments by raw label within a chunk, concatenate the text per label, tokenize, and pick whichever solo's vocabulary the label's tokens overlap with most. That much is the natural approach.

The question is which similarity metric to use. **Symmetric Jaccard** (intersection over union) is the textbook answer and works for most chunks, but it has a sharp failure mode on this data: when one solo is sparse for a chunk (one participant silent, the other talking at length), the larger solo's bigger token pool accumulates more spurious matches via common filler words. Jaccard then "wins" toward the wrong speaker, the larger solo gets the label even when the content clearly belongs to the smaller solo's speaker.

**Containment-via-smaller** fixes this:

```
score(label, solo) = |label_tokens ∩ solo_tokens| / min(|label_tokens|, |solo_tokens|)
```

Normalizing by the smaller of the two sets removes the asymmetric-size advantage. A label that contains most of a sparse solo's vocabulary is correctly identified as that speaker even when the other solo is much larger. Paired with a 1.3× margin gate and a 5-token minimum per label, labels that genuinely can't be resolved fall through to a session-stable `S1/S2/S3` numbering instead of being confidently mis-attributed.

On a seed 49-minute two-participant recording: 454 raw diarizer fragments → 189 readable Speaker A / Speaker B turns after backchannel filtering and same-speaker turn merging, with only 1-2 short single-word responses left as `S*` for downstream review.

### Other notes worth keeping

- **Reprocess mode is first-class.** Because resolution logic evolves, the pipeline can re-render outputs from cached `raw_json/*_segments.json` without re-spending OpenAI API credits. This forced the output-rendering code to live in a single function (`build_session_outputs`) instead of being entangled with the transcription loop, both the live OpenAI flow and the cache-only reprocess flow funnel through it, so changes to output shape or resolution logic apply to both paths automatically.
- **WAV not MP3 for chunks.** Avoids compression artifacts on the model side.
- **540-second chunks.** Fits comfortably under OpenAI's per-request size limit for 16 kHz mono PCM (~17 MB).
- **Fallback mix synthesis.** If the recording doesn't contain a real combined track, ffmpeg `amix` synthesizes one from the selected solos, so the pipeline doesn't break on weird recording setups.
- **Single-file architecture.** ~1200 lines of `AudioToText.py`; the whole pipeline fits in one mental model.
