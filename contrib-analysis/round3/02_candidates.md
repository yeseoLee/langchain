# Round 3 candidates (consolidated)

- Sources: `find_core.md`, `find_partners.md`, `find_v1_splitters.md`, `find_open_issues.md` (details there).
- Duplicate search was done during discovery (GitHub search via REST API / web). Nothing was executed.
- Needed: 5 more passes (5/10 passed so far).

| ID | Title (draft) | Type | Duplicate search | Source |
|---|---|---|---|---|
| R3-1 | fix(core): `ChatPromptTemplate.pretty_repr` raises `NotImplementedError` for dict content blocks (`DictPromptTemplate.pretty_repr`) | bug | 0 hits | find_core #1 |
| R3-2 | docs(core): `render_text_description(_and_args)` docstrings show wrong format; args rendered as Python repr | docs/bug | 0 hits | find_core #2 |
| R3-3 | fix(core): `DictPromptTemplate.input_variables` order is nondeterministic (`list(set())`) | bug | 0 hits | find_core #3 |
| R3-4 | fix(qdrant): set `Document.id` on documents returned by `QdrantVectorStore` | bug | 0 hits | find_partners #1 |
| R3-5 | fix(chroma): `hybrid_search` fails on records without metadata | bug (unverified) | 0 hits | find_partners #2 |
| R3-6 | fix(xai): `stop` stripping ignores call-time `model` override | bug | only #38335 (origin) | find_partners #3 |
| R3-7 | fix(langchain): `ClearToolUsesEdit.keep` counts excluded/already-cleared results | bug (behavior change) | 0 hits | find_v1 #1 |
| R3-8 | fix(langchain): `ModelCallLimitMiddleware` validate `run_limit <= thread_limit` and non-negative limits (parity with `ToolCallLimitMiddleware`) | bug | 0 hits | find_v1 #2+#3 |
| R3-9 | existing #41601: `convert_to_messages` JSONDecodeError on empty tool-call args | existing issue | 2 people asking for assignment | find_open_issues #1 |
| R3-10 | existing #41623: google_genai translator drops `extras` on image blocks | existing issue | 2 people asking for assignment | find_open_issues #2 |
| R3-11 | existing #41603: mistralai drops content when tool calls are present | existing issue | 1 person asking, closed PR exists | find_open_issues #3 |
