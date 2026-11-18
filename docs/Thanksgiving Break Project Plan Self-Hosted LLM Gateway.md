# Thanksgiving Break Project Plan: Self-Hosted LLM Gateway

A project idea that fits NVIDIA's tooling and Meta's open models, with a role for each of the three teammates.

## The idea

Your three roles map onto one theme: **building the production infrastructure around open models**. NVIDIA's strength is the tooling for serving and optimizing models, and Meta's is the open models themselves (Llama) and the PyTorch ecosystem. A project where those meet gives the backend engineer a real backend problem, gives the AI engineer real model work, and gives the fullstack freelancer a clear frontend task.

## Best fit: a self-hosted LLM gateway with benchmarking

Run an open Llama model behind a serving engine, then build the platform around it.

### Backend engineer (you)

An API gateway that handles auth and API keys, rate limiting, request queueing, response caching, and per-user usage metering. Go is a good fit for this, with MongoDB for keys and usage logs and Prometheus-style metrics for latency and throughput. This is the kind of work backend roles care about: concurrency, reliability, and observability.

### AI engineer (friend)

Picks and serves the models (vLLM, or NVIDIA's Triton / TensorRT-LLM if you have a GPU), compares quantization levels, and benchmarks quality against speed. They can also add guardrails with NVIDIA's NeMo Guardrails or build a small eval set to compare models.

### Fullstack engineer (freelance friend)

A dashboard showing usage, latency charts, and a playground for trying models. It's a well-scoped, self-contained task he can do independently.

### Why it works for both companies

Meta's open models running on NVIDIA's serving stack, with your engineering around them.

## Alternative: domain-specific RAG service

Build a question-answering system over a corpus you care about (course catalogs, docs, a game's frame data). The backend engineer builds the ingestion pipeline, vector search (Meta's FAISS or a vector DB), caching, and the API. The AI engineer handles embeddings, chunking, reranking, and measuring answer quality. NVIDIA publishes open embedding and retrieval models you can use. It's easier to get working in a week, but the backend work is less deep than the gateway.

## Keeping it realistic for one break

- **GPU access:** If nobody has an NVIDIA GPU, NVIDIA's hosted API catalog (build.nvidia.com) offers free OpenAI-compatible endpoints for experimenting, and small quantized Llama models run on CPU through Ollama or llama.cpp. Build the gateway against a swappable backend so you can plug in real GPU serving later.
- **Scope for the week:** Aim for a working gateway with auth, rate limiting, and logging, one served model, a benchmark script, and a basic dashboard. Caching, queueing, and multi-model routing are stretch goals.
- **Interfaces first:** Agree on the API contract on day one (an OpenAI-compatible `/v1/chat/completions` is a good standard) so all three of you can work in parallel without blocking each other.
- **Verify the tooling:** These projects change quickly, so check each one's current docs before committing to a stack.

## Open question

Does anyone on the team have access to an NVIDIA GPU? That decides whether you serve models locally with Triton/vLLM or lean on hosted endpoints.

## Week scope

**Goal:** a demo-able gateway by the end of the week. A client sends an OpenAI-style request with an API key. The gateway checks the key and rate limit, runs input guardrails, forwards to a locally served Llama model, runs output guardrails, logs usage, and a dashboard shows all of it.

### Assumptions

- The model is served locally with vLLM on the team's NVIDIA GPU. Model size depends on VRAM, which is still to confirm.
- The gateway is Go with Gin, MongoDB for storage, and everything starts from one `docker-compose.yml`.
- Guardrails run as a small Python service wrapping NeMo Guardrails. The gateway depends only on our own contract (below), not on NeMo's API, which still changes between releases.
- Output guardrails apply to non-streaming requests only in v1. Input guardrails apply to all requests. Checking streamed output needs buffering, which is a stretch goal.

### Scope

**Must have**

- API key auth: create, list, revoke
- `POST /v1/chat/completions`, streaming and non-streaming
- `GET /v1/models`
- Per-key rate limiting
- Usage logging: tokens, latency, status
- Input and output guardrails, with every block logged as an event
- Dashboard: usage, latency, guardrail events, key management
- Benchmark script: latency and throughput, plus pass rate on a small safety test set

**Stretch**

- Response caching
- Request queue
- Multi-model routing
- Prometheus `/metrics` endpoint
- Playground page in the dashboard
- Output guardrails on streamed responses

**Out of scope**

- User signup and accounts, billing
- Multi-instance deployment, HTTPS, a public domain
- Fine-tuning

### Ownership

| Person | Owns |
| --- | --- |
| Backend (you) | Gateway, auth, rate limiting, usage logging, admin API, MongoDB, docker-compose |
| AI engineer | vLLM serving, guardrails service, safety test set, benchmark script |
| Fullstack engineer | Dashboard, built against a mock server from day 1 |

### Daily plan

| Day | Backend | AI engineer | Fullstack |
| --- | --- | --- | --- |
| 1 | Agree on the contract, repo, docker-compose skeleton, mock server from the contract | Get vLLM serving one model on its OpenAI-compatible endpoint | Dashboard shell against the mock server |
| 2 | API key auth and non-streaming proxy | Baseline guardrails config, `/check` endpoint returning allow for everything | Keys page and usage chart on mock data |
| 3 | Streaming proxy, per-key rate limiting, usage logging | Input rails: jailbreak and topic checks | Latency and error views |
| 4 | Call guardrails before and after the model, log guardrail events | Output rails, build the safety test set | Guardrail events page |
| 5 | Finish admin endpoints, switch the dashboard to the real API | Benchmark script: latency, throughput, safety pass rate | Wire dashboard to the real API |
| 6 | Integration testing, load test, bug fixes, README | Tune rails against the test set | Polish, empty and error states |
| 7 | Buffer, demo run-through, stretch goals if on track | Write up benchmark results | Buffer |

### Ground rules for AI coding assistants

- Give the assistant the contract section first so generated code matches it.
- Read every diff before merging. You own what you merge, so ask the assistant to explain anything you couldn't explain yourself.
- Have it write tests with the code, such as table-driven Go tests for auth and rate limiting.
- One feature per branch and small commits, so a bad generation is easy to revert.
- Never paste real keys or secrets into prompts. Put `.env` in `.gitignore` in the first commit.
- Check suggested libraries and versions against the real docs. Generated code sometimes uses outdated or nonexistent APIs, especially for fast-moving tools like vLLM and NeMo Guardrails.

### Definition of done

- [ ] `docker compose up` starts the gateway, MongoDB, model server, and guardrails service
- [ ] A request with a valid key returns a model response; an invalid key returns 401
- [ ] Exceeding the rate limit returns 429 with `Retry-After`
- [ ] A jailbreak-style prompt is blocked and shows up in the dashboard
- [ ] The dashboard shows real usage and latency
- [ ] The benchmark script runs end to end and prints results
- [ ] README explains setup in under 10 steps

## API contract (v0.1)

Freeze this on day 1. After that, any change needs all three of you to agree. The fullstack engineer builds against a mock server generated from this section.

### Conventions

- Base URL: `http://localhost:8080`
- Public endpoints: `Authorization: Bearer gw_live_<random>`
- Admin endpoints: `Authorization: Bearer <ADMIN_TOKEN>`, read from an environment variable
- JSON bodies, timestamps in ISO 8601 UTC
- Every response includes an `X-Request-Id` header

### Public endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/v1/chat/completions` | OpenAI-compatible chat completion, streaming or not |
| GET | `/v1/models` | Lists models the key may use |
| GET | `/healthz` | Health check, no auth |

**Request**

```json
{
  "model": "llama-3.1-8b-instruct",
  "messages": [{"role": "user", "content": "Hello"}],
  "temperature": 0.7,
  "max_tokens": 256,
  "stream": false
}
```

**Response** (same shape as OpenAI)

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1767225600,
  "model": "llama-3.1-8b-instruct",
  "choices": [{"index": 0, "message": {"role": "assistant", "content": "Hi!"}, "finish_reason": "stop"}],
  "usage": {"prompt_tokens": 9, "completion_tokens": 2, "total_tokens": 11}
}
```

With `"stream": true` the response is server-sent events: one `data: {chunk}` line per chunk in OpenAI's chunk format, ending with `data: [DONE]`.

**Extra response headers:** `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-Guardrail-Action` (`allow` or `block`).

The model name above is a placeholder. Use whatever the AI engineer actually serves; `GET /v1/models` is the source of truth.

### Errors

All errors use the OpenAI-style envelope:

```json
{"error": {"message": "Rate limit exceeded", "type": "rate_limit_error", "code": "rate_limit_exceeded"}}
```

| Status | `code` | When |
| --- | --- | --- |
| 400 | `invalid_request` | Malformed body or unknown model |
| 400 | `guardrail_blocked` | An input or output rail blocked the request; the error also has a `guardrail` field naming the rail |
| 401 | `invalid_api_key` | Missing, unknown, or revoked key |
| 429 | `rate_limit_exceeded` | Over the key's limit; includes `Retry-After` in seconds |
| 502 | `upstream_error` | Model server or guardrails service returned an error |
| 503 | `model_unavailable` | Model server unreachable |
| 504 | `upstream_timeout` | Model server too slow |

### Admin endpoints (used by the dashboard)

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/admin/keys` | Create a key; the full key is returned only in this response |
| GET | `/admin/keys` | List keys (prefix only, never the full key) |
| DELETE | `/admin/keys/{id}` | Revoke a key |
| GET | `/admin/usage` | Usage over time |
| GET | `/admin/summary` | Headline numbers for a time range |
| GET | `/admin/guardrail-events` | Recent blocked or modified requests |

