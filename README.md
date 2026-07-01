# Provenance Guard

A backend attribution system for creative-writing platforms. It classifies submitted text as likely human-written, likely AI-generated, or uncertain; returns a plain-language transparency label with every decision; logs every event in a structured audit trail; and gives creators a path to contest a classification through an appeals workflow.

---

## Architecture Overview

### Submission flow

A creator submits a `POST /submit` request with their `text` and a `creator_id`. The text is passed simultaneously through two independent detection signals:

1. **LLM signal** — the text is sent to Groq (`llama-3.3-70b-versatile`) with a structured prompt that asks the model to estimate the probability the text was AI-generated. The model returns a float between 0 and 1.
2. **Stylometric signal** — three statistical properties of the text are computed in pure Python (sentence length variance, type-token ratio, punctuation diversity) and combined into a single score.

The two scores are combined into a single `confidence` value using a weighted average (60% LLM, 40% stylometric). The confidence maps to one of three attribution categories (`likely_human`, `uncertain`, `likely_ai`) and selects the corresponding plain-language transparency label. Every decision — both individual signal scores, the combined confidence, the attribution, and the label — is written to a structured JSON Lines audit log. The full result is returned as JSON to the caller.

### Appeal flow

A creator who disputes a classification submits a `POST /appeal` request with the `content_id` from their original submission and a `creator_reasoning` string explaining why they believe the classification is wrong. The system looks up the original log entry, updates its status from `"classified"` to `"under_review"`, and appends a separate appeal entry to the audit log containing the creator's reasoning alongside the original attribution and confidence score. The appeal is flagged for human review; the system does not automatically re-classify.

---

## Detection Signals

### Signal 1 — LLM Classification (Groq)

**What it measures:** Semantic and stylistic coherence as judged by `llama-3.3-70b-versatile`. The model evaluates holistic patterns associated with AI authorship: uniform hedging language ("it is important to note", "furthermore"), overly structured paragraph flow, absence of personal voice, and consistent formal register throughout.

**Why this signal:** An LLM is the only practical tool for capturing semantic meaning. Stylometric heuristics can measure structure but cannot tell whether a text sounds like a person or a chatbot. The Groq free tier makes this feasible at no cost.

**What it misses:** Text that has been heavily edited by a human after AI generation. Formal human writing (legal briefs, academic abstracts) that mimics the register the model associates with AI. The model's output reflects its own training distribution, not ground truth about how any specific piece of text was produced.

**Output format:** A float `0.0–1.0`. `0.0` = the model is confident the text is human-written. `1.0` = the model is confident it is AI-generated.

---

### Signal 2 — Stylometric Heuristics (Pure Python)

**What it measures:** Three statistical properties that differ systematically between human and AI writing:

- **Sentence length variance** — AI text uses more uniform sentence lengths; human writing mixes short punchy sentences with long winding ones. Computed as the standard deviation of word counts per sentence. Low variance → higher AI score.
- **Type-token ratio (TTR)** — unique words divided by total words. AI text reuses a narrower vocabulary; human writing is more lexically diverse. Low TTR → higher AI score.
- **Punctuation diversity** — em dashes, semicolons, ellipses, parentheses, and exclamation marks appear more often in human writing. AI text defaults to periods and commas. Low proportion of non-standard punctuation → higher AI score.

**Why this signal:** It is structurally independent of the LLM signal. The LLM judges meaning; stylometrics judge measurable surface properties. When the two signals disagree, that disagreement is informative — it pushes the combined confidence into the uncertain range, which is the correct behavior when evidence is mixed.

**What it misses:** Non-native English speakers who write carefully and formally often produce low sentence length variance and reduced TTR — not because they used AI, but because they are deliberate with a second language. Short texts under ~60 words produce unreliable variance and TTR estimates due to small sample sizes. Poets who favor short, uniform lines will score high on AI probability from this signal alone regardless of authorship.

**Output format:** A float `0.0–1.0`. Sub-metrics are each normalized to `0–1` and averaged. `0.0` = strongly human-like structure. `1.0` = strongly AI-like structure.

---

## Confidence Scoring

### Combination formula

