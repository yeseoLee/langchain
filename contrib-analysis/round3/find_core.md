# Round 3: new contribution candidates in `langchain_core`

Duplicate check method: GitHub's GraphQL search (and so `gh search`) is blocked in this session, so I used the REST search API (`gh api search/issues`, `is:issue` and `is:pull-request`, open and closed). The same queries in the web UI are listed under each item as `https://github.com/langchain-ai/langchain/issues?q=...`. Nothing was executed (no Python, uv or pytest), so every repro below is unverified.

Searched and dropped because they are already covered:
- `InMemoryRateLimiter` hangs on a non-positive config: #41453, #41456, #41458 (open), and PRs #41454, #41482, #41492, #41594.
- `parse_json_markdown` fence and trailing-text handling: #40297, #41607 (open), and PRs #41641, #40690.
- `filter_messages(exclude_tool_calls=...)` content blocks and invalid calls: #41290, #36492, and PRs #39359, #36563, #41293.
- `sanitize_for_postgres` docstring NUL escapes: #41177, PR #41345.
- `DictPromptTemplate` async path and list values: #39835, #39152.
- `XMLOutputParser` with list content: #39744, PR #39735.

---

## 1. `ChatPromptTemplate.pretty_print()` crashes when a message holds a dict (non-text/non-image) content block

- **Where:** `libs/core/langchain_core/prompts/dict.py:113`. `DictPromptTemplate.pretty_repr` is a bare `raise NotImplementedError`. It is reached from `_StringImageMessagePromptTemplate.pretty_repr` (`libs/core/langchain_core/prompts/chat.py:651-664`), which calls `prompt.pretty_repr(html=html)` on every sub-prompt. `from_template` builds a `DictPromptTemplate` for any dict block (`chat.py:516-528`).
- **Repro (not executed):**
  ```python
  from langchain_core.prompts import ChatPromptTemplate
  p = ChatPromptTemplate.from_messages([
      ("human", [
          {"type": "text", "text": "Summarize {topic}"},
          {"type": "file", "base64": "{data}", "mime_type": "application/pdf"},
      ])
  ])
  p.pretty_print()  # NotImplementedError
  ```
- **Why it's a bug:** `pretty_print` and `pretty_repr` are public debugging helpers on every `ChatPromptTemplate`. Multimodal file/audio blocks are a documented, supported input, and they make the whole prompt impossible to print. String and image sub-prompts already implement `pretty_repr`.
- **Dup search:** 0 issues and 0 PRs for "DictPromptTemplate pretty_repr", "pretty_repr NotImplementedError" and "pretty_print NotImplementedError".
  - https://github.com/langchain-ai/langchain/issues?q=DictPromptTemplate+pretty_repr
  - https://github.com/langchain-ai/langchain/issues?q=pretty_repr+NotImplementedError
  - https://github.com/langchain-ai/langchain/issues?q=pretty_print+NotImplementedError
- **Reference PR (style):** #34339 / #36372 (message `pretty_repr` for content blocks), both closed.
- **Fix:** implement `pretty_repr` and keep the keyword-only `html` signature. For example, return `json.dumps(self.template, indent=2, ensure_ascii=False)`, and when `html=True` wrap each `{var}` / `{{var}}` in `get_colored_text(..., "yellow")` or bold it, as `PromptTemplate.pretty_repr` does.
- **Test:** in `tests/unit_tests/prompts/test_dict.py`, check that `DictPromptTemplate(...).pretty_repr()` contains the keys and the variable placeholders. In `test_chat.py`, check that `pretty_repr()` on a `ChatPromptTemplate` with a `file` block does not raise and contains "Human Message".

## 2. `render_text_description_and_args` emits Python reprs, not the JSON its docstring promises (and both docstrings show the wrong format)

- **Where:** `libs/core/langchain_core/tools/render.py:41` (`args_schema = str(tool.args)`), plus the docstrings of `render_text_description` (line 13) and `render_text_description_and_args`.
- **Repro (not executed):**
  ```python
  from langchain_core.tools import tool
  from langchain_core.tools.render import render_text_description_and_args

  @tool
  def search(query: str, exact: bool = False) -> str:
      """Search the web."""
      return ""

  print(render_text_description_and_args([search]))
  # search(query: str, exact: bool = False) -> str - Search the web., args: {'query': {'title': 'Query', 'type': 'string'}, 'exact': {'default': False, ...}}
  ```
  The output uses single quotes and Python `False`/`None`. The docstring shows `search: This tool is used for search, args: {"query": {"type": "string"}}`, with a `:` separator and JSON.
- **Why it's a bug:** this text goes into LLM prompts (ReAct / structured-chat agents) as a JSON schema description. Python literals (`False`, `None`, single quotes) are not JSON, so models are pushed toward copying invalid JSON into their tool-input JSON. The docstrings also describe a `name: description` format, but the code produces `name - description` (or `name(sig) - description`).
- **Dup search:** 0 PRs. The issue hits are unrelated agent tickets (#12645, #27534 and others).
  - https://github.com/langchain-ai/langchain/issues?q=render_text_description_and_args
  - https://github.com/langchain-ai/langchain/issues?q=render_text_description
  - https://github.com/langchain-ai/langchain/issues?q=tool.args+json+render
- **Fix:** use `json.dumps(tool.args, ensure_ascii=False, default=str)` and correct both docstring examples to the real format. This is a behavior change in prompt text, so call it out in the release note. To stay conservative, the PR can be limited to the docstring fix, with the JSON change as an opt-in or discussed in the issue first.
- **Test:** in `tests/unit_tests/test_tools.py`, a tool with a `bool` default should render `"default": false` with no `'` characters. `json.loads` of the substring after `args: ` should succeed.

## 3. `DictPromptTemplate.input_variables` order is nondeterministic

- **Where:** `libs/core/langchain_core/prompts/dict.py:140`. `_get_input_variables` returns `list(set(input_variables))`.
- **Repro (not executed):** run the following in two processes with different `PYTHONHASHSEED`:
  ```python
  from langchain_core.prompts.dict import DictPromptTemplate
  t = DictPromptTemplate(template={"a": "{x}", "b": "{y}", "c": "{z}"}, template_format="f-string")
  print(t.input_variables)  # order differs between runs, e.g. ['z','x','y'] vs ['y','z','x']
  ```
- **Why it's a bug:** string hash randomization makes the list order change from run to run. Anything that relies on it (generated input schemas, `as_tool` signatures, snapshot tests, serialized manifests diffed in CI) becomes non-reproducible. `PromptTemplate` uses `sorted(...)` for the same purpose.
- **Dup search:** 0 issues and 0 PRs for "input_variables nondeterministic order". Nothing relevant for "input_variables order set prompt" (#41216 is about `append` not refreshing variables).
  - https://github.com/langchain-ai/langchain/issues?q=input_variables+nondeterministic+order
  - https://github.com/langchain-ai/langchain/issues?q=input_variables+order+set+prompt
  - https://github.com/langchain-ai/langchain/issues?q=DictPromptTemplate
- **Fix:** keep first-seen order with `list(dict.fromkeys(input_variables))`, or use `sorted(set(...))` to match `PromptTemplate`. Apply the same to the nested recursion.
- **Test:** for a template with keys whose variables appear in a known order, assert `input_variables == ["x", "y", "z"]` (sorted or insertion order). Add a parametrized nested dict/list case.
- **Severity note:** this is low impact. It is a good first PR but may be seen as minor. Candidates 1 and 2 are stronger.
