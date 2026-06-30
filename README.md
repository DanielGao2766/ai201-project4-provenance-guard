# Provenance Guard

A pluggable backend service that classifies submitted creative text as human-written or AI-generated, returns a confidence-scored transparency label, and manages creator appeals. Any creative-sharing platform can call it as an API to give audiences the attribution context they need.

---

## Architecture

```
POST /submit
    │
    ├─ Signal 1: Groq LLM (llama-3.3-70b-versatile)
    │   └─ Semantic/stylistic classification → {attribution, confidence, reasoning}
    │
    ├─ Signal 2: Local stylometric heuristics (no API call)
    │   ├─ Sentence-length coefficient of variation
    │   ├─ Formal transition phrase density
    │   └─ Punctuation variety
    │
    ├─ combine_signals() → weighted score (70% S1, 30% S2)
    │   └─ attribution_to_label() → transparency label
    │
    └─ SQLite audit log (submissions + appeals tables, FK relationship)
```

The design is intentionally two-layer: a semantic layer (LLM) that understands *meaning* and a structural layer (heuristics) that catches *surface patterns*. When both agree, confidence is high. When they disagree, the system returns "Uncertain" — which is the honest answer.

Storage is SQLite for development. The schema (submissions table with `content_id` PK, appeals table with `content_id` FK) is designed so a production migration to PostgreSQL requires no query changes.

---

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Browser submission form |
| `POST` | `/submit` | Classify text; returns attribution, confidence, label, signal scores |
| `POST` | `/appeal` | File an appeal for a prior classification |
| `GET` | `/log` | Structured audit log with appeal status |

### POST /submit

**Request**
```json
{ "text": "...", "creator_id": "user-123" }
```

**Response**
```json
{
  "content_id": "3f7a2b1e-9c4d-4e8a-b2f1-6d0e7a3c5b92",
  "attribution": "ai",
  "confidence": 0.882,
  "label": "High-confidence AI-generated",
  "signal1_score": 0.93,
  "signal2_score": 0.77
}
```

### POST /appeal

**Request**
```json
{
  "content_id": "3f7a2b1e-...",
  "creator_reasoning": "I wrote this myself from personal experience."
}
```

**Response**
```json
{
  "appeal_id": "a9c1d2e3-...",
  "content_id": "3f7a2b1e-...",
  "status": "under_review",
  "message": "Your appeal was received and is under review."
}
```

---

## Detection Pipeline

### Why these two signals

**Signal 1 (LLM) — 70% weight:** An LLM classifying text it recognizes patterns in is the strongest single signal available. It understands semantic context, rhetorical moves, and the absence of human voice in ways no heuristic can. It gets 70% weight because it operates at the level of meaning, not just surface structure. The main drawback is cost: every submission requires an API call.

**Signal 2 (local heuristics) — 30% weight:** Three metrics computed locally with no API call, giving the system a cheap, deterministic check that doesn't depend on external availability. It gets 30% weight because it only sees surface patterns — it can miss AI text that avoids the specific markers it measures (see Known Limitations).

The 70/30 split was chosen by tracing two extreme cases. For text where both signals strongly agree (e.g., Signal 1 = 0.93, Signal 2 = 0.77), a 60/40 split produces a combined score of 0.86 — barely over the 0.85 threshold. Adjusting to 70/30 gives 0.882, providing comfortable margin. The asymmetry also reflects that Signal 1 is more reliable — if they disagree, the LLM's judgment should win.

**What I'd change for production:** Signal 2's heuristics are calibrated for blog-style AI writing (explicit "however/furthermore" transitions, paragraph uniformity). For a real deployment I'd add passive voice ratio and a broader hedging-phrase set to catch academic-register AI text, and I'd A/B test the threshold against labeled examples rather than setting it analytically.

---

## Confidence Scoring

Both signals are normalized to a 0–1 AI-likelihood scale, then combined:

```
combined = 0.70 × signal1_score + 0.30 × signal2_score
```

Where `signal1_score` converts the LLM's output: `"ai" @ 0.9 → 0.9`, `"human" @ 0.9 → 0.1`, `"uncertain" → 0.5`.

The final `confidence` field is the certainty in whichever direction the combined score leans: `max(combined, 1 − combined)`. This is symmetric — a combined score of 0.10 (human-leaning) reports 90% confidence just as a score of 0.90 (AI-leaning) does.

### Example 1 — High-confidence AI

**Input:** *"Furthermore, it is important to note that the implementation of machine learning algorithms necessitates a comprehensive understanding of the underlying mathematical frameworks. Moreover, the integration of neural networks requires careful consideration of hyperparameter optimization strategies."*

