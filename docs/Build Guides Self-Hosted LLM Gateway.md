# Build Guides: Self-Hosted LLM Gateway

Step-by-step guides for every day of the week plan. Pick your role, follow the days in order, and run the check at the end of each step before moving on.

## How to use these guides

- Every step has a goal, the steps themselves, a check that proves it works, and a starter prompt for your AI assistant.
- Ask your assistant to explain before it writes code, and read every diff before you commit.
- These tools change quickly. When a guide names a flag, package, or config format, confirm it against the tool's current docs.
- Stuck for more than 45 minutes? Jump to **When you're stuck** at the end.

## Shared setup (everyone, Day 1)

### Repo layout

```
llm-gateway/
  gateway/        Go gateway and admin API
  guardrails/     Python /check service
  dashboard/      React dashboard
  bench/          benchmark script and safety test set
  docker-compose.yml
  .env.example
  README.md
```

### Ports and environment

| Service | Port | Owner |
| --- | --- | --- |
| Gateway | 8080 | Backend |
| Model server (vLLM) | 8000 | AI engineer |
| Model server (Ollama fallback) | 11434 | AI engineer |
| Guardrails service | 8001 | AI engineer |
| MongoDB | 27017 | Backend |
| Dashboard (dev server) | 5173 | Fullstack |

Environment variables: `ADMIN_TOKEN`, `MONGO_URL`, `UPSTREAM_URL`, `GUARDRAILS_URL`, `HF_TOKEN`. Commit an `.env.example` with placeholder values, and keep the real `.env` in `.gitignore` from the first commit.

### Git workflow

- `main` always works. One branch per feature, small commits.
- Open a pull request for each feature, even if you review it yourself the next morning.
- Never commit keys, tokens, or `.env`.

### Contract check

Before writing code, all three of you read the API contract section of the project plan and confirm it. The fullstack engineer builds against it with mocks, and the backend builds the real thing to match it.

## Backend guide (Go gateway)

### Day 1: Skeleton, config, and MongoDB

**Goal:** `docker compose up` starts MongoDB and the gateway, and `GET /healthz` returns 200.

1. In `gateway/`, run `go mod init` and add Gin and the official MongoDB Go driver.
2. Write `main.go` so it loads config from environment variables and starts Gin on port 8080.
3. Add `GET /healthz` returning a small JSON status.
4. Write a multi-stage Dockerfile: build with the Go image, run the binary in a small image.
5. Write `docker-compose.yml` with a `mongo` service and a `gateway` service that depends on it.
6. Connect to MongoDB on startup and ping it. Fail fast with a clear log line if it can't connect.

**Check:** run `docker compose up --build`, then `curl localhost:8080/healthz`. Inside Docker, the gateway reaches MongoDB at `mongo:27017`, not `localhost`.

**Starter prompt:** I'm building a Go gateway with Gin and MongoDB. Explain a good folder structure for a small service, then scaffold `main.go` with env-based config and a `/healthz` endpoint. Explain each file as you go.

### Day 2: API key auth and a non-streaming proxy

**Goal:** keys can be created and revoked, and a valid key can send a chat request and get a model answer back.

**Part A: keys**

1. Generate keys with `crypto/rand` (32 random bytes, encoded) and give them the prefix `gw_live_`.
2. Store only the SHA-256 hash and a short prefix in the `api_keys` collection. Show the full key once, at creation. SHA-256 is fine here because the keys are long and random, unlike passwords.
3. Build `POST /admin/keys`, `GET /admin/keys`, and `DELETE /admin/keys/{id}`, protected by a middleware that checks `ADMIN_TOKEN`.
4. Build an auth middleware for `/v1/*`. It reads the Bearer token, hashes it, looks it up, and rejects missing or revoked keys with a 401 `invalid_api_key` in the contract's error format. Store the key's id in the Gin context for later steps.

**Part B: proxy**

5. Build `POST /v1/chat/completions`. Parse the body, check that `model` and `messages` exist, then forward the JSON to `UPSTREAM_URL` plus `/v1/chat/completions` using Go's HTTP client with a timeout (60 seconds is a reasonable start).
6. Return the upstream's status and body to the caller. Pass the request's `context` to the upstream call so a cancelled request cancels the upstream call too.
7. Map failures to contract errors: connection refused becomes 503, timeout becomes 504, upstream 5xx becomes 502.
8. Add `GET /v1/models` that forwards to the upstream.

Until the AI engineer has vLLM running, point `UPSTREAM_URL` at Ollama (`http://localhost:11434`) or a hosted OpenAI-compatible endpoint.

**Check:** create a key through the admin endpoint, then call the gateway with it and get a reply. A wrong key returns 401. Stopping the model server returns 503.

