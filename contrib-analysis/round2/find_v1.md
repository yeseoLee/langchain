# Round 2: contribution candidates in `libs/langchain_v1/langchain`

I checked these against the code at HEAD (`5dcec18`). I did not run anything: no Python, uv or pytest. Line numbers refer to HEAD.

## Candidate 1: `fix(langchain)`: infer `openai` for `o4-*` model names in `init_chat_model`

**Where:** `libs/langchain_v1/langchain/chat_models/base.py:558-569` (`_attempt_infer_model_provider`). The OpenAI prefixes are `gpt-`, `o1`, `o3`, `chatgpt`, `text-davinci`. There is no `o4`. The docstring table at `base.py:276` lists `gpt-... | o1... | o3...`.

**Evidence that other code knows the model:** `libs/partners/openai/langchain_openai/data/_profiles.py` has an `"o4-mini"` profile. The test table in `tests/unit_tests/chat_models/test_chat_models.py:170-193` covers `o3` but not `o4`.

**Repro (not executed):**
```python
from langchain.chat_models import init_chat_model
init_chat_model("o4-mini")
# ValueError: Unable to infer model provider for model='o4-mini'. ...
init_chat_model("o3-mini")  # works -> ChatOpenAI
```

**Reference PR:** #40837 (Bedrock Mantle in `init_chat_model`). It belongs to the same series of provider-table updates. #40844 (GPT-6 fallback regex) is another example of keeping hard-coded model tables up to date.

**Fix:** add `"o4"` to the OpenAI prefix tuple and to the docstring table.

**Test:** add `("o4-mini", "openai")` and `("O4-Mini", "openai")` to the `test_attempt_infer_model_provider` parametrize list.

**Risk / pushback:** this change is small. A maintainer could answer "use the `openai:` prefix", because the docstring calls inference best-effort. It is still the same kind of change as the existing `o1`/`o3` entries, and no other provider uses an `o4` prefix.

## Candidate 2: `fix(langchain)`: normalize the provider prefix in `init_chat_model` the same way as `model_provider`

**Where:** `libs/langchain_v1/langchain/chat_models/base.py:623-651` (`_parse_model`).
- The prefix branch checks `model.split(":", maxsplit=1)[0] in _BUILTIN_PROVIDERS` **before** any normalization (`base.py:626-628`).
- Normalization (`model_provider.replace("-", "_").lower()`, `base.py:650`) happens only afterward. It therefore applies to the `model_provider=` kwarg and to inferred providers, but never to a prefix.

`init_embeddings`, the sibling function, lowercases its prefix (`embeddings/base.py:141-142`, `provider.lower().strip()`). So `"OpenAI:text-embedding-3-small"` works there.

**Repro (not executed):**
```python
init_chat_model("gemini-3-pro", model_provider="google-genai")  # works (normalized)
init_chat_model("google-genai:gemini-3-pro")  # ValueError: Unable to infer model provider
init_chat_model("OpenAI:gpt-5.5")             # ValueError (embeddings accepts "OpenAI:...")
```
In the failing cases the whole string, prefix included, goes to inference and matches nothing. If inference did match something, such as `"Claude:..."` -> `anthropic`, the model name would keep the bogus prefix.

**Reference PR:** #40837 added the underscore-heavy keys `bedrock_mantle_openai` and `bedrock_mantle_anthropic`. Users are likely to type these with hyphens, as in `bedrock-mantle-openai:...`. The kwarg form accepts the hyphens and the prefix form rejects them.

**Fix:** compute `candidate = model.split(":", 1)[0].replace("-", "_").lower()` and test `candidate in _BUILTIN_PROVIDERS`. Keep the rest of `_parse_model` as it is.

**Test:** parametrize `_parse_model` with `("google-genai:gemini-x", None) -> ("gemini-x", "google_genai")`, `("OpenAI:gpt-5.5", None) -> ("gpt-5.5", "openai")` and `("bedrock-mantle-openai:m", None)`. Add a negative case: `"foo:bar"` with an unknown prefix still goes to inference unchanged.

**Optional mirror:** `init_embeddings(..., provider="OpenAI")` is **not** lowercased (`embeddings/base.py:166-181`), even though the prefix is. The same issue or PR could normalize both paths.

**Risk / pushback:** low. Today these inputs raise `ValueError`, so accepting them breaks nothing.

## Candidate 3: `docs(langchain)`: `_BUILTIN_PROVIDERS` notes say unregistered providers still work, but they raise

**Where:**
- `libs/langchain_v1/langchain/chat_models/base.py:111-121` says: "If a provider is not in this dict, it can still be used with `init_chat_model` as long as its integration package is installed ... must be specified explicitly via the `model_provider` parameter."
- `_get_chat_model_creator` (`base.py:173-176`) raises `ValueError("Unsupported provider=...")` for any provider that is not in the dict.
- `libs/langchain_v1/langchain/embeddings/base.py:36-55` repeats the note word for word. It even mentions `init_chat_model` and `model_provider`, but the embeddings function is `init_embeddings` and its parameter is `provider`. `_get_embeddings_class_creator` and `_infer_model_and_provider` (`embeddings/base.py:76`, `181`) also raise.

**Repro (not executed):**
```python
init_chat_model("my-model", model_provider="together")  # ValueError: Unsupported provider='together'
init_embeddings("x", provider="voyageai")                 # ValueError: Provider 'voyageai' is not supported.
```

**Reference PR:** #40837 updated this same registry and its docs. #40794 (AGENTS.md doc accuracy) is a precedent for doc-accuracy fixes.

**Fix:** reword both notes. Say that unlisted providers are not accepted by `init_chat_model` or `init_embeddings` and should be imported directly from their integration package. In the embeddings copy, fix the function and parameter names.

**Test:** none needed for the docs. An optional regression test can assert that `ValueError` is raised for an unlisted provider in each module; the chat one may already exist.

**Risk / pushback:** docs-only, so it is easy to merge but has low impact.

## Rejected during the search
- **Sync/async parity in `ModelFallbackMiddleware`, `ModelRetryMiddleware`, `SummarizationMiddleware` (including the #41159 overflow retry), `HumanInTheLoopMiddleware` (`per_call` #41062), `TodoListMiddleware`, `PIIMiddleware`, `ModelCallLimitMiddleware` and `ToolCallLimitMiddleware`:** the pairs are identical, or the async hook delegates to the sync one.
- **Swallowed `GraphBubbleUp` / HITL `per_call` interrupt in tool retry, tool error or model retry middleware:** all of them re-raise `GraphBubbleUp`.
- **`ModelFallbackMiddleware` cache sanitizing (#40886) for Bedrock Mantle classes (`_llm_type` missing from `_ANTHROPIC_LLM_TYPES` or from the `prompt_cache_key` set):** this may be real, but I could not check it. `langchain-aws` is not checked out, so the classes' `_llm_type` values are unknown.
- **PII redaction of tool-call args:** this is already handled (`pii.py:225-240`).
