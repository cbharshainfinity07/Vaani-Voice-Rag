# Vaani — HH Goa 2026 Voice-Enabled RAG

A voice-enabled Retrieval-Augmented Generation system built for **HH Goa 2026 Shortlisting Task 2**.

> **Current state:** the repository is runnable locally with a deterministic demo corpus and a no-key grounded generator. Add a Sarvam key for the required voice path, reuse the OpenCode Go credential for GPT answers, and replace the demo corpus with the supplied `ai4bharat/MSMARCO-XI` export before submission.

## Exact task mapping

| Brief requirement | Implementation |
|---|---|
| Voice → STT → retrieval → answer | Browser `MediaRecorder` → Sarvam Saaras → hybrid RAG → cited answer |
| STT provider | `SarvamSpeechToText`, `saaras:v4` by default; endpoint is `https://api.sarvam.ai/speech-to-text` |
| Vast chunking strategy | paragraph, sentence-window overlap, token overlap, semantic-boundary, metadata-aware, and parent-child IDs |
| Vector DB retrieval | Local persistent vector index for zero-setup demo; optional local Qdrant adapter for deployment |
| Latency target | Local deterministic query path is below 200 ms; the current remote GPT path is measured separately and is not below 200 ms |
| Latency analytics | `scripts/benchmark.py` reports P50/P70/P100 over a query set and writes raw per-query rows |
| Harness | typed request/response schemas, stage timers, retries, per-step timeout, generator fallback, structured errors |
| Guardrails | unsafe/prompt-injection filtering, retrieval sufficiency gate, citation requirement, answer-context grounding check, abstention |

## Architecture

```text
Browser microphone / typed fallback
              │
              ▼
      Sarvam Saaras STT (voice)
              │
              ▼
  Typed PipelineRequest + input guardrails
              │
              ├──────────────┐
              ▼              ▼
      BGE-M3 / hash      BM25 lexical
      vector search       search
              └──────┬───────┘
                     ▼
         reciprocal-rank fusion + rerank
                     │
                     ▼
      Groq/OpenAI-compatible grounded LLM
        (or deterministic local generator)
                     │
                     ▼
      citation + grounding guardrail → answer/abstain
```

### Why Sarvam Saaras

The brief requires choosing **Sarvam or ElevenLabs**. This build chooses Sarvam because the task is India-focused and the Saaras API supports Indian languages and English. The adapter uses the documented `api-subscription-key` header and multipart `file` field. Confirm the model/version visible in your Sarvam dashboard before the final submission; the default here is `saaras:v4`.

## Quickstart

From this directory:

```bash
python -m pip install -r requirements.txt
python -m pip install pytest
python -m pytest -q
python -m uvicorn app.main:app --reload --port 8000
```

Open <http://127.0.0.1:8000>. The typed demo works without API keys.

### Configure APIs

Copy `.env.example` to `.env` and add keys. Never commit `.env`.

```env
SARVAM_API_KEY=your_sarvam_key
SARVAM_MODEL=saaras:v4

# Reuse the already-authenticated OpenCode Go subscription
GENERATION_PROVIDER=opencode-go
OPENCODE_GO_MODEL=gpt-5.6-luna
OPENCODE_GO_BASE_URL=https://opencode.ai/zen/go/v1/responses
```

The OpenCode Go adapter reads the existing credential from:

```text
~/.local/share/opencode/auth.json
```

It does not copy the OpenCode credential into this repository. The current implementation uses the official OpenCode Go Responses endpoint and has been smoke-tested with `gpt-5.6-luna`.

Alternative generation providers are also supported:

```env
GROQ_API_KEY=your_groq_key
GENERATION_MODEL=llama-3.1-8b-instant
GENERATION_BASE_URL=https://api.groq.com/openai/v1
```

- **Sarvam** is used for the required speech-to-text step.
- **OpenCode Go GPT 5.6 Luna** is the preferred answer model when an OpenCode Go subscription is available.
- Groq is an optional low-latency OpenAI-compatible alternative.
- Without a generation credential, the app uses a deterministic local generator that quotes the top retrieved evidence and is useful for demos/benchmarks.

## Use the supplied dataset

The current task document names `ai4bharat/MSMARCO-XI`. The Hub README lists **14 Indic language configurations**:

| Config | Language | Sarvam input code |
|---|---|---|
| `as` | Assamese | `as-IN` |
| `bn` | Bengali | `bn-IN` |
| `gu` | Gujarati | `gu-IN` |
| `hi` | Hindi | `hi-IN` |
| `kn` | Kannada | `kn-IN` |
| `ml` | Malayalam | `ml-IN` |
| `mr` | Marathi | `mr-IN` |
| `ne` | Nepali | `ne-IN` |
| `or` | Odia | `od-IN` |
| `pa` | Punjabi | `pa-IN` |
| `sa` | Sanskrit | `sa-IN` |
| `ta` | Tamil | `ta-IN` |
| `te` | Telugu | `te-IN` |
| `ur` | Urdu | `ur-IN` |

English is also available for speech input as `en-IN`, and `unknown` enables automatic language detection. The frontend language selector contains all 14 dataset languages plus English.

To index every available language configuration in one run:

```bash
python -m pip install -e ".[production]"
python scripts/ingest.py \
  --dataset ai4bharat/MSMARCO-XI \
  --all-configs \
  --split validation \
  --backend bge-m3 \
  --vector-backend qdrant \
  --qdrant-path storage/qdrant
```

The public Hub file listing currently exposes a Telugu validation file but not a Telugu training parquet file, so `--split validation` is the correct all-language command. The parquet files are large and this full run is long-running; run it in a persistent terminal/background job and verify the generated manifest. The local session verified the Hindi path end to end and verified all 14 validation files through the Hub metadata, but did not complete the full 14-language download.

