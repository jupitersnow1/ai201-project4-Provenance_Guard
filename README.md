# Provenance Guard

Provenance Guard is a backend application that classifies submitted text as **likely AI-generated**, **likely human-written**, or **uncertain**. Instead of only returning a prediction, it also provides a confidence score, a plain-language transparency label, and a way for creators to appeal classifications they believe are incorrect. Every submission and appeal is recorded in a structured audit log to make the decision process more transparent.

The project combines an LLM-based detector with a stylometric analysis of the text, then merges both signals into a single confidence score before returning the final result.

For the complete design decisions, architecture diagrams, and implementation plan, see [`planning.md`](planning.md).

---

# Getting Started

### 1. Create and activate a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate
```

### 2. Install the required packages

```bash
pip install -r requirements.txt
```

### 3. Create a `.env` file

In the project root, create a `.env` file containing your Groq API key:

```text
GROQ_API_KEY=your_key_here
```

### 4. Run the application

```bash
python app.py
```

The application runs on **http://localhost:5001**.

> **Note:** Port **5000** is commonly used by the macOS AirPlay Receiver, so this project uses port **5001** by default.

---

# Architecture

Every submission passes through the same pipeline before a classification is returned.

```text
POST /submit
      ↓
Signal A (LLM via Groq)
      ↓
Signal B (Stylometric Heuristics)
      ↓
Confidence Scoring (Weighted Combination)
      ↓
Transparency Label
      ↓
Audit Log
      ↓
JSON Response
```

When a creator submits an appeal, the system doesn't overwrite the original classification. Instead, it marks the submission as **under_review**, records the creator's reasoning, and adds a new entry to the audit log.

```text
POST /appeal
      ↓
Lookup Original Classification
      ↓
Update Status (under_review)
      ↓
Write Appeal to Audit Log
      ↓