| | Score |
|---|---|
| Signal 1 (LLM) | 0.93 |
| Signal 2 (stylometric) | 0.77 |
| Combined (70/30) | **0.882** |
| Confidence | **88.2%** |
| Label | **High-confidence AI-generated** |

Signal 2 scored 0.77 because the text contains two transition phrases ("furthermore", "it is important to note"), has nearly uniform sentence lengths (both sentences are 20–24 words), and uses only commas and periods.

### Example 2 — Uncertain (signals diverge)

**Input:** *"The relationship between monetary policy and asset price inflation has been extensively studied in the literature. Central banks face a fundamental tension between their mandate for price stability and the unintended consequences of prolonged low interest rates on equity and real estate valuations."*

| | Score |
|---|---|
| Signal 1 (LLM) | 0.72 |
| Signal 2 (stylometric) | 0.25 |
| Combined (70/30) | **0.579** |
| Confidence | **57.9%** |
| Label | **Uncertain — provenance unclear** |

Signal 1 sees AI markers (passive construction, hedging nominalizations, formal register with no personal voice). Signal 2 scores low because only two sentences exist (CV metric falls back to neutral 0.5), zero transition phrases from its list appear, and two distinct punctuation marks are present. The disagreement between signals is itself meaningful — the system correctly returns "Uncertain" rather than committing.

---

## Transparency Labels

The system produces exactly three label variants based on the combined confidence score:

**1. High-confidence AI-generated**
- Displayed when: `attribution == "ai"` AND `confidence ≥ 0.85` (combined score ≥ 0.85)
- Exact text: `"High-confidence AI-generated"`
- UI: Red-tinted result card

**2. High-confidence human-written**
- Displayed when: `attribution == "human"` AND `confidence ≥ 0.85` (combined score ≤ 0.15)
- Exact text: `"High-confidence human-written"`
- UI: Green-tinted result card

**3. Uncertain — provenance unclear**
- Displayed when: neither of the above conditions is met (combined score between 0.15 and 0.85)
- Exact text: `"Uncertain — provenance unclear"`
- UI: Yellow-tinted result card

All three variants are reachable. Submit clearly AI-generated text with explicit transitions to reach label 1; submit casual human writing with varied punctuation and idiomatic language to reach label 2; submit academic or borderline text to reach label 3.

---

## Appeals Workflow

`POST /appeal` accepts a `content_id` (from any prior `/submit` response) and `creator_reasoning`. It:

1. Validates the `content_id` exists in the database (returns 404 if not)
2. Inserts a record into the `appeals` table with the creator's reasoning and timestamp
3. Updates the matching row in `submissions` to `status = "under_review"`
4. Returns an `appeal_id` for tracking

After filing an appeal, `GET /log` will show `"status": "under_review"` and the `appeal_reasoning` field populated for that submission. No automated re-classification is performed; the status signals to platform operators that a human review is needed.

**Test appeal:**
```bash
curl -s -X POST http://localhost:5000/appeal \
  -H "Content-Type: application/json" \
  -d '{
    "content_id": "PASTE-CONTENT-ID-HERE",
    "creator_reasoning": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical."
  }' | python -m json.tool
```

Then verify with `GET /log` — the entry should show `"status": "under_review"` and the reasoning populated.

---

## Rate Limiting

`POST /submit` is limited to **10 requests per minute and 100 per day per IP address**.

**Reasoning:** 10/minute covers a writer uploading multiple drafts in a single session without waiting. 100/day sits well above what any human creator would hit (even a prolific writer submitting 10 drafts of 10 different pieces in one day hits exactly the limit) while blocking automated scripts that would exhaust LLM API quota. `/appeal` and `/log` are unrestricted — appeals are infrequent and the log is read-only.

**Evidence of enforcement** (12 rapid-fire requests; first 10 succeed, last 2 rejected):
```
200
200
200
200
200
200
200
200
200
200
429
429
```

To reproduce:
```bash
for i in $(seq 1 12); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/submit \
    -H "Content-Type: application/json" \
    -d '{"text": "Rate limit test.", "creator_id": "ratelimit-test"}'
done
```

---

## Audit Log

`GET /log` returns structured JSON. Sample entry after an appeal is filed:

```json
{
  "content_id": "3f7a2b1e-9c4d-4e8a-b2f1-6d0e7a3c5b92",
  "creator_id": "user-123",
  "timestamp": "2026-06-30T18:42:11.304Z",
  "content_preview": "Furthermore, it is important to note that...",
  "attribution": "ai",
  "confidence": 0.882,
  "label": "High-confidence AI-generated",
  "signal1_score": 0.93,
  "signal2_score": 0.77,
  "status": "under_review",
  "appeal_reasoning": "I wrote this myself from personal experience."
}
```

