# Graphify Wiki — Index

> Local Graphify/OmniRoute services are **not reachable from this cloud session**
> (Linux container, no `tasklist`/`start`, no localhost services). This wiki is
> maintained in-repo so future prompts have the context; re-sync it into the real
> Graphify graph from a local Claude Code/Desktop session.

## Nodes

- [OmniRoute v2 — simpler rebuild](omniroute-v2.md) — feasibility, scope cut, improvements, failure modes, cost model.
- [Routing free/third-party models into Claude Code](claude-code-free-model-routing.md) — `ANTHROPIC_BASE_URL` shim, why it needs an *Anthropic-in* adapter (not OmniRoute's OpenAI-out shape), free-tier limits, caching, ToS, data-training cost.

## Decisions

- Goal is **not** a general API gateway: it is getting free/third-party models to serve Claude Code. That needs an Anthropic-Messages-API-compatible shim, which is the opposite direction from OmniRoute v1/v2's OpenAI-compatible endpoint.
- Try existing tools (`claude-code-router`, LiteLLM proxy) before building.

## Open questions

- Where does the current OmniRoute source live? (not in `jokingtim24688/test`, which is empty)
- Which providers' free tiers specifically? (determines rate-limit and ToS reality)
- Is current LLM spend metered API or a flat subscription? (decides whether savings exist at all)