JSON Response
```

The full Mermaid diagrams for both workflows are available in [`planning.md`](planning.md#architecture).

# Detection Signals

Instead of relying on a single detector, Provenance Guard combines two independent signals. Each one looks at the text differently, so using both together helps produce a more balanced classification.

## Signal A — LLM-based Classification (Groq, `llama-3.3-70b-versatile`)

The first signal uses Groq's LLM to determine whether a piece of text appears to be AI-generated or human-written based on its semantic meaning and writing style.

It returns:

```json
{
  "attribution": "likely_ai | likely_human | uncertain",
  "llm_score": 0.81
}
```

where `llm_score` is a confidence value between **0** and **1**.

**Blind spot:** This approach can struggle with AI-generated text that has been heavily edited by a person. Even a few rewritten sentences can sometimes make the model classify the text as human-written. Since the model is probabilistic, its output can also vary slightly between runs.

---

## Signal B — Stylometric Heuristics (Pure Python)

The second signal doesn't rely on an LLM. Instead, it analyzes measurable characteristics of the writing itself, including:

* average sentence length
* sentence-length variance
* type-token ratio (vocabulary diversity)

Generally, AI-generated writing tends to be more consistent in structure, while human writing usually has more variation.

It returns:

```json
{
  "stylometric_score": 0.63,
  "metrics": { ... }
}
```

where `stylometric_score` is also a value between **0** and **1**.

**Blind spot:** Formal human writing, such as academic papers or legal documents, often has consistent sentence structure and vocabulary. Because of this, the heuristics can sometimes mistake it for AI-generated writing. The metrics also become less reliable on very short submissions because there isn't enough text to calculate meaningful statistics.

---

These two signals are intentionally independent. One evaluates the meaning and style of the writing using an LLM, while the other measures structural patterns with simple text statistics. Because they approach the problem differently, disagreements between them often provide useful information instead of simply being noise.

# Confidence Scoring

Both detection signals return scores between **0** and **1**. Those scores are combined into a single confidence score using a weighted average.

```python
confidence = 0.7 * llm_score + 0.3 * stylometric_score
```

The LLM signal is weighted slightly higher (**70%**) because it captures semantic and stylistic information that the heuristic approach cannot. The stylometric signal still contributes (**30%**) by providing a second, independent perspective based on measurable writing patterns.

The final confidence score is mapped to one of three categories:

| Confidence Score | Classification        |
| ---------------- | --------------------- |
| `>= 0.75`        | High-confidence AI    |
| `0.40 – 0.74`    | Uncertain             |
| `< 0.40`         | High-confidence Human |

Using three categories instead of a simple AI-or-human prediction allows the system to be honest when the evidence is mixed rather than forcing a potentially misleading answer.

# Validation

To evaluate the system, I tested it with four different types of inputs:

* clearly AI-generated text
* clearly human-written text
* formal human writing
* lightly edited AI-generated text

These examples helped verify that both detection signals behaved as expected and that the confidence score changed depending on the input instead of always producing similar results.

### Example 1 — Clearly AI-generated Text

**Confidence:** **0.672**
**Label:** *Uncertain*

```text
"Artificial intelligence represents a transformative paradigm shift in modern
society. It is important to note that while the benefits of AI are numerous,
it is equally essential to consider the ethical implications. Furthermore,
stakeholders across various sectors must collaborate to ensure responsible
deployment."
```

* **LLM score:** `0.80`
* **Stylometric score:** `0.372`
* **Final confidence:** `0.672`

The LLM correctly identified the text as AI-generated, but the stylometric score was much lower because the sample only contained three sentences. With so little text, the structural metrics become less reliable, which lowered the overall confidence and placed the result in the **Uncertain** category.

---

### Example 2 — Clearly Human-written Text

**Confidence:** **0.249**
**Label:** *High-confidence Human*

```text
"ok so i finally tried that new ramen place downtown and honestly?
underwhelming. the broth was fine but they put WAY too much sodium in it
and i was thirsty for like three hours after. my friend got the spicy
version and said it was better. probably wont go back unless someone
drags me there"
```

* **LLM score:** `0.23`
* **Stylometric score:** `0.293`
* **Final confidence:** `0.249`

Both detection signals agreed that this sample was likely written by a person, producing a much lower confidence score than the AI-generated example.

Overall, the test results showed that the confidence score changes in meaningful ways depending on the input instead of acting like a simple binary classifier.

The testing also confirmed a couple of the limitations I expected before building the project. The LLM sometimes classified formal human writing as AI-generated, while lightly edited AI text occasionally looked human-written. I decided not to adjust the scoring formula just to improve one specific example because that would risk overfitting the system instead of making it more reliable overall. Instead, I documented these behaviors in the **Known Limitations** section.

---

# Transparency Labels

After the confidence score is calculated, the system returns one of three transparency labels.

> **High-confidence AI:** "High-confidence AI: Our system has high confidence this piece was generated by AI. If you believe this is incorrect, you may submit an appeal for human review."

> **Uncertain:** "Uncertain: The system could not confidently determine whether this was written by AI or a human. A human review may help resolve this."

> **High-confidence Human:** "High-confidence human: Our system has high confidence this piece was written by a human. If you believe this is incorrect, you may still appeal."

I wanted the labels to be easy to understand while also being honest about the system's confidence. Even when the model is confident, users still have the option to submit an appeal if they believe the classification is incorrect.

---

# Appeals Workflow

Creators can challenge a classification by sending a request to the `POST /appeal` endpoint.

The request includes:

```json
{
  "content_id": "...",
  "creator_id": "...",
  "creator_reasoning": "..."
}
```

When an appeal is submitted, the system:

1. Finds the original classification using the `content_id`.
2. Updates the submission status to `under_review`.
3. Records the creator's reasoning in a new audit log entry without overwriting the original classification.
4. Returns a confirmation message.

Appeals are intended for human review only. The system does not automatically reclassify content after an appeal is submitted.

### Example

```bash
curl -s -X POST http://localhost:5001/appeal \
  -H "Content-Type: application/json" \
  -d '{"content_id": "f3fcade4-754e-45d8-b8d4-4ec86d32117d", "creator_id": "test-user-1", "creator_reasoning": "I wrote this myself from personal experience."}'
```

```json
{
  "content_id": "f3fcade4-754e-45d8-b8d4-4ec86d32117d",
  "message": "Appeal received",
  "status": "under_review"
}
```

---

# Rate Limiting

To prevent abuse, the `/submit` endpoint is limited to **10 requests per minute** and **100 requests per day** for each IP address using Flask-Limiter.

```python
@app.route("/submit", methods=["POST"])
@limiter.limit("10 per minute;100 per day")
def submit():
    ...
```

I chose these limits because they allow plenty of room for normal testing while still preventing someone from flooding the API with requests. Since every submission sends a request to the Groq API, keeping the rate limit reasonable also helps avoid unnecessary usage of the free API tier.

To verify the rate limiter was working correctly, I sent 12 requests in quick succession. The first 10 requests succeeded, while the final 2 returned a **429 Too Many Requests** response.

```text
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

# Audit Log

Every classification and appeal is saved as a structured JSON entry in `audit_log.jsonl`. The `GET /log` endpoint returns the most recent entries, making it easy to review how the system reached its decisions.

Below are sample log entries showing an initial classification, the corresponding appeal, and another classification example.

```json
{
  "timestamp": "2026-06-30T01:18:19.972036+00:00",
  "content_id": "f3fcade4-754e-45d8-b8d4-4ec86d32117d",
  "creator_id": "test-user-1",
  "attribution": "uncertain",
  "confidence": 0.672,
  "signal_outputs": {
    "llm": {
      "attribution": "likely_ai",
      "llm_score": 0.8
    },
    "stylometric": {
      "stylometric_score": 0.372,
      "metrics": {
        "avg_sentence_length": 16,
        "sentence_length_variance": 36,
        "type_token_ratio": 0.875,
        "punctuation_density": 0.005
      }
    }
  },
  "label": "Uncertain: The system could not confidently determine whether this was written by AI or a human. A human review may help resolve this.",
  "status": "classified"
}
```

