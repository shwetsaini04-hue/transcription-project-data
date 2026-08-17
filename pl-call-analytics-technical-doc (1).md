# PL call transcript analytics — technical reference

A working document covering the technical components of the system, how each one
works, and what capability is needed to build it.

Scope: transcript in, structured insight out. Audio-to-text is upstream and
treated here as a contract we consume, not a component we build.

---

## 1. System shape

The system is a **batch pipeline with specialised LLM stages**, not a
conversational multi-agent system. Stages run in a fixed graph, each with its own
prompt, its own output schema, and its own evaluation.

Three layers that change at different speeds:

| Layer | Contents | Changes | Lives in |
|---|---|---|---|
| Pipeline | Code, orchestration, storage | Slowly | Git repo |
| Knowledge | Rules, product facts, taxonomies | Constantly | Markdown in git |
| Evaluation | Gold set, harness, metrics | Grows | DB + git |

Keeping these separate is the single most important structural decision. If
business rules live inside prompts, every rule change becomes an engineering
ticket.

---

## 2. Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Python 3.11+ | Ecosystem for NLP, LLM SDKs, data work |
| Schema/validation | Pydantic v2 | Runtime validation of every LLM output |
| Orchestration | asyncio + Postgres job table | No extra infra; sufficient at this volume |
| Queue (if scaling) | Redis + Arq, or Celery | Only when the Postgres job table strains |
| Database | PostgreSQL 15+ | Relational core, JSONB for flexible insight payloads |
| Vector search | pgvector extension | Clustering and similarity without a separate store |
| PII detection | Presidio + custom recognisers | Open source, extensible for Indian identifiers |
| Annotation | Label Studio | Supports span annotation, which we require |
| Review UI | FastAPI + HTMX, or React | Internal tool; simplicity beats sophistication |
| Metrics | scikit-learn, statsmodels | Precision/recall/F1, Cohen's kappa |
| Config | Pydantic Settings + YAML | Model routing and thresholds as config, not code |
| Observability | Structured JSON logs + Grafana | Per-stage cost, latency, failure rate |

**Deliberately not used:**

- **CrewAI / AutoGen** — dynamic agent delegation adds nondeterminism and cost to
  a workflow whose steps are known in advance.
- **LangChain** — abstraction overhead for what is fundamentally
  prompt-in/JSON-out. Direct SDK calls are easier to debug.
- **LangGraph** — reasonable if checkpointing and retry logic become painful to
  hand-roll. Not needed at the start.
- **Airflow** — designed for scheduled DAGs across systems, heavier than needed
  for per-call processing.

---

## 3. Data model