**Starter prompt:** Here is my API contract section. Write the Gin middleware that validates a Bearer API key against SHA-256 hashes stored in MongoDB. First explain how the hash lookup works and why we don't store raw keys. Then write table-driven tests, then the implementation.

### Day 3: Streaming, rate limiting, and usage logging

**Goal:** streamed responses arrive chunk by chunk, keys are rate limited, and every request is logged.

**Streaming**

1. When the request has `stream` set to true, call the upstream and set the response headers `Content-Type: text/event-stream` and `Cache-Control: no-cache`.
2. Read the upstream body line by line. Write each line plus a newline to the client and flush after every event (a blank line ends an event). In Go this means using `http.Flusher` on the response writer.
3. Stop when you see `data: [DONE]` or when the client disconnects (the request context is done).
4. For token counts while streaming, check whether the upstream can include usage in the final chunk (OpenAI-style APIs offer a `stream_options` setting for this). Confirm in your upstream's docs. If it can't, log the token fields as zero for now.

**Check:** `curl -N` with `stream: true` shows text appearing gradually. If everything arrives at once, you are missing a flush or something is buffering.

**Rate limiting**

5. Keep a map from key id to a token bucket, protected by a mutex. The `golang.org/x/time/rate` package gives you this, with the rate set to the key's requests per minute divided by 60 and a small burst. Write a tiny version by hand first so you understand how a token bucket works.
6. When a key is over its limit, return 429 with a `Retry-After` header and the `rate_limit_exceeded` error. Set `X-RateLimit-Limit` and `X-RateLimit-Remaining` on every response.

**Usage logging**

7. Add a middleware that creates a request id and returns it as `X-Request-Id`.
8. After each request, insert a document into `usage_logs` with the fields from the contract. Send it through a buffered channel to a background writer so logging never slows the response.

**Check:** a loop of 100 quick `curl` calls hits 429, and MongoDB contains one log document per request. The limiter has a unit test.

### Day 4: Guardrail integration

**Goal:** the gateway calls the guardrails service before and after the model.

1. Define Go structs for the `/check` request and response from the contract.
2. Write a small client for `GUARDRAILS_URL` with a short timeout (about 5 seconds).
3. **Input stage:** before calling the model, send the messages with stage `input`. If the answer is `block`, return 400 `guardrail_blocked` with the rail name and set `X-Guardrail-Action: block`. If it is `modify`, replace the last user message's content with the returned `text`.
4. **Output stage (non-streaming only):** after the model replies, send the assistant text with stage `output`. Block or replace it the same way.
5. Insert a document into `guardrail_events` for every decision that isn't `allow`, and record the action on the usage log too.
6. If the guardrails service is unreachable, return 502 `upstream_error` (the gateway fails closed).

If the AI engineer's service isn't ready, write a throwaway stub that blocks any input containing a phrase like `ignore previous instructions` and allows everything else. The gateway work doesn't need the real service.

**Check:** send a normal prompt and get a reply. Send a blocked prompt and get a 400. Confirm the event shows up in MongoDB.

### Day 5: Admin endpoints and dashboard wiring

**Goal:** `/admin/usage`, `/admin/summary`, and `/admin/guardrail-events` return exactly the shapes in the contract.

1. **Usage:** use a MongoDB aggregation that filters by time range and optional key, then groups by hour or day with `$dateTrunc`. Sum requests, tokens, errors (status 400 or above), and blocked requests.
2. **Percentiles:** at this scale, fetch the latencies for each bucket, sort them in Go, and read off p50 and p95. It is simpler than doing it in the database.
3. **Summary:** the same calculation over the whole range, as one object.
4. **Guardrail events:** newest first, with a `limit` and a `cursor` based on the last document's id.
5. Add CORS middleware allowing `http://localhost:5173`.
6. Write a seed script that inserts fake usage logs and events so charts have data to show.

**Check:** call each endpoint with `curl` and compare field names and types to the contract examples one by one. Tell the fullstack engineer the real API is ready so he can turn his mocks off.

### Day 6: Integration, load test, and README

1. Clone the repo fresh, copy `.env.example` to `.env`, and run `docker compose up --build`. Fix everything that breaks.
2. Load test with a tool like `hey` or `k6` (for example 200 requests at concurrency 10). Watch for errors, memory growth, and slow MongoDB writes.
3. Run failure drills: stop the model server, stop the guardrails service, send malformed JSON, use a revoked key, and send a very large body. Set a maximum body size.
4. Write the README: prerequisites, setup, the one command to start everything, example `curl` calls, a short architecture sketch, and known limits (in-memory rate limiting on a single instance, no output rails on streams).

**Check:** someone who didn't write the code gets it running from the README alone.

### Day 7: Buffer and demo