```json
{
  "timestamp": "2026-06-30T01:18:20.102716+00:00",
  "content_id": "f3fcade4-754e-45d8-b8d4-4ec86d32117d",
  "creator_id": "test-user-1",
  "attribution": "uncertain",
  "confidence": 0.672,
  "signal_outputs": {
    "llm": {
      "attribution": "likely_ai",
      "llm_score": 0.8
    },
    "stylometric": {
      "stylometric_score": 0.372,
      "metrics": {
        "avg_sentence_length": 16,
        "sentence_length_variance": 36,
        "type_token_ratio": 0.875,
        "punctuation_density": 0.005
      }
    }
  },
  "label": "Uncertain: The system could not confidently determine whether this was written by AI or a human. A human review may help resolve this.",
  "status": "under_review",
  "appeal_reasoning": "I wrote this myself from personal experience."
}
```

The appeal entry carries forward the original `signal_outputs` so each log entry is self-contained — a reviewer doesn't need to cross-reference an earlier entry to see what evidence drove the decision being appealed.

```json
{
  "timestamp": "2026-06-30T01:24:43.875186+00:00",
  "content_id": "5ce66c96-8523-4901-b623-52aae617a3a5",
  "creator_id": "test-human",
  "attribution": "likely_human",
  "confidence": 0.249,
  "signal_outputs": {
    "llm": {
      "attribution": "likely_human",
      "llm_score": 0.23
    },
    "stylometric": {
      "stylometric_score": 0.293,
      "metrics": {
        "avg_sentence_length": 11,
        "sentence_length_variance": 45.2,
        "type_token_ratio": 0.873,
        "punctuation_density": 0.0
      }
    }
  },
  "label": "High-confidence human: Our system has high confidence this piece was written by a human. If you believe this is incorrect, you may still appeal.",
  "status": "classified"
}
```

---

# Known Limitations

Like any AI detection system, Provenance Guard has limitations.

### Short submissions

The stylometric signal works best when there is enough text to analyze. Very short submissions don't provide enough data for statistics like sentence-length variance or vocabulary diversity to be reliable.

For example, the three-sentence AI-generated sample from the validation section received a lower stylometric score, which pulled the overall confidence into the **Uncertain** category even though the LLM strongly classified it as AI-generated. I chose not to adjust the scoring formula just to improve that one example because doing so would likely overfit the model instead of improving its overall performance.

### Formal writing

The LLM detector can sometimes mistake formal human writing—such as academic or policy documents—for AI-generated text. On the other hand, AI-generated text that has been heavily edited by a person may appear human-written. Both of these behaviors appeared during testing and matched the limitations I expected while planning the project.

---

# Reflection

## What went well

One decision that made implementation much easier was defining the transparency labels early in `planning.md`. Since the wording and confidence thresholds were already finalized, implementing the label-mapping logic in `app.py` was mostly a matter of translating those decisions into code instead of making design choices while programming.

## What changed during implementation

One thing I learned during testing was that short AI-generated submissions created an additional edge case I hadn't fully anticipated. The planning document focused on edited AI text confusing the LLM, but I also found that the stylometric signal becomes much less reliable when there are only a few sentences to analyze.

Rather than changing the scoring formula to fit one specific example, I decided to document this limitation instead. I think that better reflects the system's actual behavior and keeps the evaluation more honest.

---

# AI Usage

I used AI throughout the project as a development assistant, but I reviewed and tested everything before adding it to the final implementation.

Specifically, I used AI to:

* help develop the prompt used for Groq's `llama-3.3-70b-versatile`, including enforcing structured JSON output (`attribution`, `llm_score`, and an optional `short_explanation`)
* improve the JSON parsing logic after discovering that the model occasionally returned extra text around the JSON response
* generate an initial version of the stylometric scoring formula based on sentence statistics and vocabulary diversity

After testing the scoring with multiple examples, I decided not to modify the heuristic simply to improve one specific test case. Instead, I documented the limitation so the behavior of the system remains transparent.

---

# API Reference

### `POST /submit`

```json
// Request
{
  "text": "...",
  "creator_id": "..."
}

// Response
{
  "content_id": "...",
  "attribution": "likely_ai | likely_human | uncertain",
  "confidence": 0.0,
  "label": "..."
}
```

---

### `POST /appeal`

```json
// Request
{
  "content_id": "...",
  "creator_id": "...",
  "creator_reasoning": "..."
}

// Response
{
  "message": "Appeal received",
  "content_id": "...",
  "status": "under_review"
}
```

---

### `GET /log`

```json
{
  "entries": [
    /* most recent audit log entries */
  ]
}
```