```sql
-- Core call record
CREATE TABLE calls (
  call_id            TEXT PRIMARY KEY,
  agent_id           TEXT NOT NULL,
  campaign           TEXT,
  lead_source        TEXT,
  direction          TEXT,
  dial_time          TIMESTAMPTZ NOT NULL,
  talk_duration_ms   INTEGER,
  dialer_disposition TEXT,
  triage_tier        SMALLINT NOT NULL,
  usability          TEXT,
  processing_status  TEXT NOT NULL DEFAULT 'pending',
  created_at         TIMESTAMPTZ DEFAULT now()
);

-- Normalised transcript, one row per turn
CREATE TABLE turns (
  call_id       TEXT REFERENCES calls(call_id),
  turn_id       INTEGER NOT NULL,
  speaker       TEXT NOT NULL,           -- 'agent' | 'customer' | 'unknown'
  channel       SMALLINT,
  start_ms      INTEGER NOT NULL,
  end_ms        INTEGER NOT NULL,
  text          TEXT NOT NULL,           -- redacted
  avg_confidence REAL,
  word_data     JSONB,                   -- word-level timings + confidence
  PRIMARY KEY (call_id, turn_id)
);

-- Deterministic metrics, computed not inferred
CREATE TABLE call_metrics (
  call_id             TEXT PRIMARY KEY REFERENCES calls(call_id),
  agent_talk_ms       INTEGER,
  customer_talk_ms    INTEGER,
  talk_ratio          REAL,
  longest_monologue_ms INTEGER,
  interruption_count  INTEGER,
  dead_air_ms         INTEGER,
  agent_wpm           REAL,
  question_count      INTEGER
);

-- One row per stage output per call
CREATE TABLE insights (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  call_id           TEXT REFERENCES calls(call_id),
  stage             TEXT NOT NULL,
  payload           JSONB NOT NULL,
  model             TEXT NOT NULL,
  prompt_version    TEXT NOT NULL,
  knowledge_version TEXT NOT NULL,
  input_tokens      INTEGER,
  output_tokens     INTEGER,
  created_at        TIMESTAMPTZ DEFAULT now(),
  UNIQUE (call_id, stage, prompt_version)
);

-- Evidence: which turns justify which claim
CREATE TABLE evidence (
  insight_id  UUID REFERENCES insights(id) ON DELETE CASCADE,
  label_path  TEXT NOT NULL,             -- e.g. 'objection_types[0]'
  turn_id     INTEGER NOT NULL,
  PRIMARY KEY (insight_id, label_path, turn_id)
);

-- Human labels: gold set and review outcomes
CREATE TABLE human_labels (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  call_id          TEXT REFERENCES calls(call_id),
  label_path       TEXT NOT NULL,
  value            JSONB NOT NULL,
  evidence_turns   INTEGER[],
  annotator        TEXT NOT NULL,
  source           TEXT NOT NULL,        -- 'gold' | 'review' | 'blind_review'
  codebook_version TEXT NOT NULL,
  split            TEXT,                 -- 'dev' | 'test' | NULL
  created_at       TIMESTAMPTZ DEFAULT now()
);

-- Reversible PII mapping, separate access control
CREATE TABLE pii_map (
  call_id     TEXT,
  placeholder TEXT,
  ciphertext  BYTEA NOT NULL,
  entity_type TEXT NOT NULL,
  PRIMARY KEY (call_id, placeholder)
);

CREATE INDEX ON insights (call_id, stage);
CREATE INDEX ON human_labels (call_id, source, split);
CREATE INDEX ON calls (dial_time, triage_tier);
```

Three design points worth noting:

**Idempotent writes.** The unique constraint on
`(call_id, stage, prompt_version)` means re-running a stage replaces rather than
duplicates. Calls get reprocessed many times; make that safe from day one.

**Provenance on every row.** Model, prompt version, and knowledge version are
stored with each insight. When flag rates shift in month six, this tells you
whether the model changed, the prompt changed, or the rules changed.

**PII map is a separate table** with its own grants, so ordinary pipeline roles
cannot read it.

---

## 4. Components

### 4.1 Ingest and normalisation

**Input contract from ASR** (see §8 for the full spec):

```json
{
  "call_id": "...",
  "provenance": { "asr_model": "...", "audio_hash": "..." },
  "audio": { "duration_ms": 214000, "recording_start_offset_ms": 3200 },
  "turns": [
    { "turn_id": 12, "speaker": "spk_0", "channel": 0,
      "start_ms": 41200, "end_ms": 47800,
      "text": "...", "avg_confidence": 0.91,
      "words": [{"w": "one", "s": 42100, "e": 42350, "conf": 0.97}] }
  ],
  "non_speech": [{"type": "hold", "start_ms": 91000, "end_ms": 118000}]
}
```

**Work performed:**

1. Trim everything before `recording_start_offset_ms` — dialer tones, IVR,
   recorded announcements.
2. Assign sequential `turn_id` values.
3. Resolve speaker roles. With dual-channel audio this is deterministic (channel
   0 = agent leg). With mono it requires a classification step over the first
   ~10 turns.
4. Validate against the Pydantic model; reject malformed transcripts to a
   dead-letter table rather than letting them poison downstream stages.

**Speaker role classification** (only needed for mono):

```python
class SpeakerRoles(BaseModel):
    agent_speaker: str
    customer_speaker: str
    confidence: float

# Small model, first 10 turns only.
# The agent opens, identifies the company, and asks questions.
```

Diarisation errors are silent and destructive — an agent's non-compliant claim
attributed to the customer disappears from your flags entirely. If dual-channel
recording is available from the dialer, take it. It is usually a configuration
flag and it removes this entire failure class.

