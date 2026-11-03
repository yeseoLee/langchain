# Contribution points that passed the rebuttal review (cumulative)

Criterion: survived the 5-perspective rebuttal (maintainer, technical, duplicates, impact, process), average >= 5.5, with no fatal rebuttal left unresolved.
Goal: 10. Duplicate check against upstream open issues/PRs not done (GitHub API blocked) — search manually before filing.

| # | Round | Title (draft) | Avg | Notes / conditions |
|---|---|---|---|---|
| 1 | R1 | fix(core): escape Mermaid edge labels | 6.4 | Escape before wrapping, `#` regression test, actual parse-failure repro. Ref #41512 |
| 2 | R1 | fix(mistralai): pass `chunk.text` to `on_llm_new_token` like other partners | 6.2 | Drop the crash claim; reframe as consistency. Ref #34739, #41643. Ask a maintainer first |

Rejected: R1-C (stale python.langchain.com links in standard-tests, 3.0)
