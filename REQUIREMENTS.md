# Herald: webhook delivery service

Rename it if you want. This is the one backend project, chosen on 3 Nov 2026. It replaces testpulse (`../testpulse/`, parked) and the gateway (`../gateway/`, parked).

What it does: a sender (your app) posts events. Herald delivers each event to every customer URL subscribed to it, and keeps retrying until the customer's server says 2xx or the event goes to a dead-letter queue. This is the problem Stripe, Razorpay and GitHub solve for their webhooks.

Why this one: any backend interviewer understands it in one sentence, and the hard parts (duplicates, retries, slow customers, crashes) are exactly what backend interviews ask about.

Real users: your GitHub repos (webguardx, agentic-qa) send their webhooks to Herald, and Herald delivers them to your own Discord or Slack webhook and to a small receiver app you run.

## Rules
- You write the code. Claude reviews and asks "why".
- Each milestone is done only when it runs with `docker compose up` and has at least one test.
- Nothing goes on the resume until it is deployed and has load-test numbers.
- Only this project. Do not start another one before a new job offer is signed.

## Stack
Python 3.12, FastAPI, Pydantic, SQLAlchemy + Alembic, Postgres 16, Redis 7, httpx, pytest, Docker Compose, k6, toxiproxy.

## Step 0 (Sat 7 to Sun 8 Nov): skeleton
- Repo, `docker compose up` starts api + postgres + redis.
- `GET /healthz` checks the DB and Redis, and has a test.

## Milestone 1 (Sat 14 to Sun 15 Nov): accept and deliver
- `POST /v1/apps` creates a sender app and returns an API key once. Store only its SHA-256 hash.
- `POST /v1/endpoints` registers a customer URL and the event types it wants (`payment.success`, `push`, ...). Returns a signing secret once.
- `POST /v1/events` (header `Authorization: Bearer <key>`) takes `{type, payload}`, stores the event, and creates one delivery row per subscribed endpoint. Returns 202 at once: delivery happens later.
- A worker picks pending deliveries and POSTs them with httpx, with a 10 s timeout.
- `GET /v1/events/{id}` shows each delivery: status, attempts, last response code.

**Done when:** a real GitHub push webhook reaches your Discord through Herald.

Check questions: Why return 202 and not wait for delivery? What happens to the event if the API process dies right after the 202?

## Milestone 2 (21-22 Nov and 28-29 Nov): never lose, never spam
- **Idempotent intake:** an `Idempotency-Key` header. The same key twice stores one event and returns the same response.
- **Outbox:** the event and its delivery rows are written in one transaction. The worker reads from the DB, so a crash between "saved" and "queued" loses nothing.
- **Retries:** exponential backoff with jitter (about 30 s, 2 min, 10 min, 1 h, 6 h, ...). After 3 days, move to dead-letter. `POST /v1/deliveries/{id}/retry` retries one by hand.
- **Signatures:** each request carries `X-Herald-Signature` = HMAC-SHA256 of timestamp + body with the endpoint secret. Write the receiver-side check too, and reject old timestamps (replay attack).
- **Rate limit:** 100 events per second per app, Redis token bucket in Lua. Over the limit returns 429 with `Retry-After`.
- **Concurrency tests:**
  - 20 parallel posts with the same `Idempotency-Key` store one event.
  - 3 workers running at once never deliver the same attempt twice (`SELECT ... FOR UPDATE SKIP LOCKED`, or say why you chose something else).
  - 500 parallel requests on one key let exactly the bucket size through.

**Done when:** all three concurrency tests pass against real Postgres and Redis in Docker.

Check questions: What does "at-least-once" mean for the customer, and why can't you promise exactly-once over HTTP? How does the customer de-duplicate (hint: send an event id header)?