### 4.2 PII redaction

**Two-pass approach:**

Pass one is regex, for identifiers with fixed formats. High precision, no model
needed:

```python
PATTERNS = {
    "PAN":     r"\b[A-Z]{5}[0-9]{4}[A-Z]\b",
    "AADHAAR": r"\b\d{4}\s?\d{4}\s?\d{4}\b",
    "PHONE":   r"\b(?:\+?91[\s-]?)?[6-9]\d{9}\b",
    "EMAIL":   r"\b[\w.+-]+@[\w-]+\.[\w.]+\b",
    "ACCOUNT": r"\b\d{9,18}\b",
}
```

Pass two is named-entity recognition for names, employers, and addresses, which
have no fixed format. Presidio with custom recognisers is the practical starting
point; expect to add an Indian-name gazetteer.

**Critical constraint — do not over-redact.** A naive redactor that masks all
numbers will destroy the values your compliance and fact-extraction stages
depend on:

| Must preserve | Must mask |
|---|---|
| `1.5%`, `18%` — interest rates | PAN, Aadhaar, account numbers |
| `18,000` — EMI amounts | Phone numbers |
| `5 lakh` — loan amounts | Exact salary figures |
| `36 months` — tenure | Date of birth |

Build a test suite of transcripts containing both categories and assert that
rates survive and identifiers do not. This test will catch more bugs than any
other in the project.

**Reversibility:**

```python
def redact(text: str, call_id: str) -> str:
    for entity in detect(text):
        placeholder = f"<{entity.type}_{next_index(call_id, entity.type)}>"
        store_encrypted(call_id, placeholder, entity.value)
        text = text.replace(entity.value, placeholder)
    return text
```

Destructive redaction cannot be undone, and you will eventually need the
original — for a dispute, an audit, or a CRM join.

**Validation:** take 50 redacted transcripts and have someone attempt to
identify the customer. Track the success rate. This is the only honest measure
of whether redaction works.

### 4.3 Triage router

Pure Python, no model. Sorts calls into processing tiers.

```python
def triage(call: Call, turns: list[Turn]) -> int:
    customer_turns = [t for t in turns if t.speaker == "customer"]
    customer_words = sum(len(t.text.split()) for t in customer_turns)

    if call.talk_duration_ms < 15_000:            return 0
    if len(customer_turns) < 2:                    return 0
    if customer_words < 15:                        return 0
    if matches_no_contact_patterns(turns):         return 0
    if customer_words < 60:                        return 1
    return 2
```

Tier definitions and treatment:

| Tier | Meaning | Processing |
|---|---|---|
| 0 | No contact | Rule-based disposition only, no LLM |
| 1 | Short refusal | One small-model call: refusal reason + red-flag scan |
| 2 | Substantive | Full pipeline |
| 3 | High risk | Tier 2 plus verifier and human review |

Tier 3 is not assigned by the router — it is a promotion applied after a
compliance flag fires, or by stratified random sampling for drift monitoring.

**Why this matters technically:** in outbound telecalling, typically 50–70% of
dials are non-conversations. Routing those away before any model call is the
largest single cost lever in the system, and it is difficult to retrofit once
downstream stages assume every call is substantive.

**Validation:** the router's tier assignments must be checked against the
human-labelled `usability` field. A router that pushes real conversations into
tier 0 loses them silently.

### 4.4 Deterministic metrics

Computed from timestamps. No model, no error, no cost.