```
confidence = (0.60 × llm_score) + (0.40 × stylo_score)
```

The LLM signal is weighted higher (60%) because it captures semantic and contextual properties that heuristics cannot. The stylometric signal (40%) acts as a structural check that is independent of the LLM's biases. When the two signals strongly disagree, the combined score lands in the uncertain range — which correctly communicates that the system does not know.

### Attribution thresholds

| Confidence | Attribution | Meaning |
|---|---|---|
| `< 0.35` | `likely_human` | Both signals lean toward human authorship |
| `0.35 – 0.65` | `uncertain` | Signals disagree or both are near the midpoint |
| `> 0.65` | `likely_ai` | Both signals lean toward AI generation |

The threshold is asymmetric by design: it takes more combined signal to assert AI generation than to assert human authorship. A false positive — labeling a human's work as AI-generated — is a worse outcome on a creative platform than a false negative.

### Validation: two example submissions with different scores

**High-confidence human — casual social media writing**

> *"ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it and i was thirsty for like three hours after"*

```json
{
  "llm_score": 0.2,
  "stylo_score": 0.1068,
  "confidence": 0.1627,
  "attribution": "likely_human"
}
```

Both signals agree strongly. The LLM reads personal voice and irregular register; the stylometric signal picks up high punctuation diversity (`?`, `WAY`) and varied sentence lengths.

---

**High-confidence AI — structured corporate paragraph**

> *"Artificial intelligence represents a transformative paradigm shift in modern society. It is important to note that while the benefits of AI are numerous, it is equally essential to consider the ethical implications. Furthermore, stakeholders across various sectors must collaborate to ensure responsible deployment."*

```json
{
  "llm_score": 0.8,
  "stylo_score": 0.4401,
  "confidence": 0.656,
  "attribution": "likely_ai"
}
```

The LLM strongly flags the hedging language and parallel structure. The stylometric signal also flags low punctuation diversity and moderate sentence uniformity. The combined confidence of `0.656` sits right at the threshold — the label returned is `likely_ai` but the score honestly communicates lower certainty than `0.8` alone would suggest, because the stylometric signal is not as strong as the LLM signal here.

---

## Transparency Label Variants

These are the exact strings returned in the `label_text` field of every `/submit` response.

### High-confidence human (`confidence < 0.35`)

> "This content shows strong indicators of human authorship. Our analysis found writing patterns consistent with a person rather than an AI writing tool."

### Uncertain (`confidence 0.35 – 0.65`)

> "This content may have been written by a person, an AI tool, or a combination of both. Our system was not able to make a confident determination. The creator can submit an appeal to provide additional context."

### High-confidence AI (`confidence > 0.65`)

> "This content was likely generated by an AI writing tool. Our analysis found strong indicators of AI authorship based on writing style and structure. If you are the creator and believe this is incorrect, you can submit an appeal."

**Design notes:** Labels use "indicators" and "patterns" rather than "detected" or "confirmed" to communicate that results are probabilistic. The AI label always includes the appeal path. The human label does not — there is no reason to invite creators to contest a favorable result. No label uses technical terms (score, classifier, confidence, logit).

---

## Rate Limiting

**Limits applied to `POST /submit`:**

```
10 requests per minute
100 requests per day
```

**Reasoning:** A legitimate creator submitting their own work might post a few pieces in a session, but would rarely exceed 10 in a single minute. The per-minute limit stops burst abuse — scripts that flood the endpoint to probe the detection logic or exhaust API quota — while giving real users generous headroom. The per-day limit (100) accommodates a power user who submits frequently throughout the day while capping automated mass-submission.

The limits are enforced per IP address using Flask-Limiter with in-memory storage. Exceeding the limit returns `HTTP 429 Too Many Requests`.

**Rate limit test results** (12 rapid requests; limit = 10/minute):

```
Request 1:  HTTP 200
Request 2:  HTTP 200
Request 3:  HTTP 200
Request 4:  HTTP 200
Request 5:  HTTP 200
Request 6:  HTTP 200
Request 7:  HTTP 429   ← limit reached (earlier test requests counted in window)
Request 8:  HTTP 429
Request 9:  HTTP 429
Request 10: HTTP 429
Request 11: HTTP 429
Request 12: HTTP 429
```