## Milestone 3 (December weekends): slow customers, deploy, measure
- **Isolation:** one customer's endpoint that takes 10 s or is down must not slow delivery to everyone else. Per-endpoint concurrency limit, and a circuit breaker per endpoint (open after N failures, probe later).
- **Ordering:** decide and write down: do you promise order per endpoint? What does it cost?
- **Replay:** `POST /v1/endpoints/{id}/replay?since=...` re-sends events from a time range.
- **Backpressure:** cap pending deliveries per endpoint. What happens when the cap is hit? Write the decision.
- **Deploy:** any host with managed Postgres and Redis. Structured logs with a request id, `/metrics`.
- **Load test:** k6 on `POST /v1/events`. Record p50, p95, p99 and delivery lag (event created to delivered) at a fixed rate, with 10% of endpoints slow. Note what broke first.

**Done when:** it is deployed, your repos send through it, and the numbers are in this README.

## Milestone 4 (January, 2 weekends): break it on purpose
A script under `chaos/` that sends a steady load of events while it breaks one thing, then checks every event against the receiver's log.

| Fault | Check |
|---|---|
| Kill the worker mid-delivery | Every event delivered at least once; duplicates counted |
| Kill Postgres for 30 s | No accepted (202) event is lost |
| Stop Redis | Rate limiter fails open or closed, and you wrote down why |
| Receiver slow and dropping 30% of requests (toxiproxy) | Retries recover; other endpoints' lag stays normal |
| Receiver down for 1 hour | Breaker opens; events go out after it comes back, or to dead-letter |

**Done when:** each fault runs 20 times and the table (lost, duplicate, max lag) is in this README.

## Data model (starting point; change it and say why)
- `apps(id, name, api_key_hash, created_at)`
- `endpoints(id, app_id, url, secret, event_types[], status, created_at)`
- `events(id, app_id, type, payload jsonb, idempotency_key, created_at)`, unique `(app_id, idempotency_key)`
- `deliveries(id, event_id, endpoint_id, status, attempt, next_attempt_at, last_status_code, last_error, created_at)`, index `(status, next_attempt_at)`
- `delivery_attempts(id, delivery_id, attempt, status_code, duration_ms, error, created_at)`

## Backend topics this covers
| Topic | Herald piece |
|---|---|
| HTTP basics | 202 vs 200, 401 vs 403, 429 with Retry-After, 413 payload too big |
| Validation | Pydantic models, one error shape |
| SQL, schema, indexes | Data model above; `EXPLAIN` the worker's pick query |
| Transactions | Event + deliveries in one transaction (outbox) |
| Locking, races | `FOR UPDATE SKIP LOCKED`; parallel idempotency keys |
| Idempotency | Intake key; event id header for the customer |
| Redis, rate limiting | Lua token bucket per app |
| Caching | Endpoint lookup by app and event type, invalidate on change |
| Background jobs | Delivery worker |
| Delivery guarantees | At-least-once, dead-letter, manual retry, replay |
| Timeouts, retries, breaker | Per-endpoint backoff with jitter and circuit breaker |
| Backpressure | Pending cap per endpoint |
| Async, GIL | Many slow outbound HTTP calls: async httpx vs threads, measure |
| Connection pooling | DB pool vs worker count; httpx connection limits |
| Security | HMAC signatures, timestamp check, SSRF: block customer URLs that point at localhost or private IPs |
| API keys, revocation | Hash only; revoke and rotate the endpoint secret |
| Migrations | Add a column to `deliveries` that already has data |
| Partitioning | Partition `delivery_attempts` by day; how you would shard by app |
| Logs, metrics | Request id everywhere; delivery lag and success rate per endpoint |
| Deployment, CI/CD | `/healthz`, two API instances, GitHub Actions runs the tests |
| Testing | Integration tests on real Postgres and Redis; chaos tests |
| SQL vs NoSQL | Notes only: why Postgres for the queue here, and when you would move to Kafka or SQS |
| P99 | Which dependency makes the tail slow under load |

## Resume line, only once Milestone 4 is done
> Built and deployed a webhook delivery service (FastAPI, Postgres, Redis): idempotent intake with a transactional outbox, at-least-once delivery with backoff, dead-lettering and replay, HMAC-signed requests, and per-endpoint circuit breakers so slow customers don't delay others; p95 intake __ ms at __ req/s, and 0 lost events across __ fault-injection runs.
