# Partner contribution candidates (round 2)

Nothing here was executed. All repros are written out but have not been run. The Mistral `on_llm_new_token` token-type issue is left out because it is already known.

Pattern being followed: #41553 (`fix(anthropic): handle null cache-creation token counts`) changed `.get(k, 0)` to `.get(k) or 0`. `dict.get(k, 0)` falls back to 0 only when the key is **missing**. When the key is present with the value `null` (common with OpenAI-compatible servers such as TGI, vLLM and proxies), it returns `None`. Groq (`chat_models.py:1614-1620`), Fireworks (`:473-474`) and OpenRouter (`:1538-1542`) already handle `None` with `or 0` / `is not None`. That gives in-repo precedent for the outliers below.

## 1. fix(huggingface): handle null token counts in usage metadata (STRONG)
- `libs/partners/huggingface/langchain_huggingface/chat_models/huggingface.py:279-285` (`_convert_chunk_to_message_chunk`), `:709-711` (`_create_chat_result`), `:856-858` (`_stream` usage-only chunk), `:923-925` (`_astream` usage-only chunk).
- Defect: at `:284` the fallback `input_tokens + output_tokens` only runs when `total_tokens` is missing. If `prompt_tokens` is `null` in that case, this raises `TypeError: unsupported operand type(s) for +: 'NoneType' and 'int'`. At the other sites, `None` values end up in `UsageMetadata`. Later, when streamed chunks are summed (`AIMessageChunk.__add__` -> `add_usage`), `None + int` raises a `TypeError`. Typed `int` fields also break.
- Repro: `_convert_chunk_to_message_chunk({"choices":[{"delta":{"role":"assistant","content":"x"}}],"usage":{"prompt_tokens":None,"completion_tokens":3}}, AIMessageChunk)` raises a TypeError. Separately, a usage chunk `{"choices":[],"usage":{"prompt_tokens":5,"completion_tokens":None,"total_tokens":None}}` produces `usage_metadata` with `None` values.
- Fix: use `usage.get(k) or 0` everywhere. Compute total as `usage.get("total_tokens") or (input_tokens + output_tokens)`. Optionally factor this into one `_create_usage_metadata` helper, as Fireworks does.
- Test: parametrize over each null field for chunk conversion and `_create_chat_result`. Assert the counts are ints and that adding two chunks does not raise.
- Merged PR ref: #41553.

## 2. fix(perplexity): handle null token counts in `_create_usage_metadata` (STRONG, small)
- `libs/partners/perplexity/langchain_perplexity/chat_models.py:95-97`.
- Defect: same as candidate 1. With `{"prompt_tokens": None, "completion_tokens": 10}` (no `total_tokens`), line 97 raises a TypeError. With `total_tokens: None`, `UsageMetadata.total_tokens` becomes `None`. The adjacent `reasoning_tokens` / `citation_tokens` handling already checks `is not None`, so this fix is consistent with the same function.
- Repro: `_create_usage_metadata({"prompt_tokens": None, "completion_tokens": 10})`.
- Fix: `.get(k) or 0`, plus `total = token_usage.get("total_tokens") or input_tokens + output_tokens`.
- Test: unit-test `_create_usage_metadata` with null values for each field.
- Merged PR ref: #41553.

## 3. fix(mistralai): handle null token counts in usage metadata (MEDIUM)
- `libs/partners/mistralai/langchain_mistralai/chat_models.py:406-408` (stream chunk) and `:848-850` (non-stream result).
- Defect: there is no arithmetic here, so nothing raises immediately. However, a `null` count (seen from Mistral-compatible self-hosted or Azure endpoints) leaks `None` into `usage_metadata`. Summing streamed chunks then raises a TypeError in `add_usage`, and callbacks and tracing see a non-int value. This is weaker than candidates 1 and 2 because I could not confirm that the hosted Mistral API ever returns null. Consider combining it with candidate 1 as a cross-partner PR, or skipping it.
- Fix and test: same as above.

## 4. fix(huggingface): `_astream` / `_agenerate` diverge from sync for non-endpoint LLMs (MEDIUM)
- `huggingface.py:827-894` (`_stream`) vs `:896-948` (`_astream`); `:728-767` (`_generate`) vs `:769-806` (`_agenerate`).
- Defect: `_stream` checks `_is_huggingface_endpoint(self.llm)` and otherwise falls back to `self.llm._stream(self._to_chat_prompt(...))`. `_astream` has no such branch and always calls `self.llm.async_client.chat_completion`. As a result, `ChatHuggingFace(llm=HuggingFacePipeline(...)).astream(...)` raises `AttributeError` (no `async_client`) instead of a clear `NotImplementedError`, which is the error `_agenerate` raises for pipelines. Likewise, `_generate` honors `streaming=True` for non-endpoint LLMs (`self.llm._stream`), but `_agenerate` ignores it.
- Repro: build `ChatHuggingFace` around a mocked `HuggingFacePipeline` (needs model download, or a mock that sets `model_id`), then `await chat.ainvoke(...)` or `async for _ in chat.astream(...)` and observe the AttributeError.
- Fix: in `_astream`, raise `NotImplementedError` for pipelines (matching `_agenerate`). For other LLMs, fall back to `self.llm._astream(llm_input, ...)` and wrap each chunk in `AIMessageChunk`, mirroring the sync path. The extra `"stop": stop` in sync `_generate` params is redundant (`_create_message_dicts` already sets it), so it is not a bug.
- Test: mock a non-endpoint LLM and assert that `_astream` delegates or raises `NotImplementedError` in the same way as sync.
- Caveat: check how often users wrap a pipeline in async code. The value is mostly a clearer error, so this is lower priority.

## Rejected
- Groq, Fireworks, OpenRouter null usage: already handled with `or 0`.
- Ollama: `prompt_eval_count` is typed `int | None`; I did not see arithmetic on it in the sampled code, so I did not pursue it.
