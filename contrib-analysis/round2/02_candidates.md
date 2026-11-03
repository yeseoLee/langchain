# Round 2 candidates (consolidated, deduplicated)

- Sources: `find_core.md`, `find_v1.md`, `find_partners.md`, `find_docs_tests.md` (details, file:line and repros are there).
- Nothing was executed. Verified by reading the code; the main lead verified directly is marked ✔.
- Needed: 8 more to pass (total goal 10, 2 passed in R1).

| ID | Title (draft) | Type | Reference PR | Source |
|---|---|---|---|---|
| R2-1 | fix(core): `RunnableWithFallbacks` with `exception_key` mutates the caller's input dict ✔ | bug | #41502, #40213 | find_core #1 |
| R2-2 | fix(core): `RunnableWithFallbacks.invoke/ainvoke` pass parent `config` instead of `child_config` ✔ (tracing effect unconfirmed) | bug | — (stream/batch already use child_config) | find_core #2 |
| R2-3 | docs(core): docstring examples importing nonexistent modules (follow-up to #40815) | docs | #40815 | find_core #3 + find_docs_tests #2 |
| R2-4 | docs(anthropic): middleware docstring examples import nonexistent classes ✔ (also file_search.py) | docs | #40815 | find_docs_tests #1 |
| R2-5 | fix(huggingface): handle null token counts in usage metadata | bug | #41553 | find_partners #1 |
| R2-6 | fix(perplexity): handle null token counts in `_create_usage_metadata` | bug | #41553 | find_partners #2 |
| R2-7 | fix(mistralai): handle null token counts | bug | #41553 | find_partners #3 |
| R2-8 | fix(huggingface): `_astream`/`_agenerate` diverge from sync for non-endpoint LLMs | bug | — | find_partners #4 |
| R2-9 | fix(langchain): normalize `provider:` prefix in `init_chat_model` (hyphen/case) ✔ | bug | #40837 | find_v1 #1 |
| R2-10 | fix(langchain): infer `openai` for `o4-*` model names ✔ | bug | #40837, #40844 | find_v1 #2 |
| R2-11 | docs(langchain): provider-table note claims unregistered providers work, but they raise | docs | #40794 | find_v1 #3 |
| R2-12 | test(text-splitters): cover `split_text_on_tokens` overlap validation and empty input | test | — | find_docs_tests #3 |
| R2-13 | fix(core): `load_prompt_from_config` mutates caller config (deprecated API) | bug | #41502 | find_core #4 |
| R2-14 | style(core): replace Sphinx double backticks per AGENTS.md | docs | — | find_docs_tests |
