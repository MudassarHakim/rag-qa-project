# Query Routes Module Explained

This document details `app/api/routes/query.py`, which is the primary interface for users to ask questions. It connects the API layer to the RAG Logic (`RAGChain`).

## 1. The Main Query Endpoint: `POST /query`

```python
@router.post("", response_model=QueryResponse)
async def query(request: QueryRequest) -> QueryResponse:
```

This endpoint handles the "Ask a question" feature. It supports three modes based on valid input flags:

1.  **Standard Mode**: Just gets the answer.
    ```python
    answer = await rag_chain.aquery(request.question)
    ```
2.  **Sources Mode** (`include_sources=True`): Returns answer + retrieved documents.
    ```python
    result = await rag_chain.aquery_with_sources(request.question)
    ```
3.  **Evaluation Mode** (`enable_evaluation=True`): Returns answer + sources + quality metrics.
    ```python
    result = await rag_chain.aquery_with_evaluation(...)
    ```

It consolidates these different return values into a unified `QueryResponse` object (defined in `schemas.py`), ensuring a consistent JSON structure for the frontend.

## 2. Streaming Endpoint: `POST /query/stream`

For a better UX (like ChatGPT), we stream tokens as they are generated instead of waiting for the full answer.

```python
@router.post("/stream")
async def query_stream(request: QueryRequest) -> StreamingResponse:
    # Generator function
    async def generate():
        for chunk in rag_chain.stream(request.question):
            yield chunk

    return StreamingResponse(generate(), media_type="text/plain")
```

-   **`StreamingResponse`**: A FastAPI special response class that keeps the HTTP connection open and sends data chunks.
-   **`rag_chain.stream`**: The method deep inside `RAGChain` (powered by LCEL) that yields tokens.

## 3. Direct Search: `POST /query/search`

This is a debugging/utility endpoint. It skips the LLM generation step and just returns what the vector database finds.

```python
@router.post("/search")
async def search_documents(request: QueryRequest):
    vector_store = VectorStoreService()
    results = vector_store.search_with_scores(request.question)
    # Returns raw document chunks + relevance scores
```
**Use Case**: If the bot gives a bad answer, developers can use this to check if the *retrieval* step failed (i.e., did it find the right documents?).

## Architecture Note

The logic here is intentionally "thin". The route handlers essentially act as dispatchers:
`Request -> Route Handler -> RAGChain -> LLM`

Complex logic (prompting, retrieval strategy, evaluation) is kept inside `RAGChain`, keeping the API layer clean.
