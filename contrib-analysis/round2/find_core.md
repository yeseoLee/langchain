# Round 2: new contribution candidates in `libs/core/langchain_core`

Excluded (already covered elsewhere): Mermaid edge-label escaping, translator index-pop mutation, RemoveMessage mapping.
Nothing below was executed; every claim comes from reading the code at the current HEAD of `contrib-analysis`.

---

## 1. `RunnableWithFallbacks` with `exception_key` changes the caller's input dict (strong)

**Pattern sibling of:** #41502 (`merge_content` changed caller-owned content) and #40213 (translator changed message content).

**Where:** `libs/core/langchain_core/runnables/fallbacks.py`
- `invoke`: line 189, `input[self.exception_key] = last_error`
- `ainvoke`: line 243
- `batch`: around line 340, `input_[self.exception_key] = output`. `abatch` has the same line.
- `stream`: line 492
- `astream`: line 556

All of these write into the dict object the caller passed in. Nothing is copied first.

**Repro (not run):**
```python
from langchain_core.runnables import RunnableLambda

def boom(_: dict) -> str:
    raise ValueError("primary failed")

def ok(d: dict) -> str:
    return f"fallback saw keys {sorted(d)}"

chain = RunnableLambda(boom).with_fallbacks([RunnableLambda(ok)], exception_key="exception")
payload = {"question": "hi"}
chain.invoke(payload)
assert payload == {"question": "hi"}  # fails: payload now has an "exception" key holding the ValueError
```
Effects you can see:
- If the caller reuses the dict (a retry loop, or a batch of shared inputs), the *primary* runnable gets a stale `exception` key on the next call.
- A prompt template that then gets an unexpected exception object may end up formatting the old error into the prompt.
- Tracing input and output can also look changed after the fact.

**Fix:** before the loop, when `exception_key` is set, do `input = {**input}` (and `input_ = {**input_}` per item in `batch`/`abatch`; `run_again` can hold the copies). It is a shallow copy, which matches how the key is added.

**Test:** in `tests/unit_tests/runnables/test_fallbacks.py`, add a parametrized test over `invoke`/`ainvoke`/`batch`/`abatch`/`stream`/`astream`. It should assert that the original dict is unchanged and that the fallback still receives `exception`. The existing `test_invoke_with_exception_key` setup can be reused.

---

## 2. `RunnableWithFallbacks.invoke`/`ainvoke` pass the parent `config` instead of `child_config` (medium: needs a tracer test to confirm)

**Pattern:** sync/async and method-to-method parity gap.

**Where:** `libs/core/langchain_core/runnables/fallbacks.py`
- `invoke` (lines 190-197): `child_config = patch_config(config, callbacks=run_manager.get_child())` is built, but the call is `context.run(runnable.invoke, input, config, **kwargs)`.
- `ainvoke` (line 246): `context.run(runnable.ainvoke, input, config, **kwargs)`.
- Compare `astream` (line ~559), which passes `child_config`, and `stream` (line ~495), which passes no config and so falls back to the context var. `batch` also uses a patched child config per item.

**Why it matters:** `ensure_config` (`runnables/config.py:271-291`) applies the context-var config first. It then lets an explicit config's non-None keys override it. When the fallbacks runnable sits inside a chain, `config["callbacks"]` is the parent chain's child manager. That overrides the `get_child()` manager set through `set_config_context`. The primary and fallback attempts therefore get the *parent chain* as `parent_run_id` instead of the `RunnableWithFallbacks` run, while `stream`/`astream`/`batch` nest them correctly. At top level (callbacks=None) the context var wins and the bug is hidden. That would explain why existing tests pass.

**Repro idea (not run):** `chain = RunnableLambda(lambda x: x) | RunnableLambda(boom).with_fallbacks([RunnableLambda(ok)])`. Invoke it with a `FakeTracer` / `RunCollectorCallbackHandler`. Find the `RunnableWithFallbacks` run and assert it has child runs. Expected to fail for `invoke`/`ainvoke` and pass for `stream`/`batch`.

**Fix:** pass `child_config` in both calls. This is a one-token change twice.

**Before filing:** write the tracer test first and confirm it fails. If it passes, drop this candidate.

---

## 3. Docstring examples that still don't run as copied (strong; direct follow-up to #40815)

#40815 fixed import paths in `runnables/fallbacks.py`, `few_shot.py`, `retry.py`, and other files, but missed these. Each one was checked against the module's `__init__`/`__all__`:

| File:line | Problem | Fix |
|---|---|---|
| `runnables/fallbacks.py:70` | `from langchain_core.output_parser import StrOutputParser` (module does not exist) | `langchain_core.output_parsers` |
| `callbacks/manager.py:2640`, `:2672`, `:2763` | `from langchain_core.runnable import RunnableLambda` | `langchain_core.runnables` |
| `prompts/few_shot.py:330` | `SemanticSimilarityExampleSelector` imported from `langchain_core.prompts` (not exported there) | `langchain_core.example_selectors` |
| `prompts/few_shot.py:331` | `from langchain_core.embeddings import OpenAIEmbeddings` | `from langchain_openai import OpenAIEmbeddings` |
| `prompts/few_shot.py:332` | `from langchain_core.vectorstores import Chroma` | `from langchain_chroma import Chroma` (or use `InMemoryVectorStore` from core) |
| `prompts/few_shot.py:346` | `from langchain_core import SystemMessage` | `from langchain_core.messages import SystemMessage` |
| `prompts/few_shot.py:359` | `AIMessagePromptTemplate` used but never imported in that example | add it to the `langchain_core.prompts` import |
| `prompts/structured.py:109` | `from langchain_core.prompts import StructuredPrompt` (`prompts/__init__.py` does not export it) | `from langchain_core.prompts.structured import StructuredPrompt` |
| `documents/base.py:70`, `:88`, `:100` | `from langchain_core.documents import Blob` (`documents/__init__.py` exports only `Document`, `BaseDocumentCompressor`, `BaseDocumentTransformer`) | `from langchain_core.documents.base import Blob` (or `langchain_core.document_loaders`, if exported there; check) |

The scan used here: every indented `from langchain_core... import ...` line in docstrings, checked against the target module. Re-run it before filing in case other examples were missed.

**Test:** docs-only. As in #40815, no unit test is needed. Optionally add a small test that parses docstring import lines for `langchain_core.*` and asserts that each name resolves, which would stop this from coming back. Mention it as a follow-up rather than putting it in the same PR.

---

## 4. `load_prompt_from_config` changes the caller's config dict (real but weak: the API is deprecated)

**Where:** `libs/core/langchain_core/prompts/loading.py`
- line 76: `config.pop("_type", ...)`
- `_load_template`: `config.pop(f"{var_name}_path")`, then `config[var_name] = template`
- `_load_examples`: `config["examples"] = ...`
- `_load_output_parser`: `config["output_parser"] = ...`
- `_load_few_shot_prompt`: `config.pop("example_prompt_path")`, and `config["example_prompt"] = load_prompt_from_config(config["example_prompt"])`. This one also changes the nested dict.

**Repro:**
```python
cfg = {"_type": "few_shot", "input_variables": [...], "prefix": "...", "suffix": "...",
       "example_prompt": {"_type": "prompt", "input_variables": ["a"], "template": "{a}"}, "examples": [...]}
load_prompt_from_config(cfg)
load_prompt_from_config(cfg)  # "_type" is gone: it now defaults to "prompt" and fails or builds the wrong type
```
**Fix:** `config = copy.deepcopy(config)` at the top of `load_prompt_from_config`.
**Test:** call it twice on the same dict and assert equal results, and that the dict is unchanged.
**Caveat:** the function has been `@deprecated` since 1.2.21 (removal in 2.0). Maintainers may not take fixes to it. File it only as a low-priority "while we're here" issue.

---

## Rejected after checking
- `_create_message_from_message_type` / `_convert_to_message` (`messages/utils.py:640-681`): `.pop` runs on a fresh `**kwargs` dict, and `message.copy()` comes first. No caller mutation, only aliasing of nested `response_metadata`/`tool_calls` dicts. Too weak.
- `BasePromptTemplate.invoke` vs `ainvoke` (`prompts/base.py:226-256`): one uses `{**…}` and `+=`, the other `.update`/`.extend`. This looks like a parity gap, but `ensure_config` copies `metadata`/`tags`, so neither leaks. It is a style difference only.
- `convert_to_openai_function(strict=True)` mutation: it already does `deepcopy` for dict input, and the BaseTool/pydantic paths go through `dereference_refs`, which builds new dicts.
- `_patch_json_schema_cache` (`tools/base.py:404-430`) returns the same cached dict on every call. This is a latent hazard if a consumer changes `tool.tool_call_schema.model_json_schema()`, but no current caller in core or the partners does (`count_tokens_approximately` copies it first). There is nothing to point at in a repro.
- `_explode_reasoning` in the OpenAI translator changes `block`, but the caller passes `raw_block.copy()`, so that is safe. The other translators build new dicts.
- `merge_dicts`/`merge_lists` (`utils/_merge.py`): both copy `left`. The values they append are aliased but are not changed later.
