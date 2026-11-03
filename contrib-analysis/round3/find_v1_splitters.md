# Round 3 candidates: langchain_v1 middleware + text-splitters

## Dropped (duplicates found)
- `Language.PERL` raises "not implemented" (character.py ~795): dup #37049/#37048/#36750, many closed PRs (#38543, #37213, ...). https://github.com/langchain-ai/langchain/issues?q=Language.PERL+not+implemented
- `ExperimentalMarkdownSyntaxTextSplitter._resolve_code_chunk` drops unterminated fence / mismatched fence close: dup #37966, #36186, #40967, PR #41013. https://github.com/langchain-ai/langchain/issues?q=ExperimentalMarkdownSyntaxTextSplitter+code+block
- `MarkdownHeaderTextSplitter` closes fence on shorter/info-string fence (markdown.py:168-178): dup #41061. https://github.com/langchain-ai/langchain/issues?q=MarkdownHeaderTextSplitter+fence
- `RecursiveJsonSplitter` sizing ignores `ensure_ascii`: dup #40761 / PR #40763. https://github.com/langchain-ai/langchain/issues?q=RecursiveJsonSplitter+ensure_ascii
- `RecursiveJsonSplitter` chunks exceed `max_chunk_size` for nested paths: dup #39127. https://github.com/langchain-ai/langchain/issues?q=RecursiveJsonSplitter+exceeds+max_chunk_size

## 1. `ClearToolUsesEdit.keep` counts excluded and already-cleared tool results
- Location: `libs/langchain_v1/langchain/agents/middleware/context_editing.py:91-98` (candidate list built before `exclude_tools` / `cleared` filtering at :104 and :126).
- Problem: `keep` is documented as "Number of most recent tool results that must be preserved", but the last `keep` ToolMessages are taken off the list regardless of whether they belong to `exclude_tools` (which are never cleared anyway). With `keep=3, exclude_tools=["search"]` and the 3 most recent results from `search`, every non-excluded result is cleared, so zero clearable recent results are preserved.
- Repro (not executed): messages = [AI(call a1 "fetch"), Tool(a1, "fetch"), AI(calls s1,s2,s3 "search"), Tool s1, Tool s2, Tool s3]; `ClearToolUsesEdit(trigger=0, keep=3, exclude_tools=("search",))`; `apply(msgs, count_tokens=count_tokens_approximately)` -> the `fetch` result is replaced by placeholder although it is among the 3 most recent clearable results... (it's the 4th most recent overall). Expected (per docs intent): preserve last 3 clearable results.
- Note: semantics change; propose fixing by filtering excluded tools before applying `keep`, or at minimum clarify the docstring. Requires maintainer opinion — flag as behavior change.
- Reference: last related merged PR #39754 (custom token_counter, commit 37f2662).
- Test idea: unit test in `tests/unit_tests/agents/middleware/.../test_context_editing.py` asserting the `fetch` ToolMessage keeps original content.
- Dup search: https://github.com/langchain-ai/langchain/issues?q=ClearToolUsesEdit+keep , ?q=ClearToolUsesEdit+keep+exclude_tools , ?q=ClearToolUsesEdit+clear_tool_inputs -> only #37815, #34145, #41238/#41478, #41449, #40261, #39754 (unrelated).

## 2. `ModelCallLimitMiddleware` lacks `run_limit > thread_limit` validation (parity with `ToolCallLimitMiddleware`)
- Location: `libs/langchain_v1/langchain/agents/middleware/model_call_limit.py:154-164`; reference implementation `tool_call_limit.py:245-250` (introduced in #34576, commit 05fe082 per `git log -S`).
- Problem: `ModelCallLimitMiddleware(thread_limit=2, run_limit=5)` is silently accepted; `run_limit` can never be reached, an obvious misconfiguration that `ToolCallLimitMiddleware` rejects with `ValueError`.
- Repro: `ModelCallLimitMiddleware(thread_limit=2, run_limit=5)` -> no error (ToolCallLimitMiddleware with same args raises).
- Fix: copy the check/message. Note: newly raising is mildly breaking for misconfigured users; mention in PR.
- Test: `pytest.raises(ValueError, match="run_limit")`; equal limits still accepted.
- Dup search: https://github.com/langchain-ai/langchain/issues?q=ModelCallLimitMiddleware+run_limit+thread_limit+validation (0 results); https://github.com/langchain-ai/langchain/issues?q=ModelCallLimitMiddleware (20 results, none about validation).

## 3. Call-limit middlewares accept negative / zero-meaningless limits
- Location: `model_call_limit.py:154-164`, `tool_call_limit.py:236-250`.
- Problem: `thread_limit=-1` or `run_limit=-3` accepted; the agent then ends before the first model call (`0 >= -1`), with a confusing message "run limit (0/-3)". No `bool` rejection either (`True` -> limit 1).
- Fix: raise `ValueError` if a limit is not `None` and `< 0` (decide whether 0 is allowed; 0 is a legitimate "disable" in some uses — keep 0 allowed).
- Test: parametrize negative values for both classes -> `ValueError`; 0 and positive OK.
- Dup search: https://github.com/langchain-ai/langchain/issues?q=CallLimitMiddleware+negative+limit (0 results); ModelCallLimitMiddleware list above has no match.
- Could be combined with #2 into one PR.