- Rehearse the demo: create a key, send a normal request, send a streamed request, send a blocked request, hit the rate limit, then show all of it in the dashboard.
- Only if everything above is solid, pick one stretch goal. Response caching for requests with temperature 0 (keyed on a hash of the model, messages, and parameters) is the most self-contained.
- Write down what you learned. It is the best interview material this project gives you.

## AI engineer guide (model serving and guardrails)

### Day 1: Serve one model

**Goal:** a model answering on an OpenAI-compatible endpoint, running on the team's NVIDIA GPU.

1. Run `nvidia-smi` and write down the GPU model, total VRAM, and driver version. Tell the team.
2. Pick a model size that fits. As a rough rule, an 8-billion-parameter model at 16-bit precision needs around 16 GB for the weights alone, plus room for context. Quantized versions (8-bit or 4-bit) need much less. Leave headroom, because the safety model will need VRAM too. Check the model card for real numbers.
3. For Llama models, accept the license on the model's Hugging Face page, create an access token, and set it as `HF_TOKEN`. Read the license terms while you are there.
4. Start vLLM with its Docker image, giving it GPU access, port 8000, your token, and the model name. If it runs out of memory, lower the maximum context length or the GPU memory fraction it is allowed to use. Check the current vLLM docs for the exact flag names.
5. Test it with `curl localhost:8000/v1/models`, then send a chat completion request.
6. Add the model server as a service in `docker-compose.yml`.

**Fallback:** if vLLM won't start after an hour or two, install Ollama, pull a small model, and use its OpenAI-compatible endpoint on port 11434. The gateway doesn't care which one is behind it.

**Check:** a chat request returns a sensible answer. Send the exact model name to the backend engineer, because it goes in `/v1/models` and in every request.

**Starter prompt:** I have an NVIDIA GPU with this much VRAM: (paste `nvidia-smi` output). Which Llama model sizes and quantizations will fit with room left for a small safety model? Explain the memory math, then give me the vLLM Docker command.

### Day 2: The /check service skeleton

**Goal:** a running Python service that implements the guardrails contract and allows everything.

1. In `guardrails/`, create a virtual environment and install FastAPI and uvicorn.
2. Define request and response models that match the contract, then implement `POST /check` so it always returns `allow`.
3. Add a Dockerfile and a compose service on port 8001.
4. Add a first, cheap rail: a list of phrases and patterns (such as attempts to override instructions) that returns `block` with a rail name and reason when matched. This is your fallback if the heavier tools give you trouble later.
5. Tell the backend engineer the service is up and send him two example `curl` commands, one that allows and one that blocks.

**Check:** both example commands return what you promised.

### Day 3: Input rails with NeMo Guardrails

**Goal:** `/check` with stage `input` runs a real NeMo Guardrails input rail after the keyword rail.

1. Install a pinned version of the `nemoguardrails` package. As of early November 2026 the latest listed version was 0.24.1. Confirm that and skim the release notes first, because the library is pre-1.0 and its API changes between releases.
2. Create a config folder with a `config.yml` and point its model settings at your local model server's OpenAI-compatible endpoint. The NeMo docs show how to configure a custom or OpenAI-compatible engine.
3. Turn on the built-in input check, which asks a model to judge each message against a written policy. It needs a prompt file holding that policy. Copy the structure from NeMo's getting-started guide instead of guessing the YAML.
4. Load the config in Python and run the messages through it from the `/check` handler. Translate the result into `allow` or `block`, with the rail name and a short reason.
5. Keep the order: keyword rail first (fast and free), NeMo second.

The self-check approach makes an extra model call for every request, so it adds latency. Measure it, because that number belongs in your benchmark results.

**If you are stuck for more than two hours:** call your local model directly with a short prompt that asks whether a message violates a written policy and requires a one-word answer. That is essentially what the self-check rail does, and it is enough for this project.

**Check:** run ten prompts (five harmless, five attacks) through `/check` and confirm the decisions look right.

### Day 4: Output rails and the safety test set

**Goal:** responses are checked too, and you can measure how well the rails work.

1. **Output rail:** for stage `output`, check the response against the policy, and add simple patterns for personal data such as email addresses and phone numbers. When something matches, return `modify` with the sensitive part masked.
2. **Test set:** create `bench/safety_set.jsonl` with at least 50 rows. Each row has a prompt, an expected result (`allow` or `block`), and a category. Include harmless prompts (including tricky but legitimate ones, like defensive security questions), instruction-override attempts, role-play jailbreaks, prompt-injection text, and requests for personal data. Keep the harmful prompts short and non-operational. They only need to be things your policy should refuse.
3. **Runner:** write `bench/run_safety.py`. It sends every prompt to the gateway, compares the outcome to the expected result, and prints the percentage of attacks blocked, the percentage of harmless prompts wrongly blocked, and a small table by category.

An AI assistant can help draft test prompts, but read all of them yourself.

**Check:** the runner prints numbers, and you can explain why each miss happened.