To index one language for a faster test:

```bash
python scripts/ingest.py \
  --dataset ai4bharat/MSMARCO-XI \
  --config hi \
  --split train \
  --backend bge-m3 \
  --vector-backend qdrant \
  --qdrant-path storage/qdrant
```

If the dataset is delivered as an export instead:

```bash
python scripts/ingest.py \
  --input path/to/MSMARCO-XI.jsonl \
  --backend bge-m3 \
  --vector-backend local \
  --out-index storage/msmarco_xi_index.json
```

The ingestion adapter accepts JSONL, JSON, CSV, and common passage fields (`text`, `passage`, `content`, `document`, `context`, `body`, `passages`, `positive_passages`). It writes a manifest with document count, chunk count, strategies, embedding dimension, and backend.

## Chunking and indexing design

Indexing creates multiple retrieval views for every parent document:

1. **Paragraph chunks** preserve natural document boundaries.
2. **Sentence windows** use three-sentence windows with one-sentence overlap.
3. **Token windows** are a fallback for long/noisy passages; default 180 tokens with 36-token overlap.
4. **Semantic chunks** split when consecutive sentence embeddings cross a distance threshold.
5. **Metadata-aware chunks** prefix title, source, topic, language, and other fields to improve filtering and recall.
6. **Parent-child linkage** is retained through `parent_id`; small child chunks retrieve while citations expose the parent relationship.

Dense retrieval and BM25 run in parallel conceptually and are fused with reciprocal-rank fusion. A lightweight rerank score combines dense similarity, lexical relevance, query-token overlap, and metadata-aware chunk views.

## Harness and guardrails

Every request returns a structured `PipelineResponse` containing:

- request ID;
- transcript;
- answer or refusal;
- citations with chunk ID, parent ID, strategy, score, text, and metadata;
- per-stage latency and total latency;
- retry attempts;
- structured error/refusal reason.

The harness retries transient step failures, enforces a per-step timeout, and falls back to the deterministic grounded generator if the remote generator fails.

Guardrail examples:

```json
{
  "status": "abstained",
  "answer": "I don't have enough evidence in the indexed dataset to answer that.",
  "refusal_reason": "insufficient_context",
  "citations": []
}
```

Unsafe instructions are blocked before retrieval. Answers without `[S1]`-style citations or with insufficient token overlap against retrieved context are refused. This is a deterministic baseline guardrail; for the final evaluation, add a calibrated NLI/entailment verifier and report its false-positive/false-negative results.

## Benchmarking

Run the required P50/P70/P100 benchmark after indexing:

```bash
python scripts/benchmark.py --count 200 --warmup 10
```

Use real representative queries for the submission:

```bash
python scripts/benchmark.py --queries eval_queries.jsonl --count 500 --warmup 25 \
  --output benchmarks/msmarco_iix.json
```

The script records raw rows and reports:

- total text-path latency;
- retrieval latency;
- P50, P70, P100;
- query count and statuses;
- answered/abstained/error counts.

**Do not reuse the sample values as final claims.** A local run on the included eight-document demo corpus produced 200/200 answered queries with:

```text
text path total: P50 3.355 ms · P70 3.514 ms · P100 4.287 ms
retrieval:        P50 2.633 ms · P70 2.713 ms · P100 3.582 ms
```

Those numbers are real local measurements from the deterministic demo path, not MSMARCO-XI or hosted Sarvam/LLM measurements. Re-run on the final corpus, hardware, concurrency policy, and deployed models. Report external STT network latency separately and transparently.

With OpenCode Go GPT 5.6 Luna enabled, a real end-to-end text smoke query returned a grounded answer with citations in 2,482.4 ms in the latest run. That was a single smoke test, not a benchmark, and it demonstrates why the submission must report remote generation latency honestly rather than claim the 200 ms target for an external model call.

## API

- `GET /health`
- `GET /api/config`
- `POST /api/query` with `{ "query": "...", "language_code": "unknown" }`
- `POST /api/voice` multipart form with `audio` and optional `language_code`
- `GET /` browser demo

Example:

```bash
curl -X POST http://127.0.0.1:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"query":"What is the capital of Goa?"}'
```

## Deployment

For the student-friendly Vercel + Python backend deployment plan, see [`DEPLOYMENT.md`](DEPLOYMENT.md). Public deployment security details are in [`SECURITY.md`](SECURITY.md). Vercel should host the static frontend; Render/Railway/Hugging Face Spaces should host FastAPI, with Qdrant Cloud or a persistent Qdrant instance for the final index.

The default Docker image runs with local hashing embeddings and the deterministic generator. For the final system, build the index during deployment or mount the pre-built index/Qdrant storage.

```bash
docker build -t vaani-rag .
docker run --rm -p 8000:8000 --env-file .env vaani-rag
```

For a serious hosted benchmark, pin model versions, keep the vector index warm, run a concurrency test separately, and record CPU/RAM/GPU, provider region, model IDs, and all timeout/retry counts.

## Submission checklist from the organizer document

- [ ] Final GitHub repository link
- [ ] Live working link
- [ ] Form completed once, only after all links are final
- [ ] 90-second team/process video (process, not product demo)
- [ ] Separate end-to-end demo video
- [ ] Both videos uploaded by every team member to Instagram, X, and LinkedIn
- [ ] At least one Instagram account is public
- [ ] Required campaign text/hashtag included on every post
- [ ] Final P50/P70/P100 results measured on the actual final build
- [ ] Task deadline: **August 22, 2026, 11:59 PM**

## License and data

Check and document the license/terms for the supplied dataset, Sarvam, embedding model, vector database, and generation provider in the final repository. Do not publish restricted dataset content or API keys.