The 429s fired at request 7 rather than 11 because earlier submissions made during the same 60-second testing window had already consumed quota. This is correct rolling-window behavior.

---

## Audit Log Sample

The audit log is stored as JSON Lines at `audit_log.jsonl`. Every `POST /submit` appends one entry; every `POST /appeal` appends a second entry and updates the status on the original. The `GET /log` endpoint returns the last N entries as JSON.

Three representative entries:

**Classified as `likely_human`:**
```json
{
  "attribution": "likely_human",
  "confidence": 0.1635,
  "content_id": "6aeb2044-71e9-4409-be8a-0971286d4b9c",
  "creator_id": "validation-2",
  "llm_score": 0.2,
  "stylo_score": 0.1088,
  "status": "classified",
  "timestamp": "2026-07-01T03:03:51.438125+00:00"
}
```

**Classified as `likely_ai`, later appealed (status updated to `under_review`):**
```json
{
  "attribution": "likely_ai",
  "confidence": 0.656,
  "content_id": "61491939-a954-4e3f-85e5-f4e6bb0a02b2",
  "creator_id": "creator-appeal-test",
  "llm_score": 0.8,
  "stylo_score": 0.4401,
  "status": "under_review",
  "timestamp": "2026-07-01T03:10:21.254549+00:00"
}
```

**Appeal entry for the same `content_id`:**
```json
{
  "content_id": "61491939-a954-4e3f-85e5-f4e6bb0a02b2",
  "creator_reasoning": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical.",
  "event": "appeal",
  "original_attribution": "likely_ai",
  "original_confidence": 0.656,
  "status": "under_review",
  "timestamp": "2026-07-01T03:11:54.137454+00:00"
}
```

Full log is viewable live at `GET /log` (returns last 20 entries by default; use `?limit=N` for more).

---

## Known Limitations

### Non-native English speakers writing formally

A non-native English speaker who writes with careful grammar and consistent sentence structure will often produce text with low sentence length variance and a reduced type-token ratio — the same statistical profile the stylometric signal uses to flag AI-generated text. This is not because their writing is AI-generated, but because deliberate second-language writing and AI writing share surface-level uniformity.

The LLM signal may partially compensate if it picks up on idiomatic word choices or personal voice, but in practice it also tends to associate formal, grammatically correct prose with AI generation. A non-native English speaker writing a formal piece could consistently score in the `uncertain` or `likely_ai` range despite writing entirely without AI assistance. This is a structural false-positive risk tied directly to the properties both signals measure, and it would require signal recalibration or an additional signal (e.g., author history or metadata) to address.

### Short texts under ~60 words

Sentence length variance and TTR are statistically unreliable at short text lengths. A haiku has fixed syllable counts and near-zero sentence length variance, which signals AI uniformity regardless of authorship. A 40-word caption with 35 unique words produces an extremely high TTR, which signals human diversity regardless of authorship. The system does not currently detect or flag texts below a minimum length threshold, meaning short creative work (poems, captions, one-paragraph entries) receives confidence scores that are less trustworthy than the number suggests.

---

## Spec Reflection

### Where the spec helped

Writing out the exact transparency label text in `planning.md` before touching any code was the most useful pre-work in the project. When it came time to implement `generate_label()`, there was no design decision to make — the function was a direct transcription of three strings already written and reviewed. It also forced a decision about label tone (probabilistic language, no jargon, appeal path only on the AI label) before the pressure of an open editor made it tempting to just return `"AI detected"` and move on.

### Where implementation diverged from the spec

The spec defined the confidence threshold for `likely_ai` as `> 0.65`. In practice, the AI detection test case ("Artificial intelligence represents a transformative paradigm shift...") consistently produces a combined confidence of exactly `0.656` — close to but not over `0.65` with the stylometric signal pulling the LLM's `0.8` downward. The spec called this `likely_ai` but the score sits in the borderline zone.

