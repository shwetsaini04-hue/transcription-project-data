# Implementation Prompt — Whisper Parameter Sweep for Human Judging

> Hand everything below the line to the coding model. Fill the `<<FILL IN>>` markers first.

---

## What you are building

A script that transcribes a small set of call recordings under **15 different Whisper decoding configurations**, then writes the results out so a **human can read them side by side and pick the best setting by eye**.

There is no automatic scoring. No WER. No gold transcripts. No metrics library. The human is the evaluator. Your only job is to produce clean, comparable, blind-labelled transcripts and make them easy to read.

Do not build a pipeline. Do not add features nobody asked for. Write the script, run it, produce the files.

## Environment (fixed)

- AWS EMR Studio notebook, kernel `Python (kunj_audio_env)`, Python 3.9
- HuggingFace `transformers`, `pipeline("automatic-speech-recognition")`
- Model: `Oriserve/Whisper-Hindi2Hinglish-Apex` (~800M params, Whisper architecture, fine-tuned on ~1000h of Indian conversational and call-centre audio). Already unpacked at:
  `/opt/models/whisper-hindi2hinglish-apex/whisper-hindi2hinglish-apex/`
- **This model outputs Romanized Hinglish** — Hindi words in Latin script, not Devanagari. Any Devanagari appearing in output is a signal something is wrong.
- Audio: outbound personal-loan telecalling recordings, WAV, **5–15 minutes each**, read from S3 via `boto3` / `s3fs`
- Audio path: `<<FILL IN: S3 prefix or local directory>>`
- Compute: `<<FILL IN: GPU type, or "CPU only">>`
- `transformers` version: `<<FILL IN — print transformers.__version__>>`

## Current production code — this is Config A, the control

```python
def transcribe_wav_file(local_wav_path):
    audio, sample_rate = read_wav_as_float32(local_wav_path, target_sample_rate=16000)
    result = pipe({
        "array": audio,
        "sampling_rate": sample_rate
    },
        chunk_length_s=10,
        stride_length_s=3,
        batch_size=1,
        return_timestamps=False,
        generate_kwargs={
            "task": "transcribe",
            "condition_on_prev_tokens": False,
            "temperature": 0.0,
            "no_repeat_ngram_size": 4,
            "compression_ratio_threshold": 2.4,
            "logprob_threshold": -1.0,
            "no_speech_threshold": 0.6
        })
    return result["text"].strip()
```

## Step 0 — Two checks before you run anything

1. **Print `generation_config.json` and `config.json`** from the model directory. Report: `forced_decoder_ids`, `lang_to_id`, `suppress_tokens`, `num_beams`, `max_length`. If the checkpoint already forces a language token, say so — it changes whether the Round 2 language configs are meaningful.

2. **Set `return_timestamps=True` for every config, including the control.** This is required for the excerpt alignment below, and sequential long-form mode will not run without it. Note in your output that the control now differs from production in this one respect.

Also fix `batch_size` at the largest value that fits memory (try 8 or 16) and **hold it constant across all configs**. It affects speed only, never output, but varying it makes runtimes incomparable.

---

## ROUND 1 — Windowing (6 configs)

Run these first. The human picks a winner before Round 2 is run.

Everything not listed stays at the control's values.

| Label | `chunk_length_s` | `stride_length_s` | Other changes |
|---|---|---|---|
| **A** | 10 | 3 | none — this is current production |
| **B** | 20 | 4 | none |
| **C** | 30 | 5 | none |
| **D** | 30 | 3 | none |
| **E** | — | — | **sequential long-form**: omit `chunk_length_s` and `stride_length_s` entirely, pass the full audio array, `condition_on_prev_tokens: False`, `temperature: 0.0` |
| **F** | — | — | **sequential long-form**: `condition_on_prev_tokens: True`, `temperature: (0.0, 0.2, 0.4, 0.6, 0.8, 1.0)`, keep all three thresholds |

**Why this matters and what to expect:** with `chunk_length_s=10, stride_length_s=3`, HF discards 3 seconds from each side of every chunk, so each forward pass contributes only ~4 seconds of new text — while the encoder still pads every chunk to the 30 seconds it was trained on. A 10-minute call becomes ~150 forward passes with heavy sentence fragmentation. At 30/5 the same call takes ~30 passes. Report the actual chunk count and wall-clock time per config so this is visible.

F is deliberately a bundle rather than a single-variable arm — it is Whisper's native long-form recipe as designed. Label it as such.

---

## ROUND 2 — Everything else (9 configs)

**Do not run Round 2 until the human has named a Round 1 winner.** That winner becomes `BASE`. Every config below is `BASE` plus exactly one change, except O.

| Label | Change from BASE |
|---|---|
| **G** | `language: "en"` |
| **H** | `language: "hi"` |
| **I** | `no_repeat_ngram_size: 0` |
| **J** | `no_repeat_ngram_size: 6` |
| **K** | `num_beams: 5` |
| **L** | `temperature: (0.0, 0.2, 0.4, 0.6, 0.8, 1.0)` with the three thresholds present |
| **M** | domain-vocabulary priming via `prompt_ids` (see below) |
| **N** | `num_beams: 5` **and** `no_repeat_ngram_size: 0` — beam search instead of n-gram blocking |
| **O** | combined candidate: language pinned + `num_beams: 5` + temperature ladder + `no_repeat_ngram_size: 0` |

