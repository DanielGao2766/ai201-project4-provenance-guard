# Provenance Guard

A backend service that classifies submitted creative text as human-written or AI-generated, returns a confidence-scored transparency label, and manages creator appeals.

---

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Browser submission form |
| `POST` | `/submit` | Classify text; returns attribution, confidence, label, signal scores |
| `POST` | `/appeal` | File an appeal for a prior classification |
| `GET` | `/log` | Audit log of all submissions with appeal status |

### POST /submit

**Request**
```json
{ "text": "...", "creator_id": "user-123" }
```

**Response**
```json
{
  "content_id": "3f7a2b1e-...",
  "attribution": "ai",
  "confidence": 0.91,
  "label": "High-confidence AI-generated",
  "signal1_score": 0.93,
  "signal2_score": 0.84
}
```

Labels:
- `"High-confidence AI-generated"` — combined score ≥ 0.85
- `"High-confidence human-written"` — combined score ≤ 0.15
- `"Uncertain — provenance unclear"` — everything in between

### POST /appeal

**Request**
```json
{ "content_id": "3f7a2b1e-...", "creator_reasoning": "I wrote this myself." }
```

**Response**
```json
{
  "appeal_id": "a9c1...",
  "content_id": "3f7a2b1e-...",
  "status": "under_review",
  "message": "Your appeal was received and is under review."
}
```

After a successful appeal, `GET /log` will show `"status": "under_review"` and the `appeal_reasoning` field populated for that entry.

---

## Detection Pipeline

Two independent signals are combined (70 / 30 weighted) into a single score:

- **Signal 1 — LLM stylometric analysis (Groq `llama-3.3-70b-versatile`)**: Evaluates writing for AI-generation markers — uniform rhythm, hedging language, absence of personal voice, predictable structure.
- **Signal 2 — Stylometric heuristics**: Three surface metrics computed locally with no additional API call:
  - Sentence-length coefficient of variation (low variance → AI)
  - Formal transition phrase density (higher density → AI)
  - Punctuation variety (low variety → AI)

---

## Rate Limiting

`POST /submit` is limited to **10 requests per minute and 100 per day per IP address**.

**Reasoning:** A working writer submitting drafts for review would rarely exceed 10 submissions in a single minute — that covers uploading multiple chapter revisions in a session. The 100/day ceiling prevents a single account from exhausting LLM quota via automation while remaining well above what any human creator would hit organically. These limits sit on the submission endpoint only; `/appeal` and `/log` are unrestricted.

**Evidence of enforcement** (HTTP status codes from 12 rapid-fire requests — first 10 succeed, last 2 are rejected):
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

`GET /log` returns structured JSON. Each entry captures:

| Field | Description |
|-------|-------------|
| `content_id` | UUID for this submission |
| `creator_id` | Submitting user |
| `timestamp` | ISO-8601 UTC |
| `content_preview` | First 200 characters of submitted text |
| `attribution` | `ai` / `human` / `uncertain` |
| `confidence` | Combined score 0.0–1.0 |
| `label` | Human-readable transparency label |
| `signal1_score` | LLM signal (0–1, higher = AI) |
| `signal2_score` | Stylometric signal (0–1, higher = AI) |
| `status` | `classified` or `under_review` |
| `appeal_reasoning` | Creator's appeal text, or `null` if no appeal filed |

---

## Setup

```bash
pip install -r requirements.txt
# Add GROQ_API_KEY to .env
python app.py
```
