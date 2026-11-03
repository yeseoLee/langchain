# Round 4 evaluation

| ID | M | T | D | I | P | Avg | Result | Decisive reason |
|---|---|---|---|---|---|---|---|---|
| R4-1 ollama chunk= | 4 | 3 | 8 | 3 | 5 | 4.6 | Reject | Core `stream()` calls `_stream` without run_manager and emits the token itself (core chat_models.py:836/854) -> the ChatOllama part is not a bug. OllamaLLM alone is real; re-scope later |
| R4-2 groq/fireworks model_name from response | 6 | 6 | 6 | 7 | 6 | 6.2 | **Pass** | Groq first. Watch for conflicts with open PRs #38029, #38759 |
| R4-3 LLMToolEmulator response.text | 8 | 7 | 7 | 6 | 8 | 7.2 | **Pass** | SummarizationMiddleware precedent, needs release note, conflict with #41579 |
| R4-4 ChatOllama ls_max_tokens | 6 | 6 | 9 | 4 | 7 | 6.4 | **Pass** | OllamaLLM parity, only positive num_predict |

Side effect: the R4-1 rebuttal finding (core calls `_stream` without run_manager and itself passes `chunk.message.content` (a list) to the callback) further weakens R1-B (Mistral token). R1-B moved to reserve.