### Day 5: Benchmark script

**Goal:** repeatable numbers for speed.

1. Write `bench/run_bench.py` using `httpx` with async requests at several concurrency levels, such as 1, 4, and 16.
2. For streamed requests, record time to first token and total time. Compute tokens per second from the usage numbers.
3. Report p50 and p95 for each concurrency level.
4. Run the same test through the gateway and directly against the model server. The difference is the gateway's overhead.
5. If your GPU has room, compare two quantization levels of the same model on speed, on safety-set results, and on a small set of about 20 questions with known answers.
6. Write a markdown table to `bench/results.md`.

**Check:** two runs of the script give similar numbers.

### Day 6: Tune the rails

- Read the false positives and false negatives from the safety runner. Adjust the policy text or the patterns, then run the set again.
- Record before and after numbers, and the latency each rail adds.
- Agree with the backend engineer on the guardrails timeout.

### Day 7: Write-up

Write one page covering the setup, the models you tried, your benchmark numbers, how well the rails worked, known limits, and what you would do next. This page is the part to show in interviews.

## Fullstack guide (dashboard)

### Day 1: Shell with mocks

**Goal:** a dashboard shell that loads fake data without needing the backend.

1. In `dashboard/`, create a Vite React project. Add Tailwind, React Router, Recharts, and MSW (Mock Service Worker).
2. Read the API contract and copy its example responses into MSW handlers for every `/admin` endpoint.
3. Build the layout: a sidebar (Overview, Keys, Usage, Guardrails) and a header.
4. Write one API client module that reads the base URL from `VITE_API_URL` and sends the admin token. Ask for the token on a simple entry screen. Don't hard-code it.

**Check:** `npm run dev` shows the shell, and each page can fetch mock data.

**Starter prompt:** Here is my API contract. Set up MSW handlers that return its example responses, and an API client that attaches a Bearer token. Explain how MSW intercepts requests so I understand it.

### Day 2: Keys page and usage chart

1. **Keys page:** a table of keys (name, prefix, rate limit, created, revoked), a create form for name and requests per minute, and a revoke button with a confirmation.
2. After creating a key, show the full key once in a dialog with a copy button and a warning that it can't be shown again.
3. **Usage chart:** requests over time as a line chart and tokens as a bar chart, from `/admin/usage`.
4. Handle loading, error, and empty states on every page from the start.

**Check:** creating, copying, and revoking a key all work against the mocks.

### Day 3: Overview, latency, and errors

1. Overview cards from `/admin/summary`: requests, error rate, blocked rate, p50 and p95 latency.
2. A latency chart with p50 and p95 lines over time.
3. A time range picker (last hour, 24 hours, 7 days) that sets `from` and `to` and chooses hourly or daily buckets.

**Check:** changing the range updates every chart and card.

### Day 4: Guardrail events page

1. A table with time, key, stage, rail, action, and reason.
2. Filters for stage and action.
3. A "load more" button that uses the cursor from the last response.
4. Colored badges for `block` and `modify`.

**Check:** paging works against a mock that returns several pages.

### Day 5: Connect to the real API

1. Add an environment flag that turns MSW off.
2. Point `VITE_API_URL` at the gateway and fix whatever differs from the mocks: field names, null values, error shapes.
3. If the browser blocks requests with a CORS error, tell the backend engineer. The fix is on the gateway side.
4. Show error messages from the API's `error.code`, not generic failures.

**Check:** every page works with real data from the seed script.

### Day 6: Polish

- Responsive layout, page titles, consistent empty and error states.
- Stretch: a playground page that sends a chat request with a chosen key and shows a streamed reply. Streaming in the browser means reading the response body with `fetch` and a stream reader.
- Stretch: a Dockerfile that serves the built dashboard so it can join the compose file.

### Day 7: Buffer

Take screenshots for the README and help rehearse the demo.

## When you're stuck

**The 45-minute rule:** after 45 minutes without progress, build the smallest example that still fails. Give your assistant the exact error, the relevant code, and the versions you use. If that doesn't solve it, ask a teammate. If you are blocked on a teammate, use a stub and keep moving.

**Handy commands:** `docker compose logs -f <service>`, `curl -v`, `mongosh`, `nvidia-smi`.

| Problem | Likely cause |
| --- | --- |
| Containers can't reach each other | Use the compose service name (like `mongo`), not `localhost` |
| vLLM runs out of memory | Smaller model or quantization, shorter max context, lower GPU memory fraction |
| Streamed text arrives all at once | Missing flush in the gateway, or something between you and the client is buffering |
| Browser shows a CORS error | The gateway is missing CORS headers for the dashboard's origin |
| Every request returns 401 | Key was copied wrong, or the hash stored doesn't match how it is looked up |
| Guardrails slow every request | The self-check rail adds a model call; use the keyword rail first and measure |
