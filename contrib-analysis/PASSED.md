# Contribution points that passed the rebuttal review: final 10

- Criterion: 5-perspective rebuttal (maintainer, technical, duplicates, impact, process), average >= 5.5, no fatal rebuttal left unresolved.
- Duplicate check: GitHub search (REST search API + web). Keyword-based, so re-check with your own search right before filing.
- Workflow: file the issue (template checklist fully filled in) -> get assigned by a maintainer -> PR (`Fixes #N`, `type(scope):`, 1-2 sentence body, AI-use note, no line numbers).
- Nothing was executed. Reproduce locally before filing.

| Rank | Round | Title (draft) | Avg | Type | Reference PR | Conditions / caveats |
|---|---|---|---|---|---|---|
| 1 | R3 | fix(core): `DictPromptTemplate.pretty_repr` raises `NotImplementedError` in `ChatPromptTemplate.pretty_print` | 7.4 | bug | #34339 (style) | Can also fix `input_variables` order (`list(set())`) |
| 2 | R3 | fix(qdrant): set `Document.id` on documents returned by `QdrantVectorStore` | 7.4 | bug | `get_by_ids` contract, Chroma | Keep `metadata["_id"]`; repro possible with in-memory Qdrant |
| 3 | R4 | fix(langchain): `LLMToolEmulator` should use `response.text` | 7.2 | bug | SummarizationMiddleware | Frame as "non-text blocks leak into the tool result", release note, check conflict with #41579 |
| 4 | R1 | fix(core): escape Mermaid edge labels | 6.4 | bug/refactor | #41512 | Escape before wrapping, `#` regression test, real parse-failure repro; exclude node names (#39816) |
| 5 | R4 | fix(ollama): `ChatOllama._get_ls_params` sets `ls_max_tokens` | 6.4 | bug | `OllamaLLM._get_ls_params` | Only positive `num_predict`; check params location |
| 6 | R2 | docs(anthropic): fix middleware docstring examples importing nonexistent classes | 6.2 | docs | #40815 | Fix 4 places in anthropic_tools.py + file_search.py |
| 7 | R2 | fix(langchain): normalize `provider:` prefix in `init_chat_model` (hyphen/case) | 6.2 | bug | #40837 | `init_embeddings` already lowercases |
| 8 | R4 | fix(groq): report the response's `model` as `model_name` | 6.2 | bug | openai/openrouter behavior | Do fireworks separately; watch for conflicts with #38029, #38759 |
| 9 | R2 | fix(huggingface): handle null token counts in usage metadata | 6.0 | bug | #41553, #41526 | Need evidence that the API/TGI returns null |
| 10 | R3 | fix(langchain): validate limits in `ModelCallLimitMiddleware` | 6.0 | bug | #34576 | Parity with ToolCallLimitMiddleware; note the new ValueError |

## Reserve (passed but below the cutoff)
- R3-6 fix(xai): `stop` stripping honors call-time `model` (5.6, follow-up to #38335)
- R1-B fix(mistralai): pass `chunk.text` to callback (6.2 -> demoted: core `stream()` path does not use this callback and itself passes a list token)

## Main rejection reasons (see each round's 03_evaluation.md)
- Already reported / rejected by a maintainer: fallbacks exception_key (#40878), o4 inference (#38243), core docstring imports (#41064), fallbacks config (#36072)
- Not a bug: ClearToolUsesEdit.keep (matches the docs), ChatOllama chunk= (core sends it)
- Low value: legacy links, double backticks, text-splitter tests, deprecated APIs
- Crowded existing issues: #41601, #41623, #41603

## Recommended filing order
1 -> 2 -> 3 first (high score, easy to test). One at a time; move on after seeing the response.
