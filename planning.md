# Provenance Guard — Planning Document

## Architecture

### Submission Flow

```
POST /submit
  │
  ├─ Extract: text, creator_id
  │
  ├─► [Signal 1: LLM Classification]
  │     Groq (llama-3.3-70b-versatile)
  │     Prompt asks: "Rate the probability this text was AI-generated (0.0–1.0)"
  │     Output: llm_score (float 0–1)
  │
  ├─► [Signal 2: Stylometric Heuristics]
  │     Pure Python — no external libraries
  │     Measures: sentence length variance, type-token ratio, punctuation density
  │     Output: stylo_score (float 0–1, higher = more AI-like)
  │
  ├─► [Signal 3: N-gram Repetition (STRETCH — Ensemble)]
  │     Pure Python — no external libraries
  │     Measures: ratio of repeated bigrams + trigrams to total n-grams
  │     AI text reuses phrase patterns more than human text
  │     Output: ngram_score (float 0–1, higher = more AI-like)
  │
  ├─► [Confidence Scoring — Weighted Ensemble]
  │     combined = (0.50 × llm_score) + (0.30 × stylo_score) + (0.20 × ngram_score)
  │     Conflict resolution: if llm and stylo disagree by > 0.4, weight shifts to llm
  │     Output: confidence (float 0–1)
  │
  ├─► [Transparency Label]
  │     confidence < 0.35  → "Likely Human-Written" label
  │     confidence 0.35–0.65 → "Uncertain" label
  │     confidence > 0.65  → "Likely AI-Generated" label
  │
  ├─► [Provenance Certificate Check (STRETCH)]
  │     If creator has verified status → append certificate badge to response
  │
  ├─► [Audit Log]
  │     Append structured JSON entry to audit_log.jsonl
  │     Fields: content_id, creator_id, timestamp, attribution,
  │             confidence, llm_score, stylo_score, ngram_score, status
  │
  └─► JSON Response
        content_id, attribution, confidence, llm_score,
        stylo_score, ngram_score, label_text, status, certificate (if applicable)
```

### Appeal Flow

```
POST /appeal
  │
  ├─ Extract: content_id, creator_reasoning
  │
  ├─► [Status Update]
  │     Look up content_id in audit log
  │     Update status: "classified" → "under_review"
  │
  ├─► [Audit Log]
  │     Append appeal entry to audit_log.jsonl
  │     Fields: content_id, event: "appeal", creator_reasoning,
  │             timestamp, status: "under_review"
  │
  └─► JSON Response
        content_id, status: "under_review", message
```

### Narrative

A submitted piece of text passes through two independent detection signals — one semantic (LLM judgment) and one structural (stylometric heuristics) — whose outputs are combined into a single confidence score. That score determines which transparency label is returned to the caller alongside the raw scores. Every decision is written to a structured append-only audit log. If a creator disputes the result, a `POST /appeal` call updates the content's status to "under review" and appends the appeal and its reasoning to the same log, creating a complete audit trail.

---

## Detection Signals

### Signal 1 — LLM Classification (Groq)

**What it measures:** Semantic and stylistic coherence as judged by a large language model. The LLM evaluates whether the text exhibits holistic patterns associated with AI authorship — uniform hedging language, overly structured paragraph flow, absence of personality or voice, consistent formality throughout.

**Output format:** A float between 0.0 and 1.0. `0.0` = the model is confident the text is human-written; `1.0` = the model is confident the text is AI-generated. Extracted from a structured JSON response enforced by the prompt.

**What it misses:** The LLM cannot detect AI text that has been heavily edited by a human, nor human text written in a deliberately formal or structured style (e.g., legal writing, academic abstracts). It also has no access to metadata (authorship history, platform behavior) that a real attribution system might use. Its output reflects the training distribution of the LLM itself, not ground truth.

---

### Signal 2 — Stylometric Heuristics (Pure Python)

**What it measures:** Three statistical properties of the text that differ systematically between human and AI writing:

1. **Sentence length variance** — AI-generated text tends to use more uniform sentence lengths. Human writing is more erratic: short punchy sentences mix with long winding ones. Computed as the variance of word counts per sentence. Low variance → higher AI probability.

