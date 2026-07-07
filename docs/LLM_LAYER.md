# Market Terminal V3 — LLM Analysis Layer

Version: 1.0 · Date: 2026-07
Extends PRD **E7** from single-provider (Anthropic) to **multi-provider**:
Anthropic, OpenAI, Google, and OpenRouter (aggregator for additional models)
behind one abstraction. Architecture context: [`ARCHITECTURE.md`](ARCHITECTURE.md) §6.

Design stance: LLMs are an **analysis and narration layer** over terminal
data. They never trade, never modify state, and every LLM feature has a
deterministic local fallback — the terminal is fully functional offline.

---

## 1. Provider abstraction (`data::llm`)

```
trait LlmProvider {
    fn id(&self) -> LlmProviderId;                  // anthropic | openai | google | openrouter
    fn models(&self) -> Vec<ModelInfo>;             // id, context window, pricing tier
    async fn chat(&self, req: ChatRequest) -> ChatStream;  // streaming; tool-use capable
}
```

- Implementations sit on the shared `data::http` client (retry/backoff/tracing
  — T-1.2); provider-specific wire formats are translated at the boundary into
  one internal `ChatRequest`/`ChatEvent` shape (messages, system prompt, tool
  definitions, temperature, max tokens).
- **OpenRouter** is one adapter that exposes many third-party models (DeepSeek,
  Llama, Qwen, …) — the cheapest way to satisfy "different LLMs" beyond the big
  three; internally it is just another `LlmProvider` with a larger model list.
- Token usage and cost are accounted per request (provider-reported usage ×
  configured price table) and aggregated per task/day.
- API keys per provider in the OS keyring; a provider without a key is simply
  absent from routing (no errors, features fall back).

## 2. Task routing

Every LLM feature is a named **task** with a routing entry. Routing is user
configuration (with shipped defaults), not code:

| Task | Purpose | Default tier | Fallback chain |
|---|---|---|---|
| `key_activity.summary` | summarize Key Activity events (prompt exists in `domain::keyactivity`) | cheap/fast | other cheap model → local template |
| `morning.brief` | morning market overview: breadth regime, sector leaders, FUTOI/HI2 anomalies, expiration calendar | strong | mid model → local template |
| `screener.explain` | one-paragraph "why did this instrument surface" for a screener row | cheap/fast | local template |
| `strategy.critique` | review a backtest/gate report: overfitting smells, fragile parameters, unrealistic assumptions | strong (reasoning) | mid model → skip |
| `journal.review` | weekly trade-journal review: patterns by time/setup/instrument, discipline breaches | strong | mid model → local stats table |
| `news.digest` (C, with M7 news source) | condense calendar/news into instrument-tagged bullets | cheap/fast | skip |
| `chat.terminal` (C) | chat panel with **read-only** tool use over IPC (E7.3) | strong (tool-use) | disabled without key |

Routing entry: `task → [ (provider, model, effort/params), … ]` — first healthy
entry wins; per-task temperature/max-tokens; chain ends in the local fallback.

**Model-tier guidance (defaults):** cheap/fast = small models (e.g.
Haiku-class / mini-class); strong = frontier models (e.g. Sonnet/Opus-class,
GPT-frontier, Gemini-Pro-class). Concrete model ids live in config, not code —
they churn too fast to hard-code.

## 3. Context builders

Each task has a deterministic **context builder** in `app` that assembles its
input from terminal state (not from the LLM's imagination):

- structured JSON context blocks (breadth metrics, screener rows, gate-report
  numbers, journal aggregates) + a fixed task prompt template;
- context size budgeted per task (truncation rules explicit, biggest-signal
  first);
- the same context object feeds the **local fallback** renderer (template
  text), guaranteeing feature parity offline;
- prompt templates are versioned files, not string literals scattered in code.

## 4. Privacy and safety

1. **Redaction pass** before any external call: account identifiers, absolute
   account equity, and absolute position sizes are stripped or normalized to
   relative values (% of equity, R-multiples). Redaction is a pure `domain`
   function with tests.
2. **Per-task privacy level:** tasks touching the journal/positions
   (`journal.review`) can be pinned to a designated provider or disabled;
   market-only tasks (brief, screener) are unrestricted.
3. **No write tools:** `chat.terminal` tool-use exposes read-only IPC commands
   (query panels, read metrics, open a panel). No order placement, no settings
   mutation, no file access. Tool registry is an explicit allowlist.
4. **Prompt-injection posture:** external text (news, later M7) is wrapped as
   untrusted data in prompts; tool allowlist + read-only surface bounds the
   blast radius by construction.
5. Keys in keyring only; requests/responses logged without keys; response log
   is local and rotated.

## 5. Cost control and caching

- Per-day cost budget (configurable, default modest) — hard stop with UI
  notice; per-task counters visible in a small usage panel section (health
  panel, E8.3).
- Response cache keyed by (task, context hash): scheduled tasks (morning
  brief) never re-bill for the same context; manual "regenerate" bypasses.
- Scheduled tasks run at configured times (brief pre-open); on failure they
  degrade to the local fallback and mark the output as such.

## 6. Failure and degradation ladder

```
primary (provider, model)
  → next entry in task chain (different provider)
    → local deterministic fallback (template over the same context)
      → feature renders with "offline summary" badge
```

Timeouts are short (LLM output is advisory, never blocking); streams render
incrementally; a hung stream cancels at deadline and falls down the ladder.

## 7. Epic/TODO mapping

| Item | Where |
|---|---|
| Multi-provider client, streaming, keyring, fallback | T-6.1 (reworked: multi-provider) |
| Task routing config + cost caps + usage UI | T-6.11 (new) |
| Key Activity summary on live data | T-6.2 |
| Morning brief | T-6.3 |
| Chat panel with read-only tool use (C) | T-6.4 |
| Redaction function + tests | part of T-6.1 |
