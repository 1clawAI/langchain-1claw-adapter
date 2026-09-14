# langchain-1claw-adapter

**Vault-backed credential resolution and policy-checked tool calls for LangChain agents.**

> Status: **alpha / reference implementation.** Interfaces will change. Production users should pin a specific commit.

## What this is

LangChain has 1,000+ tool integrations, but most of them take an API key directly as a parameter. In production, this means:

- The agent process holds long-lived credentials in memory
- A compromised agent (prompt injection, jailbreak, or just a bug) can use those credentials for *anything* the credential is scoped to
- There is no per-call policy check, no audit log of *intent*, and no fast revocation that doesn't involve key rotation

This adapter sits between the LangChain `Tool` and the API call. Instead of the agent holding the credential, it holds an opaque *handle*. To use the credential, the tool submits an **intent** to a vault, the vault checks an **operator policy**, and either signs / forwards the request or denies it.

```
+---------+   intent    +-------+   policy check   +--------+
|  Agent  | ----------> | Vault | ---------------> | Policy |
+---------+             +-------+ <--------------- +--------+
                            |    sign / deny
                            v
                     +-----------+
                     | Upstream  |
                     |   API     |
                     +-----------+
```

The agent never reads the credential.

## Why this matters

This is the missing primitive behind several open conversations in the LangChain repo:

- [#36232 — Cryptographic agent identity, intent verification, kill switch](https://github.com/langchain-ai/langchain/issues/36232)
- [#35393 — Agent Identity Verification for Tool Calls](https://github.com/langchain-ai/langchain/issues/35393)
- [#36306 — x402 payment primitive for paid API consumption](https://github.com/langchain-ai/langchain/issues/36306)

Identity primitives answer *who* is calling. Vault-backed intent answers *what* is allowed. You need both.

## Install

```bash
pip install -e .
```

(Not yet on PyPI — pin from git.)

## Usage

```python
from langchain_1claw import VaultBackedTool, MockVault

vault = MockVault(
    policies={
        "weather-api": {
            "endpoint_allowlist": ["api.weather.example/v1/*"],
            "per_call_usd_cap": 0.05,
            "daily_usd_cap": 5.00,
        },
    },
    credentials={
        "weather-api": "sk-live-real-key-stays-here-not-in-agent-memory",
    },
)

weather_tool = VaultBackedTool(
    name="get_weather",
    description="Look up the current weather for a city.",
    vault=vault,
    credential_handle="weather-api",
    endpoint="https://api.weather.example/v1/current",
)

# The agent sees `weather_tool` like any LangChain tool — but cannot see
# the credential, and cannot call endpoints outside the allowlist.
```

When the agent invokes the tool, the adapter:

1. Builds an intent: `{ tool: "get_weather", endpoint: "...", args: {...}, estimated_usd: 0.01 }`
2. Submits to the vault
3. Vault checks policy (endpoint allowlist, per-call cap, daily cap)
4. If allowed: vault performs the upstream call with the real credential, returns the response
5. If denied: vault returns a structured `IntentDeniedError`

## Plugging in a real vault

`MockVault` is for development. Real deployments swap in:

- **`OneClawVault`** — HSM-backed credential storage, policy-checked intent submission, audit log. See https://x.com/1clawAI.
- Any other implementation of the `Vault` protocol in `src/langchain_1claw/vault.py`.

The interface is intentionally narrow — three methods (`submit_intent`, `get_policy`, `record_audit`) — so it's easy to back with any vault / KMS / HSM that supports policy-checked signing.

## What's in this repo

- `src/langchain_1claw/vault.py` — `Vault` protocol + `MockVault` reference implementation
- `src/langchain_1claw/intent.py` — `Intent`, `IntentResult`, `IntentDeniedError` types
- `src/langchain_1claw/tool.py` — `VaultBackedTool` (LangChain `BaseTool` subclass)
- `examples/paid_api_call.py` — end-to-end example with policy denial demo
- `examples/tier2_live_agent_demo.py` — a real LangChain agent (Claude Haiku) driving a multi-turn conversation through the vault, including an adversarial prompt asking it to leak the credential. See [`INTEGRATION_VERIFIED.md`](./INTEGRATION_VERIFIED.md#tier-2--live-llm-agent-adversarial-prompt-included) for the full transcript and result.
- `tests/` — sanity tests

## Roadmap

- [ ] Async support (`arun` path)
- [ ] LangGraph node wrapper
- [ ] `OneClawVault` adapter (talks to a live 1Claw vault)
- [ ] x402 payment integration
- [ ] Per-agent-DID scoping (pairs with cryptographic identity from #35393 / AIP)
- [ ] Real upstream HTTP client (currently demos via stub)

## License

MIT — see [LICENSE](./LICENSE).

## Why "1claw"

[@1clawAI](https://x.com/1clawAI) is building the vault + policy + intent layer for AI agents. This adapter is the LangChain-side glue. The same pattern works for AutoGen, CrewAI, MCP servers, and direct OpenClaude integrations — separate adapters for each.