2. **Type-token ratio (TTR)** — Unique words divided by total words. AI text reuses a narrower vocabulary relative to its length; human writing is more lexically diverse. Low TTR → higher AI probability.

3. **Punctuation density** — Em dashes, semicolons, ellipses, and unconventional punctuation appear more often in human writing. AI text favors periods and commas almost exclusively. Low density of non-standard punctuation → higher AI probability.

**Output format:** Each sub-metric is normalized to a 0–1 range and averaged into a single `stylo_score` float. `0.0` = strongly human-like structure; `1.0` = strongly AI-like structure.

**What it misses:** Non-native English speakers often write with lower TTR and more uniform sentence lengths than native speakers — not because their writing is AI-generated, but because they are deliberately careful. A poet who favors short, uniform lines will score high on AI probability from this signal alone. Very short texts (under ~50 words) produce unreliable variance and TTR estimates.

---

### Combining Signals

```
confidence = (0.60 × llm_score) + (0.40 × stylo_score)
```

The LLM signal is weighted higher (60%) because it captures semantic and contextual properties that the stylometric signal cannot. The stylometric signal (40%) acts as a structural check that is independent of the LLM's own biases. When the two signals strongly disagree (e.g., `llm_score = 0.8`, `stylo_score = 0.2`), the combined score lands in the uncertain range — which is the correct behavior, since disagreement between signals means the system genuinely doesn't know.

---

## Uncertainty Representation

| Confidence Range | Attribution | Meaning |
|---|---|---|
| 0.00 – 0.34 | `likely_human` | Both signals lean toward human authorship |
| 0.35 – 0.65 | `uncertain` | Signals disagree or both are near the midpoint |
| 0.66 – 1.00 | `likely_ai` | Both signals lean toward AI generation |

**What a score of 0.60 means:** The system has a moderate lean toward AI but is not confident. The confidence is high enough to flag the content for transparency but not high enough to assert AI generation. The label for this score uses hedged language ("may have been") rather than assertive language ("was generated by").

