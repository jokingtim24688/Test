# Routing free/third-party models into Claude Code

Status: **mechanically confirmed by design (documented env vars + existing OSS tools);
the free-tier economics below are analysis, not measured on the user's accounts.**

## Short answer

Yes, mechanically. Claude Code resolves its API endpoint from environment variables:

| Variable | Purpose |
|---|---|
| `ANTHROPIC_BASE_URL` | Where Claude Code sends requests |
| `ANTHROPIC_AUTH_TOKEN` | Bearer token sent to that endpoint |
| `ANTHROPIC_MODEL` | Model name requested |
| `ANTHROPIC_SMALL_FAST_MODEL` | The cheap model used for background/cheap work |

Point `ANTHROPIC_BASE_URL` at a local shim that speaks the **Anthropic Messages
API** and translates to whatever provider you want. (Verify the exact variable
names against current Claude Code docs / `/config` before wiring — they move.)

**This means OmniRoute v2 is the wrong shape for this job.** v1/v2 as designed
exposes an *OpenAI-compatible* endpoint. Claude Code does not speak that. What's
needed is the reverse adapter: **Anthropic Messages in → provider's format out**.

Also: this already exists. `claude-code-router`, LiteLLM proxy, and y-router do
exactly this. Trying one first is a day of work; building one is a month.

## Why Claude Code is the hardest possible client for a shim

It is not a chat app. Per turn it sends a very large system prompt, ~15-20 tool
definitions, full file contents, and long `tool_use`/`tool_result` chains. The
shim has to faithfully translate:

- **Tool use** — `tool_use` / `tool_result` blocks ↔ the provider's function-calling
  shape, including **parallel** tool calls in one assistant message. This is where
  these shims break, and the break is silent: the agent "works" but edits nothing.
- **Streaming** — Anthropic SSE event types (`content_block_start`, `delta`, `stop`)
  are not OpenAI's chunk shape. Claude Code's UI depends on them.
- **`cache_control` blocks** — see the economics section; this is the killer.
- **Long context** — Claude Code routinely exceeds 100K tokens on real work.
- **Stop reasons** — `tool_use` vs `end_turn` must be mapped exactly, or the agent
  loop never terminates (or terminates mid-task).

## Will free models actually work? (the honest part)

Five things go wrong, roughly in order of how fast they bite:

1. **Rate limits, immediately.** Free tiers are sized for chat — a few requests per
   minute, a token-per-day cap. One Claude Code task is dozens of turns, each
   resending the whole conversation. Expect to exhaust a daily free quota in
   **minutes**, not hours. This is the #1 practical blocker.
2. **No prompt caching.** Claude Code is built around it: the big stable prefix is
   cached, so turn 20 only pays for what's new. Third-party endpoints generally
   ignore `cache_control`, so **every turn re-pays full price for the entire
   context**. On a free tier priced in quota rather than dollars, this multiplies
   consumption by roughly the number of turns.
3. **Your code becomes training data.** Free tiers are usually free *because*
   inputs are used for training. Claude Code sends your source files, and whatever
   secrets are in them. This is the real cost of "free", and it is not recoverable.
4. **Terms of service.** Many free tiers prohibit proxying, resale, or automated
   agent use. Risk is account termination, not a fine. Check the specific
   provider's terms — don't assume.
5. **Tool-calling reliability is the binding constraint, not intelligence.** Free
   tiers serve the small models. A model that writes decent code but is sloppy at
   strict multi-turn tool use will loop, hallucinate edits, claim success on files
   it never wrote, or stall. A weak model in an agent harness isn't "a bit worse" —
   it's qualitatively broken.

### What does work

- **Local models via Ollama** — no rate limit, no ToS issue, no data leaving the
  machine. Bounded only by VRAM. This is the genuinely free option.
- **Split routing** — keep the main agent loop on a capable model and point only
  `ANTHROPIC_SMALL_FAST_MODEL` at the free/cheap one. Captures most of the saving
  with little of the breakage.

## Does it pay for itself?

Infrastructure cost is ~$0 (a local shim process). So the question is only whether
the *output* is worth it.

- **If the LLM bill is the problem:** a Claude subscription is already flat-rate —
  routing to free models saves nothing against it, and only helps against
  metered API spend.
- **The hidden cost is turns.** If a weaker model needs 3x the turns and you have
  to review and redo its work, that's negative ROI at any token price, because
  your time is the expensive input. A $0 model that wastes an hour costs more than
  a $2 call that works.
- **The honest sweet spot** is local models for bulk/mechanical work, a real model
  for the main loop. Not "all free, all the time."

## Recommended order of attack

1. Try `claude-code-router` or LiteLLM proxy against **one** provider. One evening.
2. Measure on a real task: does it complete, how many turns, how much quota.
3. Only if a specific translation gap blocks you, write the shim — and write it as
   an **Anthropic-in** adapter, which is a different project from OmniRoute.