Rather than adjusting the threshold to guarantee the test case lands on the right side, the system was left as-is: `0.656` is correctly reported as `likely_ai` (it's above `0.65`) and the score honestly communicates moderate rather than high confidence. This is better engineering than tuning thresholds to hit expected outputs — the confidence score exists precisely to communicate cases like this one.

---

## API Reference

| Endpoint | Method | Description |
|---|---|---|
| `/submit` | `POST` | Submit text for attribution analysis |
| `/appeal` | `POST` | Contest a classification by content_id |
| `/log` | `GET` | Retrieve recent audit log entries |

**`POST /submit` — request body:**
```json
{
  "text": "The content to analyze",
  "creator_id": "unique-creator-identifier"
}
```

**`POST /submit` — response:**
```json
{
  "content_id": "uuid",
  "attribution": "likely_human | uncertain | likely_ai",
  "confidence": 0.0,
  "llm_score": 0.0,
  "stylo_score": 0.0,
  "label_text": "Plain-language label shown to readers",
  "status": "classified"
}
```

**`POST /appeal` — request body:**
```json
{
  "content_id": "uuid-from-submit-response",
  "creator_reasoning": "Why the creator believes the classification is wrong"
}
```

**`GET /log` — query params:** `?limit=N` (default 20)

**Note:** Server runs on port `5001`. macOS AirPlay Receiver occupies port 5000 by default.

---

## AI Usage

### Instance 1 — Flask skeleton and `signal_llm()`

**What I directed:** I provided the detection signals section and architecture diagram from `planning.md` and asked Claude to generate (1) a Flask app skeleton with a `POST /submit` stub returning hardcoded JSON, (2) a `signal_llm()` function calling Groq with `llama-3.3-70b-versatile` that returns an `ai_probability` float, and (3) JSON Lines audit log helpers.

**What it produced:** A working Flask skeleton and a `signal_llm()` function with a clear prompt structure. The log helpers matched the spec's required fields.

**What I revised:** The generated prompt sent to Groq returned prose text rather than structured JSON, causing the parser to fail on the first test. I rewrote the prompt to explicitly instruct the model to respond with only a JSON object (`{"ai_probability": <float>}`) and added a code-fence stripping step to handle cases where the model wraps its response in markdown. I also verified the output float range was being clamped to `[0, 1]`, which the original did not do.

---

### Instance 2 — `signal_stylometric()` and `compute_confidence()`

**What I directed:** I provided the stylometric heuristics section of `planning.md` (listing sentence length variance, TTR, and punctuation diversity as the three sub-metrics) and the combination formula (`0.60 × llm_score + 0.40 × stylo_score`) and asked Claude to generate a standalone `signal_stylometric()` function and a `compute_confidence()` function.

**What it produced:** A working stylometric function. The sentence splitting used `re.split(r'[.!?]+', text)` which correctly handles consecutive punctuation. TTR and punctuation density were computed as specified.

**What I revised:** The normalization bounds for each sub-metric were the main thing I adjusted. The generated code used a TTR threshold of `0.6` as the "fully AI" floor, but testing on real inputs showed that even clearly human casual writing has TTR around `0.55–0.65` for short texts, which would have mis-scored it. I recalibrated the TTR normalization floor to `0.5` and the human ceiling to `0.85` based on testing across the four validation inputs. Similarly, the sentence variance clamp was originally set at `std_dev >= 5` words; I raised it to `8` after observing that formal human writing with longer sentences was not getting sufficient credit for variance.

---

## Running Locally

```bash
# Clone and set up
git clone https://github.com/BarkotBeyene/ai201-project4-provenance-guard.git
cd ai201-project4-provenance-guard
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Add your Groq API key
echo "GROQ_API_KEY=your_key_here" > .env

# Start the server
python app.py
# Server runs at http://localhost:5001

# Submit content
curl -X POST http://localhost:5001/submit \
  -H "Content-Type: application/json" \
  -d '{"text": "Your text here", "creator_id": "your-id"}'

# Submit an appeal
curl -X POST http://localhost:5001/appeal \
  -H "Content-Type: application/json" \
  -d '{"content_id": "uuid-from-submit", "creator_reasoning": "Your reasoning here"}'

# View the audit log
curl http://localhost:5001/log
```