**POST /admin/keys**

```json
// request
{"name": "hackathon-demo", "rate_limit_rpm": 60}
// response 201
{"id": "key_01", "name": "hackathon-demo", "key": "gw_live_...", "rate_limit_rpm": 60, "created_at": "2026-11-23T10:00:00Z"}
```

**GET /admin/usage?from=...&to=...&key\_id=...&bucket=hour|day**

```json
{"buckets": [{"start": "2026-11-23T10:00:00Z", "requests": 42, "prompt_tokens": 1800, "completion_tokens": 5200, "errors": 1, "blocked": 3, "p50_latency_ms": 820, "p95_latency_ms": 2100}]}
```

**GET /admin/summary?from=...&to=...**

```json
{"requests": 1280, "error_rate": 0.012, "blocked_rate": 0.04, "total_tokens": 310000, "p50_latency_ms": 800, "p95_latency_ms": 2300}
```

**GET /admin/guardrail-events?limit=50&cursor=...**

```json
{"events": [{"id": "evt_01", "time": "2026-11-23T10:05:00Z", "key_id": "key_01", "stage": "input", "rail": "jailbreak", "action": "block", "reason": "Prompt matched jailbreak pattern"}], "next_cursor": null}
```

### Guardrails service contract (internal)

The AI engineer's Python service exposes one endpoint. The gateway calls it before the model (`input`) and after it (`output`).

```json
// POST /check request
{"stage": "input", "messages": [{"role": "user", "content": "..."}]}
// for stage "output": {"stage": "output", "messages": [...], "response": "..."}

// response
{"action": "allow", "rail": null, "reason": null, "text": null}
```

`action` is `allow`, `block`, or `modify`. On `modify`, the gateway uses `text` instead of the original content (for example, PII masked). If the service is unreachable, the gateway fails closed with a 502 for now; decide as a team whether to change that.

### MongoDB collections

- `api_keys`: `id`, `name`, `key_hash`, `prefix`, `rate_limit_rpm`, `created_at`, `revoked_at`. Store only a hash of the key.
- `usage_logs`: `request_id`, `key_id`, `model`, `status`, `prompt_tokens`, `completion_tokens`, `latency_ms`, `stream`, `created_at`
- `guardrail_events`: `id`, `request_id`, `key_id`, `stage`, `rail`, `action`, `reason`, `created_at`