**What a score of 0.35 means:** This sits exactly at the boundary between uncertain and likely-human. The label will be the uncertain variant. A score of 0.34 would produce the likely-human label. This boundary was chosen deliberately asymmetrically: it takes more signal to call something AI-generated than to call it human, because a false positive (labeling a human's work as AI) is worse than a false negative on a creative platform.

**Calibration approach:** Scores are tested against a set of known inputs before deployment — clearly AI-generated paragraphs should consistently score above 0.70, clearly human casual writing should score below 0.30, and formal human writing (legal prose, academic text) should score in the 0.35–0.55 range rather than triggering a high-confidence AI label.

---

## Transparency Label Variants

These are the exact strings returned in the `label_text` field of the API response and displayed to readers.

### High-Confidence AI (`confidence > 0.65`)

```
"This content was likely generated by an AI writing tool.
Our analysis found strong indicators of AI authorship based on
writing style and structure. If you are the creator and believe
this is incorrect, you can submit an appeal."
```

### Uncertain (`confidence 0.35–0.65`)

```
"This content may have been written by a person, an AI tool,
or a combination of both. Our system was not able to make a
confident determination. The creator can submit an appeal to
provide additional context."
```

### High-Confidence Human (`confidence < 0.35`)

```
"This content shows strong indicators of human authorship.
Our analysis found writing patterns consistent with a person
rather than an AI writing tool."
```

**Design rationale:** Labels use "indicators" and "patterns" rather than "detected" or "confirmed" to convey that this is probabilistic, not certain. The AI label always mentions the appeal path. The human label does not prompt an appeal — there is no reason to invite creators to contest a favorable result. None of the labels use technical terms (score, classifier, confidence, logit).

---

## Appeals Workflow

**Who can submit:** Any creator who has a `content_id` from a prior submission. No authentication is implemented in this prototype; in production, the appeal endpoint would verify the `creator_id` matches the original submission.

**Required fields:**
- `content_id` (string) — the ID returned by the original `/submit` call
- `creator_reasoning` (string) — the creator's explanation of why they believe the classification is wrong

**What the system does when an appeal is received:**
1. Looks up the original audit log entry by `content_id`
2. Updates the entry's `status` field from `"classified"` to `"under_review"`
3. Appends a new audit log entry of type `"appeal"` containing the `content_id`, `creator_reasoning`, timestamp, and updated status
4. Returns a JSON confirmation to the caller

**What a human reviewer would see when opening the appeal queue:** A `GET /log` call filtered by `status: "under_review"` would show all entries pending review. Each entry includes the original attribution and confidence score alongside the creator's reasoning, giving the reviewer everything needed to make a judgment without re-running the pipeline.

**Automated re-classification:** Not implemented. The appeal is a signal for human review, not a trigger for automatic re-scoring. This is intentional — automated re-classification based on creator reasoning could be gamed by anyone who learns what language convinces the system.

---

## Anticipated Edge Cases

### Edge Case 1: Non-native English speakers writing carefully and formally

A non-native English speaker who writes in a deliberate, grammatically careful style will often produce text with low sentence length variance and reduced TTR relative to native speakers — not because they used an AI tool, but because they are being careful with a second language. The stylometric signal will incorrectly score this text as AI-like. The LLM signal may partially compensate if it picks up on idiomatic word choices or personal voice, but combined confidence may still land in the uncertain range or above.

**Why the signal fails here:** The stylometric heuristics were calibrated against native English writing. The properties they measure (variance, TTR, punctuation diversity) correlate with naturalness in native writing but do not account for the different naturalness profile of second-language writing.

### Edge Case 2: Short texts under ~60 words (poems, captions, one-paragraph entries)

Sentence length variance and TTR are statistically unreliable at very short text lengths. A haiku has three sentences with fixed syllable counts — its variance will be near zero, falsely signaling AI uniformity. A 40-word caption may use 35 unique words (very high TTR), falsely signaling human diversity. The LLM signal may also struggle because short texts give the model little evidence to work with.

**Why the signal fails here:** Both heuristics assume a text long enough to produce a stable distribution. Below ~60 words, the sample is too small, and the signal should be treated as unreliable. The system does not currently detect or flag this condition.

---

## AI Tool Plan

### M3 — Submission Endpoint + First Signal

**Spec sections to provide:** Detection Signals (Signal 1 — LLM Classification), Architecture diagram (Submit Flow), Uncertainty Representation (output format only).

**What to ask the AI to generate:**
1. Flask app skeleton with `POST /submit` stub that reads `text` and `creator_id` from the JSON body and returns a hardcoded placeholder response including `content_id`, `attribution`, `confidence`, and `label_text`.
2. A standalone `signal_llm(text)` function that calls the Groq API with `llama-3.3-70b-versatile`, asks the model to return a JSON object with an `ai_probability` field (float 0–1), and returns that float.
3. JSON Lines audit log helpers: `log_event(entry)` and `read_log(limit)`.
4. `GET /log` endpoint returning the last 20 log entries.

**How to verify before wiring into the route:** Call `signal_llm()` directly on three test strings — one clearly AI (structured corporate paragraph), one clearly human (casual social media post), one borderline (formal academic writing). Print the raw Groq response and the parsed float. Confirm the float is in 0–1 range and that scores roughly rank in the expected order before integrating into the endpoint.

---

### M4 — Second Signal + Confidence Scoring

**Spec sections to provide:** Detection Signals (Signal 2 — Stylometric Heuristics), Uncertainty Representation (full — thresholds and combination formula), Architecture diagram (Submit Flow).

**What to ask the AI to generate:**
1. A standalone `signal_stylometric(text)` function computing sentence length variance, TTR, and punctuation density, normalizing each to 0–1, and returning their average as `stylo_score`.
2. A `compute_confidence(llm_score, stylo_score)` function applying the 60/40 weighted formula.
3. An `attribution_from_confidence(confidence)` function returning the attribution string (`"likely_ai"`, `"uncertain"`, `"likely_human"`) based on the defined thresholds.
4. Updated `POST /submit` wiring both signals, computing confidence, and returning all individual scores alongside the combined result.

**How to verify:** Run all four Milestone 4 test inputs (clear AI, clear human, two borderline) through the full pipeline and print `llm_score`, `stylo_score`, and `confidence` side by side. Confirm that clear AI > 0.65 and clear human < 0.35. If a borderline input produces a surprising score, check which signal is the outlier and investigate that function independently.

---

### M5 — Production Layer

**Spec sections to provide:** Transparency Label Variants (exact text), Appeals Workflow (fields, status changes, log behavior), Architecture diagram (both flows), Rate Limiting reasoning.

**What to ask the AI to generate:**
1. A `generate_label(confidence)` function returning the exact label text strings defined in this spec for each of the three confidence ranges.
2. `POST /appeal` endpoint that reads `content_id` and `creator_reasoning`, appends an appeal entry to the audit log with `status: "under_review"`, and returns a confirmation JSON.
3. Flask-Limiter setup with `storage_uri="memory://"` and `@limiter.limit("10 per minute;100 per day")` applied to `POST /submit`.

**How to verify:**
- Label variants: submit one input from each confidence range and confirm the returned `label_text` matches the exact strings in this spec.
- Appeal: submit a content ID from a prior `/submit` call, then `GET /log` and confirm the appeal entry is present with `status: "under_review"` and `creator_reasoning` populated.
- Rate limiting: run the 12-request loop and confirm the first 10 return `200` and the remaining return `429`.

---

## Stretch Features

### S1 — Ensemble Detection (3 signals)

**Signal 3: N-gram Repetition Rate**

What it measures: the ratio of repeated 2-gram and 3-gram phrases to total n-grams. AI text reuses phrase patterns at higher rates than human text because it generates tokens based on learned distributions that favor common co-occurrences. Human writing is more inventive at the phrase level.

Output: `ngram_score` float 0–1. Higher = more AI-like (more repetitive phrase patterns).

Weighting strategy (3-signal ensemble):
```
confidence = (0.50 × llm_score) + (0.30 × stylo_score) + (0.20 × ngram_score)
```

Conflict resolution: if `llm_score` and `stylo_score` differ by more than 0.4, the ensemble shifts weight further toward the LLM (semantic judgment is more reliable than structural heuristics when signals strongly conflict). The n-gram signal serves as a tiebreaker in moderate-disagreement cases.

Why this weighting: LLM (50%) carries the most weight because it understands meaning. Stylometric (30%) provides structural independence. N-gram (20%) adds phrase-level evidence without over-indexing on a signal that fails on short texts.

---

### S2 — Provenance Certificate

**Design:** A creator can earn a "Verified Human Author" certificate by completing a verification challenge on their content. The challenge: the creator must answer a knowledge question about specific details in their submitted text that only the actual author would know (e.g., "What specific emotion did you intend in the third sentence?" or "Describe something about the context in which you wrote this that isn't in the text"). The system stores the certificate against the `creator_id`.

**Endpoints:**
- `POST /verify` — accepts `content_id` and `verification_answer`; stores verified status; returns certificate
- `GET /certificate/<content_id>` — returns certificate status and badge text

**What "verified" means on subsequent submits:** if a creator holds a certificate, future `/submit` responses include a `certificate` field with the badge text, distinguishable from the standard transparency label.

**Storage:** certificates stored in `certificates.json` keyed by `creator_id`.

---

### S3 — Analytics Dashboard

**Endpoint:** `GET /analytics`

**Metrics:**
1. **Detection pattern** — count and percentage breakdown of `likely_human`, `uncertain`, `likely_ai` across all classified entries
2. **Appeal rate** — number of appeals / number of classified submissions
3. **Average confidence by attribution** — mean confidence score within each of the three categories (shows whether the system is calibrated: `likely_ai` entries should average higher than `uncertain`, which should average higher than `likely_human`)

---

### S4 — Multi-Modal Support

**Second content type:** image descriptions (alt-text or caption strings submitted alongside or instead of prose text). The pipeline differences:

- Signal 1 (LLM): prompt is reworded to assess whether an image description reads as AI-generated (AI descriptions tend to be formulaic: "A [adjective] [noun] [verb phrase] against a [adjective] background")
- Signal 2 (Stylometric): TTR and punctuation diversity still apply; sentence length variance is less meaningful for single-sentence captions, so it is down-weighted
- Signal 3 (N-gram): applies as-is

**Endpoint extension:** `POST /submit` accepts an optional `content_type` field (`"text"` default, `"image_description"` for captions/alt-text). Routes to the appropriate signal configuration.
