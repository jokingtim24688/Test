# OmniRoute v2 — "simpler, but better"

Status: **design proposal, unverified against the existing OmniRoute source**
(the source is not in this session's repo; see Assumptions).

## Assumptions

Based on the `auto-launcher` skill, OmniRoute is a local Windows service that
other tools route calls through, alongside Graphify. This note assumes it is an
**LLM/API gateway**: one endpoint in front of several model providers. If that's
wrong, the scope table below is the only part that changes — the method
(freeze a contract, keep 8 features, delete the rest) still holds.

## Verdict

Yes — a rewrite is usually *easier* than a cleanup for this class of service,
because the whole thing is a thin, stateless request-translation layer with one
small piece of real state (usage + budget). The hard parts are not the code,
they're the provider quirks. So: keep the quirk handling, throw away the
infrastructure.

## Scope: the 8 things v2 keeps

| # | Feature | Why it survives |
|---|---------|-----------------|
| 1 | One OpenAI-compatible endpoint (`POST /v1/chat/completions`) | Every client/SDK already speaks it; zero client changes |
| 2 | Provider adapters (Anthropic, OpenAI, Google, OpenRouter, local Ollama) | The actual value; ~150 lines each |
| 3 | Model aliases → ordered candidate list, in one YAML file | Replaces a routing engine with a lookup |
| 4 | Failover + bounded retry (429 / 5xx / timeout) | The reason a router exists at all |
| 5 | SSE streaming passthrough | Non-negotiable for UX |
| 6 | API keys with per-key monthly budget | Stops runaway spend |
| 7 | Usage + cost log in SQLite | Required to prove the thing pays for itself |
| 8 | `/healthz` + `/stats` (read-only HTML, no JS build) | Operability without a frontend project |

## Scope: what gets deleted

- Postgres + Redis → **single SQLite file** (WAL mode). One process, one file, no daemons to auto-launch.
- Plugin/middleware system → plain functions in a list.
- Admin SPA → one server-rendered page.
- Custom auth → bearer tokens hashed in SQLite.
- Multi-file config / env sprawl → `omniroute.yaml` + `.env` for secrets only.
- Docker Compose stack → single binary/script + a Windows service wrapper.

Target: **under ~2,000 lines**, one language, startup under 1s.

## Improvements over v1 (the point of the rewrite)

1. **Price-aware routing.** Pricing table as a dated data file; pick the cheapest candidate that meets a declared capability set (context length, tools, vision). Return `X-OmniRoute-Cost-Estimate` on every response.
2. **Exact-match response cache.** Hash of (model, messages, params) → response, with TTL, opt-in per key, never for streaming-with-tools. This is the single biggest cost lever (see economics).
3. **Circuit breaker per provider.** After N consecutive failures, skip that provider for 60s instead of retrying into it.
4. **Idempotency keys.** Prevents double-billing when a client retries a request that already reached a provider.
5. **Recorded-fixture test suite.** Capture one real response per provider per feature (text, stream, tools, vision), replay offline. Makes provider API drift a failing test instead of a 2am outage.
6. **Structured logs with secret redaction by construction** — keys never enter the log struct, rather than being regex-scrubbed on the way out.

## What can go wrong (pre-mortem)

| Risk | Reality | Mitigation |
|------|---------|-----------|
| Mid-stream provider error | SSE already returned HTTP 200, so you cannot change the status | Emit a terminal `error` event in-stream; never retry a stream that already emitted content |
| Tool-calling schema differences | The *one* place providers genuinely diverge; silent corruption of agent loops | Normalize to one internal shape; fixture test per provider; fail loudly on unmappable fields |
| Token counts differ from provider's billing | Your cost numbers drift from the real invoice | Always prefer provider-reported usage; estimate only as fallback and label it |
| Retry storms | A provider blip becomes a self-inflicted outage and a double bill | Cap total attempts at 3, exponential backoff + jitter, circuit breaker, idempotency keys |
| Pricing table rot | Cost routing silently optimizes against stale prices | Prices carry an `as_of` date; warn in `/stats` past 60 days |
| Cache leaking between users | Serious privacy bug | Cache key includes API key identity; cache off by default |
| SQLite write contention | Logging blocks requests under load | WAL mode + async batched writes; usage log is append-only |
| Single point of failure | Everything you own now depends on one process | `/healthz`, auto-restart as a Windows service, and a documented direct-to-provider fallback |
| Rewrite never ships | The usual failure mode | Freeze the HTTP contract first, run v2 beside v1 on a different port, cut over per client |

## Does it pay for itself?

Running cost, self-hosted: **~$0** on your own machine (idle Python/Node process,
a few watts), or **~$5/mo** on a small VPS if it needs to be reachable remotely.
No GPU, no managed DB. Unlike a crypto miner, there's no electricity-vs-revenue
race here — the cost floor is essentially zero.

It earns money only through the LLM bill it sits in front of:

- **Cache hits** — a 20% exact-hit rate on a $200/mo spend saves ~$40/mo.
- **Price routing** — sending the easy calls (classify, extract, rewrite) to a cheap model is typically a 30–60% cut on *those* calls.
- **Budget caps** — the value is bounded by the one runaway loop they prevent.

Break-even is trivially met **if** monthly LLM spend is meaningful (say >$50).
Below that, the honest answer is that v2 is worth building for reliability and
for not maintaining v1, **not** for the savings — and the savings claim should
be checked against `/stats` after a month, not assumed.

The real cost is your time: roughly a weekend for a working v2, plus the
provider-quirk tail. The cheapest version of this project is to keep v1 running
until v2 passes the same fixture suite.
