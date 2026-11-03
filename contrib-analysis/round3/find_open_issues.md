# Open-issue contribution points (round 3)

Checked 2026-11-03 against local HEAD `b69e5a3`. Source: GitHub search API (open, `bug` label, no assignee, created after 2026-09-20) plus the issue pages. Workflow note: `require_issue_link.yml` requires an external PR author to be assigned, so step one for each item is to comment on the issue with a concrete fix plan and ask to be assigned.

Caveat on competition: none of the ~50 recent unassigned bug issues has zero competition. Most have 1-3 PRs that were auto-closed because the author wasn't assigned. The three below have the least: no linked open PR, no maintainer reply, no rejection. In each case the reporter (or one other person) has already asked for assignment, and maintainers often favor the reporter.

## 1. core: `convert_to_messages` raises JSONDecodeError on empty-string tool-call arguments
- Link: https://github.com/langchain-ai/langchain/issues/41601 (opened 2026-10-31; labels bug, core, external; unassigned)
- Competition: 2 people have asked for assignment, the reporter sahil-mangla (has a PR ready; an earlier attempt, #41598, was auto-closed) and Sparsh-2007. No PR is linked in search. No maintainer reply.
- Code: `libs/core/langchain_core/messages/utils.py:651-653` in `_create_message_from_message_type`. It calls `json.loads(args, strict=False)` on any string, so `""` raises an error. `None` gets past it and then fails pydantic validation.
- Fix: if `args` is `None`, or a string that is empty after stripping whitespace, use `{}`. This matches `parse_tool_call` in `output_parsers/openai_tools.py`. Leave non-empty invalid JSON as it is (out of scope per the reporter).
- Test: in `tests/unit_tests/messages/test_utils.py`, parametrize `arguments` over `""`, `None`, `"{}"` and assert `tool_calls[0]["args"] == {}`.

## 2. core: `content_blocks` drops `extras` when converting Google GenAI `image_url` blocks
- Link: https://github.com/langchain-ai/langchain/issues/41623 (opened 2026-11-02; unassigned)
- Competition: 2 people have asked for assignment, the reporter hyeonsang010716 (has a fix on a fork) and bindhushree05. No PRs found. No maintainer reply.
- Code: `libs/core/langchain_core/messages/block_translators/google_genai.py:398-433`. Both new `image` block dicts (data-URI and raw-base64 branches) are built without copying `item.get("extras")`. That loses the Gemini thought `signature`, and the next turn fails with "missing a thought_signature" (langchain-google#1837).
- Fix: in both branches, add `if "extras" in item: block["extras"] = item["extras"]`. Also consider carrying over `id`.
- Test: in `tests/unit_tests/messages/block_translators/test_google_genai.py`, build an `AIMessage` with `response_metadata={"model_provider": "google_genai"}` and an `image_url` data-URL block that has `extras={"signature": "c2ln"}`. Assert `content_blocks[0]["extras"]["signature"] == "c2ln"`. Repeat for raw base64.

## 3. mistralai: assistant text is dropped when the message also has tool calls
- Link: https://github.com/langchain-ai/langchain/issues/41603 (opened 2026-10-31; unassigned)
- Competition: 1 person, Kylinny, has said "I'd like to take this one" (an informal claim, no assignment). One PR, #41619 by 21-aakash, is closed. No maintainer reply. Overlaps with open #41305 (thinking blocks dropped), which is the higher-risk part.
- Code: `libs/partners/mistralai/langchain_mistralai/chat_models.py:579-583` in `_convert_message_to_mistral_chat_message`. `if tool_calls and content: content = ""` was a workaround for #21196, from when the Mistral API rejected messages with both content and tool calls.
- Fix: remove the blanking and keep `content`, cleaning list content with `_clean_block`. First confirm in the Mistral docs that content plus tool_calls is accepted now. A maintainer will need to sign off on that compatibility change.
- Test: in `tests/unit_tests/test_chat_models.py`, convert `AIMessage(content="plan", tool_calls=[...])` and assert the payload keeps `content == "plan"` and `tool_calls`. Update any existing test that asserts `""`.
