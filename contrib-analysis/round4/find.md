# Round 4: new candidates (parity bugs)

Checked against PASSED.md and the round2 and round3 `02_candidates.md` files. None of these topics appeared there. Duplicate searches used the GitHub REST search API (`is:issue` and `is:pull-request`). Nothing was executed.

Dropped after the duplicate check:
- nomic `vision_model` missing from the `__init__` overloads / `embed_image(model=None)`: already filed as issue #41020 (open), and PR #41104 (closed) tried to fix it.
- `OllamaLLM._stream`/`_astream` set `generation_info["finish_reason"] = self.stop` (the stop-sequence list) instead of `done_reason`: issue #37370 and PRs #37371 and #37460, all closed.

---

## R4-1 fix(ollama): pass `chunk=` to `on_llm_new_token` in `_stream`/`_astream`

- **Where:** `libs/partners/ollama/langchain_ollama/chat_models.py:1304` (`_stream`) and `:1386` (`_astream`); `libs/partners/ollama/langchain_ollama/llms.py:531` (`_stream`) and `:562` (`_astream`).
- **Bug:** The aggregation paths in the same files pass `chunk=chunk` (`chat_models.py:1152/1178`, `llms.py:451/409`). The `.stream()` paths call `run_manager.on_llm_new_token(chunk.text, verbose=...)` without it. As a result, callback handlers get `chunk=None` during `stream()`/`astream()`, so they cannot see `reasoning_content`, `tool_call_chunks` or `usage_metadata`. They do see them under `invoke()`. Every sibling partner passes the chunk: groq (`:742/798`), openrouter (`:633/676/725/768`) and fireworks (`:1393`).
- **Related merged PR:** #41643 `fix(anthropic): report streamed content-block tokens to callbacks`, the same class of fix for Anthropic.
- **Repro (not run):** Write a `BaseCallbackHandler` that records the `chunk` kwarg. Mock `ChatOllama._client.chat` to yield 2 stream parts. Run `list(llm.stream("hi", config={"callbacks":[h]}))`. Expect a `ChatGenerationChunk`, get `None`. The same happens for `OllamaLLM`.
- **Fix:** Add `chunk=chunk` to the 4 call sites. Optional bundle: the sync warning at `chat_models.py:1253` is missing a space (`"...'load'." "This..."`). The async twin at `:1335` has it.
- **Test:** Write sync and async tests for ChatOllama and OllamaLLM with a mocked client that assert `isinstance(recorded_chunk, ChatGenerationChunk/GenerationChunk)`. Use the patching pattern in `tests/unit_tests/test_chat_models.py`.
- **Duplicate check:**
  - `ollama on_llm_new_token`: issues returned only unrelated hits (#37869, #37878 about agent ainvoke). PRs: #17348 and #15169 (community-era, closed, unrelated).
  - `ollama stream callback chunk`: nothing relevant in either issues or PRs.
  - `ChatOllama _stream chunk callback`: old closed issues only (#29383, #13072), and no PRs.
  - `ollama chunk=chunk` and `ChatOllama on_llm_new_token`: nothing relevant. Open PR #38019 `feat(ollama): native content-block streaming events` touches streaming generally. Check that it does not rewrite these lines before opening.

## R4-2 fix(groq,fireworks): report the response's `model` (or call-time override) as `model_name`

- **Where:**
  - groq `libs/partners/groq/langchain_groq/chat_models.py:717` (`_stream`), `:773` (`_astream`), `:847` (`_create_chat_result` `llm_output`).
  - fireworks `libs/partners/fireworks/langchain_fireworks/chat_models.py:1382`, `:1451`, `:1505`.
- **Bug:** These always use `self.model_name`. The request params do honor `invoke(..., model="other")` because kwargs are merged into `params`. So `response_metadata["model_name"]` (and the `UsageMetadataCallbackHandler` key) reports the constructor model, not the one actually called. The same is true when the API returns a resolved or aliased model ID.
- **Parity:** openai reads `chunk.get("model")` in `base.py:1667` and `response_dict.get("model", self.model_name)` in `:2140`. openrouter reads `chunk_dict.get("model") or model_name` in `:90` and `response_model or self.model_name` in `:889`. The same files' `_get_ls_params` already use `params.get("model", self.model_name)` (groq `:611`, fireworks `:1312`). Related merged/known: R3-6 xai `stop` honoring call-time `model` (#38335 follow-up).
- **Repro (not run):** Run `ChatGroq(model="a").invoke("hi", model="b")` with a mocked client whose response has `"model": "b"`. Today `msg.response_metadata["model_name"] == "a"`.
- **Fix:** Use `response.get("model") or params.get("model") or self.model_name`, and the equivalent for chunks. groq is the core of this PR. fireworks can go in a separate PR.
- **Test:** Mock `client.create` and assert the model name for invoke, stream and the call-time override.
- **Duplicate check:**
  - `groq model_name response_metadata`: issues returned only unrelated hits (#40518 UsageMetadataCallbackHandler drops when model_name missing, which is related but different). PRs: #38070 and #32315, both unrelated.
  - `groq model_name`: issue #38659 and PR #38759 (`_combine_llm_outputs` None-safety, open). These touch `_combine_llm_outputs`, not `model_name`. Mention them to avoid conflicts.
  - `groq model_name override`, `groq response model`, `ChatGroq model_name llm_output`, `fireworks model_name response_metadata`: no match.

## R4-3 fix(langchain): `LLMToolEmulator` should use `response.text`, not raw `response.content`

- **Where:** `libs/langchain_v1/langchain/agents/middleware/tool_emulator.py:192-196` (sync) and `:247-251` (async).
- **Bug:** `ToolMessage(content=response.content, ...)` forwards the emulator model's raw content. With reasoning or thinking models (Anthropic extended thinking, OpenAI Responses reasoning) that is a list that includes `thinking`/`reasoning` blocks, sometimes with signatures. Those blocks are injected into the agent's tool result and sent back to the main model. This can leak reasoning or produce provider errors, for example Anthropic rejects a `thinking` block inside `tool_result` content.
- **Parity:** `SummarizationMiddleware` uses `response.text.strip()` for the same kind of internal model call (`summarization.py:952/986`).
- **Repro (not run):** Use a `GenericFakeChatModel` that returns `AIMessage(content=[{"type":"thinking","thinking":"..."},{"type":"text","text":"72F"}])`. The emulated `ToolMessage.content` is that list, but it should be `"72F"`.
- **Fix:** `content=response.text` in both paths. Note that this changes behavior: list content becomes a string.
- **Test:** Sync and async tests with a fake model returning thinking and text blocks, asserting `tool_msg.content == "72F"`.
- **Duplicate check:**
  - `LLMToolEmulator content` / `LLMToolEmulator`: issue #34274 (closed; redundant condition / ImportError). PRs: #39288 (merged batch; read its body: it touches only the `tools_to_emulate` check and the required `model`), #39290 (import error), #34280 (open, `model` not Optional), #41579 (internal calls kept off the messages stream). None of these change `response.content`.
  - `tool_emulator thinking`: 0 issue hits, 0 PR hits.

## R4-4 fix(ollama): `ChatOllama._get_ls_params` drops `ls_max_tokens` (`num_predict`)

- **Where:** `libs/partners/ollama/langchain_ollama/chat_models.py:1186-1200`.
- **Bug:** `OllamaLLM._get_ls_params` (`llms.py:314-321`) adds `ls_max_tokens` from `kwargs.get("num_predict", self.num_predict)`, and that is unit-tested (`tests/unit_tests/test_llms.py:21-34`). `ChatOllama` overrides `_get_ls_params` without calling `super()` and never sets `ls_max_tokens`. So LangSmith traces for ChatOllama never show the token limit, even with `num_predict=256` set.
- **Parity:** This is the sibling class in the same package.
- **Repro (not run):** `ChatOllama(model="llama3", num_predict=3)._get_ls_params()` has no `"ls_max_tokens"` key.
- **Fix:** Add `if max_tokens := params.get("num_predict", self.num_predict): ls_params["ls_max_tokens"] = max_tokens`. That reads the call-time `num_predict` from `params`/`kwargs`, the same way as the LLM class.
- **Test:** Mirror `test_llms.py`'s ls-params test in `test_chat_models.py`, covering both the constructor value and a call-time override.
- **Duplicate check:**
  - `ChatOllama ls_max_tokens`: 0 issue hits, 0 PR hits.
  - `ollama ls_max_tokens`: issue #35300 (closed; core integer temperature, unrelated), 0 PRs.
  - `ChatOllama num_predict tracing`: 0 issue hits, 0 PR hits.