**Notes you must act on:**

- **If BASE is a chunked config, L may produce output byte-identical to BASE.** The compression-ratio, logprob, and no-speech thresholds only fire inside Whisper's temperature-fallback loop, which the chunked path largely bypasses. If the outputs match exactly, **report that as a finding** — it means the thresholds currently in production are decorative. Do not quietly present a null result as a real one.
- **If Step 0 showed the checkpoint forces a language token,** G and H may be no-ops. Check and report.
- **`no_repeat_ngram_size: 4` suppresses legitimate repetition.** In telecalling, agents repeat digits to confirm ("double five, double five") and customers backchannel ("haan haan haan ji"). I and J exist to find out whether the guard is costing more than it saves.
- **For M**, build the prompt string from these terms and pass via `processor.get_prompt_ids(...)`:
  `pre-approved, top-up, foreclosure, part-payment, processing fee, ROI, tenure, EMI, disbursal, sanction letter, NACH, e-mandate, CIBIL, bureau, KYC, PAN, Aadhaar, salary account, lakh, crore` plus `<<FILL IN: your product/scheme names>>`. Keep it under 200 tokens. If `prompt_ids` is not supported in the chunked path in this transformers version, say so and skip M rather than faking it.

---

## Evaluation audio

Use **6 to 8 calls**, chosen deliberately, not at random:
- 2 clean-line, normal-pace
- 2 noisy or poor-line
- 2 heavy code-switching or fast speaker
- 1–2 that contain rate/EMI/tenure discussion

More calls than this is counterproductive — the bottleneck is human reading time, not compute.

---

## Output format — this is the part that determines whether the experiment is usable

### The reading-load problem

A 12-minute call transcribes to roughly 2,000 words. Fifteen versions is 30,000 words per call. Nobody can compare that. So:

**Extract aligned excerpts, don't stack full transcripts.**

For each call, pick **4 time windows of 60–90 seconds each**:
1. **Opening** — agent greeting, product name, any compliance disclosure
2. **Numbers** — wherever rate, EMI, tenure, or amount is discussed (find this by scanning the control transcript for digits and for `lakh|hazaar|percent|EMI|tenure`)
3. **Difficult** — the noisiest or fastest stretch, or the highest code-switch density
4. **Close** — call wrap-up, next steps, consent

Windows are defined **by timestamp**, and each config's text for that window is sliced from its own timestamped segments. Every config gets the same wall-clock range, so the human is comparing the same speech.

### File layout

```
outputs/
  round1/
    call_003.md          <- 4 excerpts, each with all 6 versions stacked
    call_007.md
    ...
    _KEY.md              <- label -> config mapping. DO NOT open before judging.
    _runtimes.csv        <- label, call_id, wall_clock_s, n_chunks
    full/
      call_003__A.txt    <- complete transcripts, for spot-checking only
      ...
```

### Inside each call file

```markdown
# Call 003  (11m 42s)

## Excerpt 1 — Opening  [00:00 – 01:15]

### Version A
<text>

### Version B
<text>
...
```

### Blind labelling — required

Shuffle the config-to-letter mapping **per call**, using a fixed seed so it is reproducible. Version A in `call_003.md` must not be the same config as Version A in `call_007.md`. Write the mapping only to `_KEY.md`.

If the human can see `chunk_length_s=30` above a transcript, they will read it more favourably. Blind comparison costs nothing and makes the judgment worth having.

---

## Judging rubric — include this at the top of every call file

```markdown
For each version, note:

1. DROPPED SPEECH  - whole phrases or seconds missing vs the others?
2. LOOPS           - any phrase repeated 3+ times, or text that runs on nonsensically?
3. NUMBERS         - are amounts, rates, tenure, EMI correct and complete?
4. DOMAIN TERMS    - pre-approved / foreclosure / processing fee / CIBIL etc. rendered correctly?
5. SCRIPT          - any Devanagari, or Hindi wrongly translated into English? (both are failures)
6. PHANTOM TEXT    - words appearing over silence, hold music, or ringing?
7. READABILITY     - sentence boundaries sane, or fragments glued together?

Rate each version 1-5 overall. Note the single worst error you see.
```

---

## Deliverables

1. `sweep.py` — config definitions as a plain list of dicts, runner, and the excerpt/report writer. Cache raw output per (config, call) as JSON so nothing is ever re-transcribed.
2. The `outputs/` tree above.
3. `FINDINGS.md`, short, containing:
   - the Step 0 audit result (does the checkpoint force a language?)
   - runtime and chunk count per config — this makes the windowing cost visible
   - **any config that produced output identical to another config**, stated plainly
   - anything that errored or silently fell back, and what you did about it

## Constraints

- Same audio, same splits, same `batch_size` across every run. The config under test is the only thing that varies.
- Do not change the model checkpoint.
- Do not compute WER, CER, or any accuracy score. The human judges.
- Do not editorialise in the transcript files. No annotations, no highlighting of differences, no "this version looks better." Raw text only — anything you add biases the reader.
- If a config crashes or is unsupported in this transformers version, report it and continue. Do not substitute a different config silently.

## Before you write code

Print your Step 0 audit findings, your estimated total runtime for Round 1, and confirm the `<<FILL IN>>` values with me. Then write `sweep.py` in full — no placeholders, no `# TODO`.
