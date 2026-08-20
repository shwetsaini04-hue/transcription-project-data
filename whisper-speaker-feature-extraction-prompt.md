# Prompt: Extract Timing and Acoustic Features for Speaker Separation

Paste everything below the line into your coding chatbot, alongside your existing transcription code.

---

I have an existing Whisper transcription pipeline for two-speaker phone call recordings (a call center agent and a customer, single mixed mono channel). I want to modify it to extract timing and acoustic features that will later help me separate the two speakers. Do not implement speaker separation itself — only produce the features.

Modify my code to do the following.

## 1. Enable word-level timestamps

Run Whisper with `word_timestamps=True` so every word has a start and end time.

## 2. Rebuild turns from word gaps, ignoring Whisper's own segments

Whisper's segment boundaries are based on punctuation, not speakers. Discard them and rebuild turns myself:

- Walk through the full word list in order.
- Compute the gap between the end of each word and the start of the next.
- Start a new turn whenever the gap exceeds a threshold.
- Make the threshold a configurable constant at the top of the file, default `0.6` seconds.
- Assign each turn a sequential integer ID starting at 1, never reused or renumbered.

## 3. For each turn, compute and store

| Field | Description |
|---|---|
| `turn_id` | Sequential integer, starting at 1 |
| `start_time` | Seconds from start of audio |
| `end_time` | Seconds from start of audio |
| `duration` | `end_time - start_time` |
| `text` | Joined words for this turn |
| `gap_before` | Silence in seconds between previous turn's end and this turn's start; `null` for the first turn |
| `gap_after` | Silence until the next turn starts; `null` for the last turn |
| `word_count` | Number of words in the turn |
| `speech_rate` | Words per second over the turn's duration |
| `median_pitch` | Median fundamental frequency in Hz over this turn's audio span, via `librosa.pyin`; `null` if the turn is too short or extraction fails |
| `pitch_std` | Standard deviation of pitch values for the same span; `null` on failure |
| `mean_energy` | Mean RMS energy over the turn's audio span |
| `is_backchannel` | `true` when duration is under 0.8 seconds and word count is 3 or fewer |
| `avg_logprob` | From whichever Whisper segment the turn's words came from; if it spans multiple segments, take the mean weighted by word count |
| `no_speech_prob` | Same approach as above |
| `compression_ratio` | Same approach as above |
| `language` | Detected language of the source segment |

## 4. Store call-level metadata

- `audio_duration`
- `total_speech_time` — sum of all turn durations
- `total_silence_time` — audio duration minus total speech time
- `turn_count`
- `mean_gap` and `median_gap` across all turns
- `whisper_model` name and the pause threshold used

## 5. Output

Write one JSON file per call containing the call-level metadata and the ordered list of turns.

Keep the raw Whisper output saved separately and unmodified, so I can re-derive turns with a different threshold without re-transcribing.

## Requirements

- Load the audio **once** and reuse it for all pitch and energy computation. Do not re-read the file per turn.
- Wrap pitch extraction in `try/except`. A failure on one turn must not stop the run.
- Preserve my existing file handling and directory structure.
- Add clear comments explaining what each feature is for.
