# Round 3 rebuttal 3: duplicates and competition

Method: GitHub search API (`search/issues`, `is:issue` / `is:pull-request`), using keywords different from the finders' (2026-11-03). The per-issue REST endpoints (comments, timeline) returned 403 in this session, so for R3-9..R3-11 the comment counts and assignees come from search metadata, and comment bodies come from `find_open_issues.md`. Nothing was executed and nothing was committed.

## Per-candidate findings

- **R3-1 `DictPromptTemplate.pretty_repr`**: `pretty_repr dict` PRs: 5, all on `BaseMessage.pretty_repr` (#36370, #35022, #34339, all closed). None touch prompt templates. Issue `pretty_print image_url ChatPromptTemplate`: 0. Risk Low. Score 7.
- **R3-2 `render_text_description` docs**: PRs: 1 (#25172, an old styling PR). `render_text_description_and_args`: 0. Risk Low, but it is docs-only, so it carries little weight. Score 5.
- **R3-3 `DictPromptTemplate.input_variables` order**: there are 12 `DictPromptTemplate` PRs, all on the `aformat` theme (#41206, #40143, #39984, #39859, which are duplicates of one another and all closed), plus 6 issues (#39835 open). Nothing covers ordering. `input_variables sorted set`: 1 unrelated PR (#40237). The area is crowded with drive-by PRs, which suggests maintainers are tired of it. Risk Med. Score 5.
- **R3-4 qdrant `Document.id`**: `qdrant document id` PRs: 17, the closest being #40598 (mismatched ids) and #38789 (tests for #32283). None set `Document.id` on returned documents. Issue `qdrant Document.id`: 0. Re-check #32283's scope before filing. Risk Low-Med. Score 6.
- **R3-5 chroma `hybrid_search` metadata**: `chroma hybrid` / `chroma hybrid metadata` PRs: 3-5, all unrelated or old (#13352, #8636). Issues: 0. Competition is low, but the bug is still unverified. Risk Low. Score 5.
- **R3-6 xai `stop` plus `model` override**: `xai stop model`: only the origin #38335 is relevant (closed, merged); the other 10 are dependency bumps. `xai bind stop`: 0. Risk Low. Score 6, since the fix is small and niche.
- **R3-7 `ClearToolUsesEdit.keep`**: 5 PRs (#41478 content-block inputs, #39265, #39754, #37818) and 8 issues including open #37815 ("trigger semantics degenerate"), #41238 and #34145. Nobody covers the `keep` counting itself, but this is an active area and it overlaps semantically with #37815, so a maintainer may fold it in there. Risk Med. Score 6.
- **R3-8 `ModelCallLimitMiddleware` validation**: 7 PRs, none on validation (#37072 is tests, closed). 6 issues: #40866 and #40444 open, #41450 and #41307 closed. None is about `run_limit <= thread_limit`. `thread_limit run_limit validation`: 0. Risk Low. Score 7.
- **R3-9 #41601 `convert_to_messages`**: open, 2 comments, no assignee. The reporter, sahil-mangla, filed a duplicate earlier (#41598, closed) and says a PR is ready. Sparsh-2007 has also asked. `convert_to_messages JSONDecodeError` / `empty arguments` PRs: 0 so far. The reporter is likely to win this. Risk High. Score 3.
- **R3-10 #41623 google_genai `extras`**: opened 2026-11-02, 1 comment, no assignee. `google_genai extras` / `genai image extras` PRs: 0 relevant. It is very fresh, so claims will keep piling up, and the fix may belong in core (the title says `content_blocks`). Risk Med-High. Score 4.
- **R3-11 #41603 mistralai content with tool calls**: open, 1 comment (Kylinny's informal claim), no assignee. PR #41619 (21-aakash, closed 2026-11-01) already attempted it, and open #41305 (4 comments) overlaps. 291 broad hits for `mistralai tool calls content` show the area is busy. A third attempt needs maintainer sign-off on the API compatibility question. Risk High. Score 3.

## Summary table

| ID | score | risk |
|---|---|---|
| R3-1 | 7 | Low |
| R3-2 | 5 | Low |
| R3-3 | 5 | Med |
| R3-4 | 6 | Low-Med |
| R3-5 | 5 | Low |
| R3-6 | 6 | Low |
| R3-7 | 6 | Med |
| R3-8 | 7 | Low |
| R3-9 | 3 | High |
| R3-10 | 4 | Med-High |
| R3-11 | 3 | High |
