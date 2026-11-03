# Contribution points that passed the rebuttal review (cumulative)

Criterion: survived the 5-perspective rebuttal (maintainer, technical, duplicates, impact, process), average >= 5.5, with no fatal rebuttal left unresolved.
Goal: 10. Duplicate check: keyword search via the GitHub web UI (an exhaustive search was not possible).

| # | Round | Title (draft) | Avg | Notes / conditions |
|---|---|---|---|---|
| 1 | R1 | fix(core): escape Mermaid edge labels | 6.4 | Escape before wrapping, `#` regression test, actual parse-failure repro. Ref #41512 |
| 2 | R1 | fix(mistralai): pass `chunk.text` to `on_llm_new_token` like other partners | 6.2 | Drop the crash claim; reframe as consistency. Ref #34739, #41643. Ask a maintainer first |

| 3 | R2 | docs(anthropic): fix middleware docstring examples importing nonexistent classes | 6.2 | Also fix file_search.py. Ref #40815 |
| 4 | R2 | fix(huggingface): handle null token counts in usage metadata | 6.0 | Need evidence of null responses. Ref #41553, #41526 |
| 5 | R2 | fix(langchain): normalize `provider:` prefix in `init_chat_model` | 6.2 | Same as init_embeddings. Ref #40837 |
| 6 | R3 | fix(core): `DictPromptTemplate.pretty_repr` raises `NotImplementedError` in `ChatPromptTemplate.pretty_print` | 7.4 | Can also fix input_variables order (R3-3) |
| 7 | R3 | fix(qdrant): set `Document.id` on returned documents | 7.4 | Keep `metadata["_id"]` (compatibility) |
| 8 | R3 | fix(langchain): validate limits in `ModelCallLimitMiddleware` | 6.0 | Parity with #34576, note the new ValueError |
| 9 | R3 | fix(xai): `stop` stripping honors call-time `model` | 5.6 | Follow-up to #38335 (borderline) |

Duplicate search (GitHub web search): no hits for R1-A, R1-B, R2-4, R2-5, R2-9 (R1-A's node-name part overlaps #39816, so scope it to edge labels only)

Rejected: R1-C (stale python.langchain.com links in standard-tests, 3.0)