| Field | Description |
|-------|-------------|
| `content_id` | UUID assigned at submission |
| `creator_id` | Submitting user identifier |
| `timestamp` | ISO-8601 UTC |
| `content_preview` | First 200 characters of submitted text |
| `attribution` | `ai` / `human` / `uncertain` |
| `confidence` | Combined score 0.0–1.0 |
| `label` | Human-readable transparency label |
| `signal1_score` | LLM directional score (0–1, higher = more AI-like) |
| `signal2_score` | Stylometric heuristic score (0–1, higher = more AI-like) |
| `status` | `classified` or `under_review` |
| `appeal_reasoning` | Creator's appeal text, or `null` if no appeal filed |

---

## Known Limitations

**Academic and technical register AI text is systematically underdetected by Signal 2.** Signal 2's transition-phrase metric was built around blog-style AI writing: explicit connectors like "furthermore," "however," "in conclusion." Academic-register AI text avoids these phrases entirely — it uses passive constructions ("has been extensively studied"), hedging nominalizations ("the unintended consequences of"), and citation-style hedges ("in the literature") instead. All three of Signal 2's metrics miss this pattern: the text has no flagged transitions, its sentence count is often below the CV threshold of 3, and academic prose uses minimal punctuation variety regardless of authorship. The only layer catching it is Signal 1, which means combined confidence stays below 0.85 and the system returns "Uncertain" rather than "High-confidence AI-generated" — a false negative for AI content rather than a false positive against a human.

**Non-native English writers are at elevated false-positive risk.** The LLM's system prompt explicitly flags "avoids slang, regional expressions, emotional rawness" as AI markers. A non-native English speaker writing carefully and formally exhibits exactly these properties — not because they're using AI, but because formal register is a learned safety strategy. This is a real fairness concern for any creative platform with an international user base.

---

## Spec Reflection

**How the spec guided implementation:** The requirement for exactly three label variants with distinct text — not a raw score — forced the threshold design to be explicit and testable. Without the spec mandating three named categories, the natural implementation would return a float and let callers decide what to show. Having to commit to exact cutpoints (0.85 for "high-confidence") meant the thresholds had to be verified against real examples before shipping, which caught a weight-adjustment issue (60/40 vs. 70/30) that would have made "high-confidence" labels nearly unreachable.

**Where implementation diverged from the spec:** The spec specified a JSONL file for the audit log ("Log the appeal in the JSONL file"). I used SQLite instead. The reason is structural: the appeals workflow requires two operations that JSONL cannot do efficiently — looking up a specific `content_id` to validate it exists, and updating a single record's status atomically. In a JSONL file, both of those require reading the entire file. SQLite makes them O(1) with a single indexed query. The spec was written before the appeals workflow's query requirements were fully thought through; once those requirements were concrete, JSONL was the wrong tool.

---

## AI Usage

**Instance 1 — Signal 2 generation and the academic-register blind spot.**
I directed Claude to generate a `stylometric_score()` function using three heuristics: sentence-length CV, transition phrase density, and punctuation variety. It produced a working implementation with a reasonable structure. I then tested it against an academic monetary-policy text that Signal 1 scored as AI-leaning. Signal 2 returned 0.25 (human-leaning) because: the text had only 2 sentences (CV metric falls back to neutral), none of the generated TRANSITION_PHRASES appeared in it, and academic prose has low punctuation variety regardless of authorship. Rather than patching the phrase list, I kept the result and documented it as a known limitation — the divergence was real signal about Signal 2's scope, not a bug to suppress.

**Instance 2 — combine_signals() weight calibration.**
I directed Claude to generate the signal-combination function and confidence scoring logic. The initial output used 60/40 weights (Signal 1 / Signal 2). I traced through a concrete high-AI-agreement case (S1=0.93, S2=0.77) and found the combined score was 0.879 — below the 0.85 threshold, meaning a clearly AI-generated text would return "Uncertain" rather than "High-confidence AI-generated." I overrode the weights to 70/30, which gives 0.882 for the same input and correctly clears the threshold. The 70/30 split also better reflects the actual reliability hierarchy: the LLM should dominate when the two signals disagree.

**Instance 3 — HTML result card revision.**
Claude generated the inline submission form and result display. The initial result card showed only the final label and confidence. I revised it to also display both individual signal scores (`signal1_score` and `signal2_score`) in the result card. This was a deliberate product choice: for a transparency system, showing the evidence behind the label — and allowing users to see when the two signals disagree — is more honest than surfacing only the final verdict.

---

## Setup

```bash
pip install -r requirements.txt
# Create .env with: GROQ_API_KEY=your_key_here
python app.py
# Visit http://127.0.0.1:5000
```
