# Health Profiler API

Short: a backend demo that accepts typed or scanned lifestyle surveys (OCR), extracts fields, enforces a guardrail for incomplete data, extracts risk factors, computes a deterministic non‑diagnostic risk score, and returns concise, actionable recommendations.

## Quick checklist
- Node 18+ installed
- .env with API_KEY (optional — AI calls require it)
- Port: 3000 by default

## Repo layout (key files)
- index.js — Express server and /profile endpoint (image/text input)
- aiService.js — parsing, local scoring, AI recommendation call + safe parsing
- src/services/riskScoringService.js — deterministic evidence-informed scoring
- src/services/recommendationService.js — concise local fallback recommendations and required-fields guardrail
- package.json — run scripts

## Setup

1. Clone repo and install:
```bash
git clone <your-repo-url>
cd health-profiler-api
npm install
```

2. Create `.env` in project root:
```
API_KEY=your_google_generative_ai_key  # optional if you want AI recommendations
```

3. Run server:
```bash
node index.js
```

4. (Optional) Expose local server with ngrok:
```bash
ngrok http 3000
# copy the https://... URL and use it in Postman/curl
```

## Endpoint

POST /profile
- Accepts multipart/form-data
  - survey_image (File) — optional image to OCR
  - survey_data (Text) — optional raw JSON or free text survey
- Returns JSON (see examples)

Behavior summary:
- Step 1 — parse answers from text or OCR; returns answers, missing_fields, confidence.
- Guardrail — if fewer than 2 of the required fields (age, smoker, exercise, diet) are present → immediate stop and return the guardrail JSON.
- Step 2 — extract factors (e.g., smoking, poor_diet, low_exercise) with confidence.
- Step 3 — deterministic risk scoring (0–100), non-diagnostic, includes rationale.
- Step 4 — concise, non-diagnostic recommendations (short strings). No citations printed.

HTTP responses:
- success: 200 with structured profile
- incomplete profile guardrail: 200 with {"status":"incomplete_profile","reason":">50% fields missing","missing_fields":[...]}
- general failure: 200 with {"status":"error","reason":"..."}

(You can change status code mapping to 400 in index.js if desired.)

## Example requests

1) Text input (curl)
```bash
curl -X POST http://localhost:3000/profile \
  -F 'survey_data={"age":42,"smoker":true,"exercise":"rarely","diet":"high sugar"}'
```

2) Image upload (with OCR via Tesseract)
```bash
curl -X POST http://localhost:3000/profile \
  -F "survey_image=@/path/to/survey.jpg"
```

## Example outputs

1) Step 1 parsed output (successful parse)
```json
{
  "answers": {"age":42,"smoker":true,"exercise":"rarely","diet":"high sugar"},
  "missing_fields": [],
  "confidence": 0.92
}
```

2) Guardrail (if >50% required fields missing)
```json
{
  "status":"incomplete_profile",
  "reason":">50% fields missing",
  "missing_fields":["age","diet"]
}
```

3) Factors extraction
```json
{
  "factors":["smoking","poor_diet","low_exercise"],
  "factor_confidence":0.88
}
```

4) Risk classification (non-diagnostic)
```json
{
  "risk_level":"high",
  "score":78,
  "rationale":["smoking","high sugar diet","low activity"]
}
```

5) Final recommendations (concise)
```json
{
  "risk_level":"high",
  "factors":["smoking","poor_diet","low_exercise"],
  "recommendations":["Quit smoking","Reduce sugar intake","Walk 30 mins daily"],
  "status":"ok"
}
```

## Implementation notes
- OCR: Tesseract.js (index.js) reads uploaded image buffer and returns text.
- Parsing: `aiService.parseSurveyTextToObject` normalizes common key names and values.
- Guardrail: `recommendationService.generateRecommendations` enforces the "at least 2 of 4 fields" rule and throws a structured error if rule fails; aiService maps that to the exact guardrail JSON.
- Scoring: `riskScoringService.computeHeartAttackRisk` is a deterministic heuristic (age bands, smoking, diabetes, HTN, cholesterol, activity, diet, BMI) and returns normalized 0–100 and detected factors. This is intentionally non‑diagnostic.
- Recommendations: AI used only for concise recommendations (if API_KEY present); fallback to `recommendationService` short messages.

## Testing
- Manual via curl/Postman (examples above).
- Recommended unit tests:
  - parseSurveyTextToObject parsing variants (JSON vs free-text)
  - required-fields guardrail
  - computeHeartAttackRisk with sample profiles

## Limitations & disclaimers
- Not a clinical tool. For clinical risk use validated calculators (Framingham, ASCVD, SCORE) and measured vitals/labs.
- Scoring heuristics are approximations for prioritization only.
- AI outputs may vary; code normalizes and falls back to deterministic messages if parsing fails.