```python
def compute_metrics(turns, non_speech) -> CallMetrics:
    agent_ms = sum(t.end_ms - t.start_ms for t in turns if t.speaker == "agent")
    cust_ms  = sum(t.end_ms - t.start_ms for t in turns if t.speaker == "customer")

    hold_spans = [(s["start_ms"], s["end_ms"])
                  for s in non_speech if s["type"] == "hold"]

    dead_air = 0
    for prev, curr in zip(turns, turns[1:]):
        gap = curr.start_ms - prev.end_ms
        if gap > 3000 and not overlaps_hold(prev.end_ms, curr.start_ms, hold_spans):
            dead_air += gap

    interruptions = sum(
        1 for prev, curr in zip(turns, turns[1:])
        if curr.start_ms < prev.end_ms
        and prev.speaker != curr.speaker
        and (prev.end_ms - prev.start_ms) > 2000
    )

    return CallMetrics(
        agent_talk_ms=agent_ms,
        customer_talk_ms=cust_ms,
        talk_ratio=agent_ms / max(agent_ms + cust_ms, 1),
        longest_monologue_ms=max_continuous_span(turns, "agent"),
        interruption_count=interruptions,
        dead_air_ms=dead_air,
        agent_wpm=word_count(turns, "agent") / max(agent_ms / 60_000, 0.01),
        question_count=count_questions(turns, "agent"),
    )
```

Two things this depends on: **timestamps** (insist on them from ASR even if you
are not using audio features yet) and **hold events from the dialer** (without
them, hold time reads as dead air and the metric is meaningless).

### 4.5 Segmentation

One model call that divides the transcript into phases as turn ranges.

```python
class Phase(BaseModel):
    name: Literal["opening", "pitch", "eligibility",
                  "objection", "close", "wrapup"]
    start_turn: int
    end_turn: int

class Segmentation(BaseModel):
    phases: list[Phase]
```

This is a force multiplier rather than an insight in itself. Once phases exist,
the objection stage reads the objection phase instead of 300 turns — lower cost,
higher accuracy, less distraction.

Model tier: small. Validation: against human-labelled phase boundaries, with
tolerance of ±1 turn. Exact precision is not required, because the purpose is
scoping rather than measurement.

### 4.6 Extraction stages

Every stage has the same shape. Build the pattern once.

```python
@dataclass
class StageSpec:
    name: str
    model: str                    # from config, not hardcoded
    output_schema: type[BaseModel]
    knowledge_files: list[str]
    phase_filter: list[str] | None
    few_shot_ids: list[str]
    requires_evidence: bool = True

async def run_stage(spec: StageSpec, call: Call, turns: list[Turn]) -> Insight:
    scoped = filter_by_phase(turns, spec.phase_filter)
    knowledge = load_knowledge(spec.knowledge_files)
    examples = load_examples(spec.few_shot_ids)

    prompt = render(spec.name, knowledge=knowledge,
                    examples=examples, turns=scoped)

    for attempt in range(2):
        raw = await call_model(spec.model, prompt,
                               response_schema=spec.output_schema)
        try:
            parsed = spec.output_schema.model_validate_json(raw)
            validate_turn_ids(parsed, scoped)
            return build_insight(spec, call, parsed, raw)
        except ValidationError as e:
            prompt = append_error(prompt, e)

    dead_letter(call.call_id, spec.name, "validation_failed")
```

**Schema enforcement.** Use the provider's structured-output or tool-use mode
rather than asking for JSON in prose. Then validate with Pydantic anyway —
structured output guarantees shape, not that `turn_id` values exist in the call.
`validate_turn_ids` is a real check that catches models inventing citations.

**Every label carries evidence:**

```python
class LabelWithEvidence(BaseModel):
    value: str
    evidence_turns: list[int] = Field(min_length=1)
    diffuse: bool = False          # judged holistically, not from specific turns

class ObjectionOutput(BaseModel):
    objections: list[LabelWithEvidence]
```

**Fact extraction differs from classification.** Nulls must be honest:

```python
class ExtractedFacts(BaseModel):
    stated_income: int | None = None
    existing_emi: int | None = None
    amount_sought: int | None = None
    tenure_months: int | None = None
    rate_quoted_by_agent: str | None = None
    evidence: dict[str, list[int]] = {}
    low_confidence_fields: list[str] = []
```

A model that invents an income figure is worse than one that returns null,
because the invented figure flows silently into lead scoring. Test this
explicitly with calls where the fact was never discussed.

Wire ASR confidence through: if the words backing a number had confidence below
a threshold, the field lands in `low_confidence_fields`.

**Model routing lives in config:**

