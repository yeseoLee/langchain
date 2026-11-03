# Round 3: partner contribution candidates

Duplicate checks used the GitHub REST search API (`gh api search/issues`, `is:issue` and `is:pull-request`), which covers the same data as `https://github.com/langchain-ai/langchain/issues?q=...`. Nothing was executed; the repro snippets below have not been run.

## Dropped during the duplicate check
- **`QdrantVectorStore.delete(ids=None)` / `Chroma.delete(ids=None)` do not delete everything, although the core docstring says "If `None`, delete all"**: the same class of issue was closed as not planned in #40509 (`InMemoryVectorStore.delete(None)`), https://github.com/langchain-ai/langchain/issues/40509. Related: closed PR #31712 (qdrant metadata delete).
- **Qdrant has no async support**: duplicates #32195 (open), #34812 (open PR), #39648 (closed PR).
- **Exa retriever does not set `Document.id`**: weak, because the id is already stored in `metadata["id"]`.

---

## 1. fix(qdrant): set `Document.id` on documents returned by `QdrantVectorStore` (get_by_ids / search)

- **Where:** `libs/partners/qdrant/langchain_qdrant/qdrant.py:1023-1036` (`_document_from_point`), used by `get_by_ids` (`:877`) and by every search method. The same pattern exists in the legacy store, `libs/partners/qdrant/langchain_qdrant/vectorstores.py:~2050`.
- **Bug:** the point id is written only to `metadata["_id"]`. `Document(id=...)` is never set. The core `VectorStore.get_by_ids` contract says "The returned documents are expected to have the ID field set to the ID of the document in the vector store". Chroma already follows it (`vectorstores.py:44,62,1204`).
- **Repro:**
  ```python
  store = QdrantVectorStore.from_texts(["foo"], embeddings, location=":memory:", collection_name="t", ids=[UUID])
  store.get_by_ids([UUID])[0].id                 # None, expected UUID
  store.similarity_search("foo", k=1)[0].id      # None
  ```
  The `standard-tests` `VectorStoreIntegrationTests.test_get_by_ids` checks `id`. Check whether qdrant overrides or skips that test.
- **Sibling precedent:** Chroma `_results_to_docs` sets `id=`.
- **Fix:** add `id=str(scored_point.id)` in `_document_from_point`, or keep the int form. Keep `metadata["_id"]` so existing users are not broken.
- **Test idea:** a unit test with the in-memory `QdrantClient(":memory:")` that asserts `doc.id == metadata["_id"]` for `get_by_ids` and `similarity_search`.
- **Dup search:** "qdrant get_by_ids", "qdrant Document.id", "qdrant _id", "QdrantVectorStore id field" found no match (only unrelated #39937, #22231, #31712).

## 2. fix(chroma): `Chroma.hybrid_search` crashes when a record has no metadata

- **Where:** `libs/partners/chroma/langchain_chroma/vectorstores.py:719-727`.
- **Bug:** it builds `Document(..., metadata=record["metadata"])` with no fallback. Chroma returns `None` metadata for documents added without metadatas, or when the search's `.select(...)` does not include metadata (the docstring example selects only `K.DOCUMENT, K.SCORE, "title", "year"`). `Document.metadata` is a `dict`, so `None` fails pydantic validation, and the subscript can also raise `KeyError`. Every other path uses `result[1] or {}` (`:44,62`).
- **Repro:**
  ```python
  db = Chroma.from_texts(["a", "b"], embeddings)   # no metadatas
  db.hybrid_search(Search().rank(Knn(query="a")).limit(2).select(K.DOCUMENT))
  ```
  Expected: ValidationError or KeyError. Not verified, because chromadb's `rows()` shape could not be inspected (no venv).
- **Fix:** `metadata=record.get("metadata") or {}` and `id=record.get("id")`.
- **Test idea:** a unit test with a mocked `_collection.search` that returns `rows()` == `[[{"id": "1", "document": "a", "metadata": None}]]` and asserts `metadata == {}`.
- **Dup search:** "chroma hybrid_search" found nothing; "chroma hybrid metadata" found only unrelated results.

## 3. fix(xai): `stop` dropping ignores a call-time `model` override

- **Where:** `libs/partners/xai/langchain_xai/chat_models.py:564-588`.
- **Bug:** `_get_request_payload` decides whether to drop `stop` with `self._resolve_model_profile()` and `_model_rejects_stop(self.model_name)`. Both use the **instance** model, not `payload["model"]`. Two cases:
  - `ChatXAI(model="grok-4.20-0309-non-reasoning").invoke(msgs, model="grok-4", stop=["\n"])` sends `stop` to a reasoning model, and the API returns a 400.
  - The reverse case drops `stop` silently for a model that supports it.
- **Merged precedent:** #38335 `fix(xai): drop unsupported stop parameter` introduced this logic and covers only the instance model.
- **Fix:** resolve with `model_name = payload.get("model", self.model_name)` and use `_get_default_model_profile(model_name)`. This is a small refactor of `_resolve_model_profile` to accept a name. Keep the private signature backward compatible.
- **Test idea:** add parametrized unit tests on `_get_request_payload` with an instance model and a `model=` kwarg pair. Assert that `"stop"` is present or absent in the payload.
- **Dup search:** "xai stop" found only #38335 (the merged origin PR) and unrelated Perplexity stop issues (#40830, #40914). No issue covers the override case.
