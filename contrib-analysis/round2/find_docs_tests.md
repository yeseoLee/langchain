# Round 2: docs / test-gap / small-bug candidates

All findings verified by reading source (no code executed). Duplicate-PR check via `gh` was not possible (GraphQL is blocked in this session); search open PRs manually before starting.

## Candidate A (strongest): docs(anthropic): fix middleware docstring examples importing nonexistent classes

- **Files:** `libs/partners/anthropic/langchain_anthropic/middleware/anthropic_tools.py`, lines 608/613, 649/654, 1099/1104, 1148/1153
- **Evidence:** the four docstring examples run
  `from langchain.agents.middleware import StateTextEditorToolMiddleware` (and `StateMemoryToolMiddleware`, `FilesystemTextEditorToolMiddleware`, `FilesystemMemoryToolMiddleware`). None of these names exist anywhere in `libs/langchain_v1/langchain/agents/middleware/`. The real classes are `StateClaudeTextEditorMiddleware` (L599), `StateClaudeMemoryMiddleware` (L638), `FilesystemClaudeTextEditorMiddleware` (L1090) and `FilesystemClaudeMemoryMiddleware` (L1136). They are exported from `langchain_anthropic.middleware` (`__init__.py` L4-7). Copying any of the examples fails with `ImportError`. The `root_path=` kwarg in the examples is correct (L692).
- **Precedent:** #40815 (`40fe8d6`, `docs(core): fix docstring examples that don't run as copied`)
- **Fix:** `from langchain_anthropic.middleware import StateClaudeTextEditorMiddleware` and so on, and use the matching class name in `middleware=[...]`. That is 8 lines changed.
- **Test idea:** none needed for a docs-only change. Optionally, a unit test can assert that each example's imported name is exported from `langchain_anthropic.middleware`.

## Candidate B: docs(core): fix remaining docstring imports from nonexistent modules (missed by #40815)

- **Evidence (all modules or names that do not exist):**
  - `libs/core/langchain_core/callbacks/manager.py:2640`, `:2672`, `:2763`: `from langchain_core.runnable import RunnableLambda` (the module is `langchain_core.runnables`). These are in the `adispatch_custom_event` / `dispatch_custom_event` examples.
  - `libs/core/langchain_core/runnables/fallbacks.py:70`: `from langchain_core.output_parser import StrOutputParser` (should be `langchain_core.output_parsers`). #40815 edited this file and missed this line.
  - `libs/core/langchain_core/prompts/few_shot.py:330-332`: `SemanticSimilarityExampleSelector` imported from `langchain_core.prompts` (it lives in `langchain_core.example_selectors`), `OpenAIEmbeddings` from `langchain_core.embeddings` (it lives in `langchain_openai`), and `Chroma` from `langchain_core.vectorstores` (it lives in `langchain_chroma`). #40815 edited this file too.
- **Fix:** correct the import paths (about 7 lines). This is a direct follow-up to #40815, so frame it that way.
- **Note:** this is in core, not langchain_v1/partners, but it is the same class of error and low risk. It can be merged with Candidate A or kept as its own PR.

## Candidate C: test(text-splitters): cover `split_text_on_tokens` overlap validation and empty input

- **File:** `libs/text-splitters/langchain_text_splitters/base.py:511-513`
- **Evidence:** `split_text_on_tokens` raises `ValueError("tokens_per_chunk must be greater than chunk_overlap")` when `tokens_per_chunk <= chunk_overlap`. A grep for that message in `tests/` finds nothing, so the branch is untested. The existing tests (`test_split_text_on_tokens`, `test_decode_returns_no_chunks`, test_text_splitters.py ~3097-3124) only cover the happy path. The empty-text path (`encode("") == []` should return `[]`) is also untested.
- **Related inconsistency (document, don't change):** `TextSplitter.__init__` (base.py:94) allows `chunk_overlap == chunk_size`, but the token path rejects equality, and only lazily at split time. The inconsistency guards a real infinite-loop case (a step of 0), so tests should pin the current behavior rather than change it.
- **Test ideas:**
  1. Parametrize over `(chunk_overlap, tokens_per_chunk)` in `[(7, 7), (8, 7)]` and assert `pytest.raises(ValueError, match="tokens_per_chunk must be greater")`.
  2. `split_text_on_tokens(text="", tokenizer=...)` returns `[]`.
  3. Input shorter than one chunk returns a single chunk.
- **Fix:** tests only. No source change.

## Assessed and deprioritized: Sphinx double-backtick cleanup

- Exact count (excluding ``` code fences): `core/langchain_core/utils/_gateway.py` 19, `core/langchain_core/load/serializable.py` 5, `partners/anthropic/langchain_anthropic/chat_models.py` 4 (around L522-523 and L1716-1717), `partners/fireworks/langchain_fireworks/chat_models.py` 2. That is 30 in total, and these are the only ones across core, v1 and partners.
- This is a valid AGENTS.md conformance fix and is mechanical. But it has no user-visible impact: `_gateway.py` is private and was recently added (#39115, #39703), so maintainers may see it as churn. Do it only as a single tiny `docs(core)` PR (core files only, to keep one scope), or fold it into Candidate B.

## Rejected
- langchain_v1 middleware docstring examples (`ModelRetryMiddleware`, `ToolRetryMiddleware`, and others): the kwargs and imports match their signatures and exports.
- A partner top-level import scan (`from langchain_<partner> import X`): every name resolves.
