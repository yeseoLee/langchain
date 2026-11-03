# Round 2 rebuttal 3: duplicates and competition

Method: local `git log --grep` over master history (HEAD is 2026-11-03), plus GitHub REST search
(`search/issues`, langchain-ai/langchain). The REST API would not return per-PR merge status, so a PR
counts as "not merged" when its number is missing from local history. Nothing was executed.

Main finding: several candidates already have an issue and one or more competing PRs. Those PRs were
**closed without merging**. That pattern (many parallel external PRs auto-closed) suggests maintainers
close unassigned or drive-by PRs. A new PR on the same topic will probably meet the same fate unless
it is linked to an assigned issue.

## Per candidate

- **R2-1 fallbacks `exception_key` mutates input**: issue #40878 (closed 2026-09-28) reports this exact bug.
  PR #40913 "don't mutate caller's input dict" was closed and is not in master; #41181 (non-dict
  exception_key) was closed too. The bug still exists, but it was already reported and attempted.
  Risk High.
- **R2-2 invoke/ainvoke pass parent config**: issue #36072 is OPEN (since 2026-03, "drops parent_run_id").
  Three PRs (#36159, #36182, #36189) were all closed unmerged. This has been known for a long time, so
  it is either intentional or blocked by maintainers. Risk High.
- **R2-3 core docstring nonexistent imports**: issue #41064 is OPEN with the same scope (output_parser, runnable,
  documents.Blob). PRs #41113 and #41353 were closed unmerged. Other contributors are actively competing. Risk High.
- **R2-4 anthropic middleware docstring imports**: no issue or PR found. #40815 only touched core. Risk Low.
- **R2-5 huggingface null token counts**: no PR found. #41553 fixed Anthropic only, and groq #38759 (open) is
  the same pattern for another partner. Risk Low-Med.
- **R2-6 perplexity null token counts**: no hit. #40285 touched perplexity usage metadata, but for cost and search
  fields, not null tokens. Risk Low.
- **R2-7 mistralai null token counts**: #38483 (closed 2026-06, not in master) worked on nested dict usage in
  `_combine_llm_outputs`, which is nearby. Mistral also has many open issues (#41490, #39794). Risk Med.
- **R2-8 huggingface async vs sync divergence**: only old fixes (#31385, #18784). No current competing work. Risk Low.
- **R2-9 `provider:` prefix normalization (hyphen/case)**: no issue or PR hits. Risk Low. Lowercasing
  could be argued to be a design choice.
- **R2-10 infer openai for `o4-*`**: issue #38243 (closed) plus three PRs (#38443, #38464, #38472), all
  closed unmerged. `base.py` still lists only `gpt-`, `o1`, `o3`. This looks like a deliberate rejection
  (o4 is an older or legacy family). Risk High.
- **R2-11 provider-table note about unregistered providers**: no hits. Risk Low. Check whether "work"
  means via the integration package.
- **R2-12 text-splitters token overlap and empty-input tests**: #32205 (merged 2025) added the validation,
  possibly with tests. Check the existing tests before filing. Risk Med.
- **R2-13 `load_prompt_from_config` mutation**: the API was deprecated in #36200 (CVE path-traversal fix).
  Maintainers are unlikely to accept fixes to deprecated APIs. No duplicate found. Risk Med (rejection
  rather than duplicate).
- **R2-14 Sphinx double backticks**: #38095 already did a mass replacement, and 490 `` `` `` hits remain in
  core (many are legitimate, e.g. inside fenced code blocks). Mass style PRs from outsiders are low value. Risk Med.

## Summary table (score = attractiveness after duplicate risk, 1-10)

| ID | score | risk |
|---|---|---|
| R2-1 | 3 | High |
| R2-2 | 2 | High |
| R2-3 | 2 | High |
| R2-4 | 7 | Low |
| R2-5 | 7 | Low |
| R2-6 | 7 | Low |
| R2-7 | 5 | Med |
| R2-8 | 6 | Low |
| R2-9 | 6 | Low |
| R2-10 | 1 | High |
| R2-11 | 6 | Low |
| R2-12 | 4 | Med |
| R2-13 | 3 | Med |
| R2-14 | 2 | Med |