```yaml
stages:
  segmentation:   { model: small,  verify: false }
  outcome:        { model: small,  verify: false }
  objections:     { model: small,  verify: true  }
  facts:          { model: small,  verify: true  }
  compliance:     { model: large,  verify: true  }
  pitch_quality:  { model: large,  verify: false }
```

Changing which model runs which stage must be a config edit, not a code change.
You will tune this repeatedly against the cost/accuracy trade-off.

### 4.7 Verifier

An independent check on whether cited turns actually support a claim.

```python
class VerifierResult(BaseModel):
    supported: bool
    confidence: float
    reason: str
    better_evidence_turns: list[int] | None = None
```

The verifier sees the claim, the cited turns, and the rule — but not the
extracting stage's reasoning. Independence is the point; a verifier that sees
the original chain of thought will rationalise it.

**Two things it checks:**

*Positive claims* — do the cited turns say what the claim says? Example: a
`rate_too_high` objection citing a turn where the **agent** stated the rate is
unsupported, because the rule requires the objection to come from the customer.

*Absence claims* — "the agent never disclosed APR" cannot be verified from the
cited turn alone. The verifier must scan the full transcript for the thing
alleged to be missing. If APR appears at turn 47, the flag is a false positive
and dies before reaching a reviewer.

**Where to apply it:** compliance flags, and any label where the model has a
strong prior it could guess from. Skip it where being wrong is cheap. It roughly
doubles the cost of any stage it guards.

**Monitor the drop rate per stage.** If the objection stage has 30% of its claims
rejected, that stage has a problem invisible in its accuracy score. Also log
every rejection — a verifier that is too aggressive silently eats true
positives, and only inspection reveals that.

### 4.8 Merge and reconciliation

Assembly is plumbing. The contradiction checks are the substance.

```python
CONTRADICTION_RULES = [
    Rule(
        name="converted_but_hard_refusal",
        check=lambda r: r.outcome == "converted"
                        and "hard_refusal" in r.objection_types,
        action="flag_for_review",
    ),
    Rule(
        name="compliance_rate_but_no_fact",
        check=lambda r: "rate_stated_without_apr" in r.compliance_flags
                        and r.facts.rate_quoted_by_agent is None,
        action="rerun_facts_on_cited_turn",
    ),
    Rule(
        name="emi_exceeds_income",
        check=lambda r: r.facts.existing_emi
                        and r.facts.stated_income
                        and r.facts.existing_emi > r.facts.stated_income,
        action="mark_low_confidence",
    ),
]
```

Write these as explicit rules rather than asking a model to "sort it out" —
you need to know which contradictions occur and how often.

**The contradiction rate is a production metric.** It requires no ground truth,
so it works on calls that were never in your gold set. A jump from 2% to 12%
after a prompt change is a real signal that something broke.

### 4.9 Evaluation harness

One command, run constantly.

```
$ python -m eval.run --split dev --stages outcome,objections

stage       label              n    P     R     F1    evid_IoU  Δ F1
outcome     converted         38  0.89  0.82  0.85    0.71     +0.04
outcome     callback          51  0.76  0.81  0.78    0.68     -0.01
objections  rate_too_high     44  0.81  0.73  0.77    0.55     +0.11
objections  emi_burden        29  0.62  0.51  0.56    0.48     +0.02

12 calls changed since prompt_version=v3
  → newly correct: 88213, 77401, 91002 ...
  → newly wrong:   80114, 82290 ...
```

**Metrics implemented:**

- **Precision, recall, F1** per label value, not just overall accuracy. Overall
  accuracy hides failure on rare classes, which are usually the ones that matter.
- **Evidence IoU** — `|predicted ∩ gold| / |predicted ∪ gold|` over turn IDs.
  This is what separates reasoning from guessing. A stage at F1 0.85 with IoU
  0.40 is pattern-matching and will fail on distribution shift.
- **Cohen's kappa** for annotator agreement during labelling, and for
  model-versus-human agreement on subjective labels where chance agreement is
  high.
- **Exact and normalised match** for extracted facts, plus a separate null-recall
  figure: of facts genuinely absent, how many were correctly returned as null.

