# Round 4 rebuttal 1: strict maintainer ("would I assign this?")

Lens: I triage a flood of AI-assisted parity PRs. I assign work that fixes a bug users can observe, has a small diff, and does not reopen settled design questions.

## R4-1 ollama `chunk=` in `on_llm_new_token`
- Half the premise is wrong (see rebuttal 2). In `BaseChatModel.stream()` core calls `self._stream(...)` **without** `run_manager` (`core/langchain_core/language_models/chat_models.py:837`), and the `_should_stream` path in `_generate` does the same (`:2024`). Core then emits `on_llm_new_token(..., chunk=chunk)` itself (`:854`, `:2042`). So ChatOllama's `run_manager.on_llm_new_token` in `_stream` (`chat_models.py:1304`) is effectively dead code, and handlers already get the chunk.
- The `OllamaLLM` half is real: `BaseLLM.stream` passes `run_manager` (`llms.py:559-560`) and does not emit on its own, so `llms.py:531/562` send `chunk=None`.
- I would assign it only re-scoped to "`OllamaLLM` stream callbacks lack `chunk`". `OllamaLLM` is the legacy text-completion class with low priority. The ChatOllama edit is harmless consistency, but the issue cannot claim it as a user bug.

## R4-2 groq/fireworks `model_name`
- Plausible. Override via `invoke(model=...)` is an unusual pattern, and the aliasing argument is speculative. Groq returns `model` in its responses, so using it is cheap.
- Conflict risk: open PR #38029 `feat(groq): native content-block streaming events` touches the same `_stream` code, and #38759 touches `_combine_llm_outputs` (which also hardcodes `self.model_name` at `:894`). I would ask the author to wait or coordinate. Groq-only, with fireworks dropped, I would assign it.

## R4-3 `LLMToolEmulator` `response.text`
- Best of the four. v1 middleware is actively maintained, the fix is two lines, and it follows the `SummarizationMiddleware` precedent. It is a behavior change (list to str), but the old behavior is clearly accidental for a "tool result" string.
- One push-back: who runs an emulator with a thinking model? It happens: the default emulator model is an Anthropic model, and users do turn on thinking. Assign.

## R4-4 ChatOllama `ls_max_tokens`
- Trace-metadata only. It is correct and tiny, with a sibling precedent that is unit-tested. Low value, but low risk too. I would assign it if bundled, or merge a drive-by PR.

| ID | score | verdict |
|---|---|---|
| R4-1 | 4 | Re-scope to OllamaLLM only; chat half is not a bug |
| R4-2 | 6 | Groq-only; coordinate with #38029/#38759 |
| R4-3 | 8 | Assign |
| R4-4 | 6 | Assign as small trace fix |
