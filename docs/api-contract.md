### Conventions

- Base URL: `http://localhost:8080`
- Public endpoints: `Authorization: Bearer gw_live_<random>`
- Admin endpoints: `Authorization: Bearer <ADMIN_TOKEN>`, read from an environment variable
- JSON bodies, timestamps in ISO 8601 UTC
- Every response includes an `X-Request-Id` header

### Public endpoints

| Method | Path                   | Purpose                                             |
| ------ | ---------------------- | --------------------------------------------------- |
| POST   | `/v1/chat/completions` | OpenAI-compatible chat completion, streaming or not |
| GET    | `/v1/models`           | Lists models the key may use                        |
| GET    | `/healthz`             | Health check, no auth                               |

**Request**

```json
{
  "model": "llama-3.1-8b-instruct",
  "messages": [{ "role": "user", "content": "Hello" }],
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
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hi!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 9, "completion_tokens": 2, "total_tokens": 11 }
}
```

With `"stream": true` the response is server-sent events: one `data: {chunk}` line per chunk in OpenAI's chunk format, ending with `data: [DONE]`.

**Extra response headers:** `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-Guardrail-Action` (`allow` or `block`).

The model name above is a placeholder. Use whatever the AI engineer actually serves; `GET /v1/models` is the source of truth.

### Errors

All errors use the OpenAI-style envelope:

```json
{
  "error": {
    "message": "Rate limit exceeded",
    "type": "rate_limit_error",
    "code": "rate_limit_exceeded"
  }
}
```

| Status | `code`                | When                                                                                                |
| ------ | --------------------- | --------------------------------------------------------------------------------------------------- |
| 400    | `invalid_request`     | Malformed body or unknown model                                                                     |
| 400    | `guardrail_blocked`   | An input or output rail blocked the request; the error also has a `guardrail` field naming the rail |
| 401    | `invalid_api_key`     | Missing, unknown, or revoked key                                                                    |
| 429    | `rate_limit_exceeded` | Over the key's limit; includes `Retry-After` in seconds                                             |
| 502    | `upstream_error`      | Model server or guardrails service returned an error                                                |
| 503    | `model_unavailable`   | Model server unreachable                                                                            |
| 504    | `upstream_timeout`    | Model server too slow                                                                               |

### Admin endpoints (used by the dashboard)

| Method | Path                      | Purpose                                                      |
| ------ | ------------------------- | ------------------------------------------------------------ |
| POST   | `/admin/keys`             | Create a key; the full key is returned only in this response |
| GET    | `/admin/keys`             | List keys (prefix only, never the full key)                  |
| DELETE | `/admin/keys/{id}`        | Revoke a key                                                 |
| GET    | `/admin/usage`            | Usage over time                                              |
| GET    | `/admin/summary`          | Headline numbers for a time range                            |
| GET    | `/admin/guardrail-events` | Recent blocked or modified requests                          |

**POST /admin/keys**

```json
// request
{"name": "hackathon-demo", "rate_limit_rpm": 60}
// response 201
{"id": "key_01", "name": "hackathon-demo", "key": "gw_live_...", "rate_limit_rpm": 60, "created_at": "2026-11-23T10:00:00Z"}
```

**GET /admin/usage?from=...&to=...&key_id=...&bucket=hour|day**

```json
{
  "buckets": [
    {
      "start": "2026-11-23T10:00:00Z",
      "requests": 42,
      "prompt_tokens": 1800,
      "completion_tokens": 5200,
      "errors": 1,
      "blocked": 3,
      "p50_latency_ms": 820,
      "p95_latency_ms": 2100
    }
  ]
}
```

**GET /admin/summary?from=...&to=...**

```json
{
  "requests": 1280,
  "error_rate": 0.012,
  "blocked_rate": 0.04,
  "total_tokens": 310000,
  "p50_latency_ms": 800,
  "p95_latency_ms": 2300
}
```

**GET /admin/guardrail-events?limit=50&cursor=...**

```json
{
  "events": [
    {
      "id": "evt_01",
      "time": "2026-11-23T10:05:00Z",
      "key_id": "key_01",
      "stage": "input",
      "rail": "jailbreak",
      "action": "block",
      "reason": "Prompt matched jailbreak pattern"
    }
  ],
  "next_cursor": null
}
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
