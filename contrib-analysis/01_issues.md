# LangChain: 3 issue candidates ready to file

- Analysis date: 2026-11-03
- Target: `langchain-ai/langchain` master `2775506`
- Tools: codegraph (37,845 nodes / 97,297 edges), git history
- Constraint: the GitHub API (`gh`) cannot reach the upstream repo, so **open issues and PRs for duplicates / work already in progress have not been checked**. Search before filing.

## 0. Project landscape

- **Workflow is strictly enforced**
  - `require_issue_link.yml`: if an external PR has no `Fixes #N` or the author is not the issue's assignee, the PR is closed automatically.
  - `reopen_on_assignment.yml`: a closed PR reopens once the author is assigned to the issue.
  - So the order is fixed: **file the issue → get assigned by a maintainer → open the PR**.
- **PR rules**
  - Title format is `type(scope): description`.
  - A release note is required for behavior changes.
  - `make format lint test` must pass.
  - English only.
  - A long, obviously AI-generated description gets the PR closed.
- **Activity level**
  - 64 commits from 10/24 to 11/02. Mostly maintainers (ccurme, mdrxy) and bots.
  - External merges in the last ~1 week: #41502, #41512, #41642 (core), plus #40213 (merged 11/02, long-open), all small bug fixes, each with tests.
  - The successful pattern: **one minimal fix + a regression test + a reference to the merged PR it follows from**.
- **Pollution**
  - Several merged PRs say "written with help from an AI agent". AI use is tolerated, but the PR template warns that low-quality AI text is closed.
- **Issue tracking**
  - Done on GitHub Issues; no external tracker.
  - `close_unchecked_issues.yml` closes issues whose template checkboxes are unchecked.

---

## Issue A. `fix(core)`: Mermaid edge labels are not escaped

**Background (merged PR to reference):** #41512 added `_escape_mermaid_label()` to `graph_mermaid.py` and applied it **only to node metadata**.

**Problem**
- `libs/core/langchain_core/runnables/graph_mermaid.py:230-251`: `edge.data` is inserted as-is into `-- &nbsp;{edge_data}&nbsp; -->`.
- If a LangGraph conditional edge's `path_map` key, or a user-supplied edge label, contains characters such as `( ) [ ] { } | " < >` or `-->`, Mermaid parsing fails.
  - Example: label `"route (fallback)"` or `"a|b"`.
- Node names (`node_name`, line 175) are also inserted without escaping. This could be handled together, but keep the scope minimal.

**Proposed fix**
- Before splitting into words, apply `edge_data = _escape_mermaid_label(str(edge.data))` (a few lines).
- This also fits the "refactoring" category: applying a reusable function that was just committed to code it does not cover yet.

**Tests**
- Add to `tests/unit_tests/runnables/test_graph.py`: an edge with `Edge(..., data="a (b) | c", conditional=True)`, asserting the output contains `#40;`, `#124;` and so on.
- Confirm the existing snapshots (labels without special characters) are unchanged.

**codegraph impact**
- Callers of `draw_mermaid`: `Graph.draw_mermaid`, then `draw_mermaid_png`, plus LangGraph's `get_graph().draw_mermaid()`.
- The output changes only for labels containing special characters.

**Unverified:** how real LangGraph `path_map` labels render on mermaid.ink (not tested, since running code and the network were blocked).

---

## Issue B. `fix(mistralai)`: `on_llm_new_token` receives a list token

**Background (merged PR to reference):** #41643 (anthropic, 11/02) changed the callback token to `chunk.text`, because list content was being passed incorrectly or not reported at all.

**Problem**
- `libs/partners/mistralai/langchain_mistralai/chat_models.py:899-902` (`_stream`) and `:929-932` (`_astream`):
  ```python
  run_manager.on_llm_new_token(token=cast("str", new_chunk.content), chunk=gen_chunk)
  ```
- `_convert_chunk_to_message_chunk` (around line 359) turns content into **a list of blocks** when `output_version == "v1"` or a `thinking` block (Magistral) is present.
  - The `cast` is only a type hint; at runtime a list is passed.
- Impact:
  - Callbacks that assume `token: str`, such as `StreamingStdOutCallbackHandler`, can raise `TypeError` or print wrong output.
  - LangSmith token events are wrong.
- Comparison (grep across partners):
  - fireworks, groq, ollama, perplexity and openrouter all already pass `chunk.text`.
  - **Mistral is the only remaining case.**

**Proposed fix**
- In both places, change `token=cast("str", new_chunk.content)` to `token=gen_chunk.text` (2 lines).

**Tests**
- In `libs/partners/mistralai/tests/unit_tests/test_chat_models.py`, with a mocked stream and `output_version="v1"`, assert that every token in a callback handler is a `str`.
- Follow the style of the test added in #41643.

---

## Issue C. `docs(standard-tests)`: legacy `python.langchain.com` links in docstrings

**Background:** the docs site has moved to `docs.langchain.com/oss/python/...` (the PR template also links there).

**Problem (excluding `langchain_classic`)**
- `libs/standard-tests/langchain_tests/integration_tests/chat_models.py:1612`: `https://python.langchain.com/docs/concepts/chat_models/#standard-parameters`
- Same file, `:2618`: `https://python.langchain.com/api_reference/_modules/langchain_openai/...` (legacy API reference source page).
- These docstrings are read by authors of new integration packages, so stale links create confusion.
- The 7 occurrences in `langchain_classic` are excluded: it is a legacy package, so the fix is low-value and easily rejected.

**Proposed fix**
- Replace with the corresponding `docs.langchain.com` page, or with a GitHub source link to `langchain_openai.chat_models.base` at a fixed commit.

**Unverified:** whether the old URLs are actually broken or redirect (the proxy blocked access to `python.langchain.com`). Check in a browser before filing.