**The diff view matters as much as the numbers.** Aggregates tell you something
changed; the list of flipped calls tells you what. That is what you iterate on.

**Dev and test are separated in code**, not by convention. The harness should
refuse to run against test without an explicit flag, and should log every test
run. Tuning against the test set destroys your only honest measurement.

### 4.10 Review interface

Requirements, technically stated:

- Queue ordered by priority: high-severity compliance flags, then
  model-versus-dialer-disposition disagreements, then stratified random sample.
- Claim rendered with cited turns extracted, not buried in the full transcript.
- Audio playback seeking to `turn.start_ms`. Requires the review service to hold
  a signed URL to the original recording.
- ASR confidence surfaced inline on low-confidence spans.
- For absence-based rules, an explicit statement of the range searched.
- Confirm / reject / unsure with keyboard shortcuts.
- Reject reasons as a fixed enum:
  `wrong_label | wrong_evidence | ambiguous_rule | asr_error`
- Blind mode on a configurable fraction of the queue, stored with
  `source='blind_review'`.
- PII unmasking behind a permission check, with access logged.

The reject enum is the highest-value element. Each value routes to a different
fix: prompt, verifier, codebook, or upstream ASR. Without it, you cannot tell
which of the four you are dealing with.

### 4.11 Orchestration

A Postgres-backed job table is sufficient at this volume and avoids a second
piece of infrastructure.

```sql
CREATE TABLE jobs (
  id          BIGSERIAL PRIMARY KEY,
  call_id     TEXT NOT NULL,
  stage       TEXT NOT NULL,
  status      TEXT NOT NULL DEFAULT 'queued',
  attempts    SMALLINT DEFAULT 0,
  last_error  TEXT,
  locked_by   TEXT,
  locked_at   TIMESTAMPTZ,
  UNIQUE (call_id, stage)
);
```

Workers claim jobs with `SELECT ... FOR UPDATE SKIP LOCKED`, which gives safe
concurrent consumption without a broker. Retries use exponential backoff;
after three attempts the job goes to dead-letter with its error preserved.

Use the provider's **batch API** for stages with no latency requirement, which
is all of them. It is typically half price.

Move to Redis and a proper worker framework only when the job table becomes a
bottleneck — which at 5,000 calls a day it will not.

### 4.12 Monitoring

Metrics to emit per stage per day:

| Metric | Alert condition |
|---|---|
| Stage failure rate | > 2% |
| Schema validation failure rate | > 1% |
| Verifier rejection rate | Deviation > 10pp from baseline |
| Contradiction rate | Deviation > 5pp from baseline |
| Cost per call | > budget threshold |
| Label distribution | Any class shifts > 10pp week over week |
| Tier 0 proportion | Sudden change — indicates upstream break |

**Label distribution drift is the most important of these.** If "not interested"
moves from 40% to 60% overnight, either the business changed or the pipeline
broke. Either way it needs to surface that day, not at month end.

---

## 5. Knowledge layer

Markdown files in git, one concept per file, following the Open Knowledge Format
convention — YAML frontmatter with a `type` field, body content, links between
concepts.

```
knowledge/
  objections/
    rate-too-high.md
    emi-burden.md
    cibil-concern.md
  compliance/
    must-disclose-apr.md
    prohibited-guarantee-claims.md
    lender-identity-disclosure.md
  products/
    pl-salaried.md
    pl-self-employed.md
  taxonomy/
    dispositions.md
  metrics/
    qualified-lead.md
  CHANGELOG.md
```

Each file serves two readers: **annotators**, who apply the rules by hand, and
**prompts**, which load the same text at runtime. One source, so definitions
cannot drift between human and model.

```python
def load_knowledge(paths: list[str]) -> str:
    parts = []
    for p in paths:
        fm, body = parse_frontmatter(KNOWLEDGE_DIR / p)
        parts.append(f"## {fm['title']}\n{body}")
    return "\n\n".join(parts)
```

Version the directory. Stamp every insight with `knowledge_version`. A rule
change that shifts flag rates must be attributable.

**Caveat:** OKF is a young specification. Adopt the file and folder convention —
it costs nothing, being markdown in git — but do not build dependencies on
OKF-specific tooling.

---

## 6. Cost model

Rough arithmetic, to be validated against your own data.

Inputs: 5,000 calls/day, ~60% triaged to tier 0, leaving ~2,000 substantive
calls. A 10-minute call is roughly 1,500–2,000 tokens of transcript.

Without segmentation, sending the full transcript to every stage costs roughly
20k input tokens per call. With phase scoping, that falls to around 6–8k.

The dominant levers, in order:

1. **Triage** — removes ~60% of calls entirely.
2. **Model routing** — small versus large models differ by roughly an order of
   magnitude in price. Most stages do not need the large model.
3. **Segmentation** — halves input tokens on the stages that follow it.
4. **Tiered verification** — verify compliance flags and a sample, not
   everything.
5. **Batch API** — typically 50% discount.

Track `input_tokens` and `output_tokens` per insight row so cost attribution is
per stage, not just a monthly bill.

---

## 7. Capabilities needed

| Capability | Where it is used | Notes |
|---|---|---|
| Python, async, Pydantic | Everything | Core requirement |
| SQL and schema design | Storage, harness | Postgres proficiency, JSONB |
| Prompt engineering | Extraction stages | The largest single time sink |
| Evaluation methodology | Harness, gold set | Precision/recall, kappa, IoU, sampling |
| NLP fundamentals | Redaction, metrics | NER, tokenisation, text normalisation |
| Data annotation management | Gold set | Codebook design, agreement measurement |
| Basic frontend | Review UI | Internal tool; simplicity over polish |
| Domain knowledge — PL telecalling | Schema, codebook | Cannot be outsourced |
| Compliance knowledge | Rules, redaction | Needs a named person in the loop |

The two most commonly underestimated: **evaluation methodology** and
**annotation management**. Both are treated as administrative and both determine
whether the project produces something trustworthy.

---

## 8. Upstream contract — ASR requirements

The transcription layer must deliver:

**Required:**

- Dual-channel recording where available — removes diarisation error entirely
- Word-level timestamps
- Word-level confidence scores
- Verbatim output, with vendor "smart formatting" disabled
- Both spoken and normalised number forms
- Non-speech segments labelled and timed (silence, hold, DTMF)
- Turn structure with speaker labels

**Valuable:**

- Overlap markers for interruption detection
- Per-segment language identification
- N-best alternatives on low-confidence spans
- Audio quality metrics (SNR, clipping)

**Provenance, for audit:**

- `audio_file_hash`, `asr_model`, `asr_version`, `transcription_timestamp`,
  `sample_rate`, `channels`, `codec`

**The single most dangerous default** in commercial ASR products for this use
case is output cleanup. If an agent says "sirf one point five percent" and the
transcript renders "1.5% per annum", the system has invented the disclosure it
was checking for. Disable it.

**From the dialer, separately:** `hold_events`, `transfer_events`,
`recording_start_offset`, `dialer_disposition`, `agent_id`, `campaign`,
`lead_source`.

---

## 9. Decisions to settle before building

| Decision | Consequence if deferred |
|---|---|
| Where inference runs | Determines what redaction must achieve; involves compliance sign-off, so it is the long-pole item |
| Dual-channel recording available? | Determines whether diarisation is a component at all |
| Sample rate stored (8 kHz vs 16 kHz) | Caps achievable accuracy on digits, which are the values that matter most |
| Schema owner | Without one named person, schema drifts by committee |
| Human review capacity | Compliance detection without review capacity produces a dashboard nobody acts on |

---

## 10. Build order

1. Skeleton with stub stages, end to end
2. Evaluation harness, before any prompt exists
3. Ingest, redaction, triage, deterministic metrics — ship this
4. Outcome stage, using the full stage pattern
5. Segmentation
6. Compliance stage plus verifier
7. Remaining classification stages, one at a time
8. Fact extraction
9. Merge and contradiction rules
10. Review interface and feedback loop
11. Production hardening and monitoring
12. Archive backfill

**Governing rule:** never change two things at once. One stage, one prompt
version, one measurement. Attribution is impossible otherwise, and without
attribution the harness is decoration.
