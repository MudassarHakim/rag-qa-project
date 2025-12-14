# Vector Store Module Explained

This document details the `app/core/vector_store.py` module, which manages interactions with the Qdrant vector database. This is where embeddings are stored and searched.

## 1. Imports & Configuration

```python
from langchain_qdrant import QdrantVectorStore
from qdrant_client import QdrantClient
from qdrant_client.http.models import Distance, VectorParams
```

-   **`QdrantClient`**: The official Python client for Qdrant.
-   **`QdrantVectorStore`**: LangChain's wrapper. It simplifies adding documents and searching.
-   **`VectorParams`**: Used to define the schema of the collection (dimensionality and distance metric).

## 2. Client Initialization: `get_qdrant_client`

```python
@lru_cache
def get_qdrant_client() -> QdrantClient:
    # ... checks settings ...
    return QdrantClient(url=..., api_key=...)
```

-   **Caching**: Like embeddings, we cache the client connection to avoid overhead.
-   **Settings**: Connects to Qdrant Cloud using credentials from `config.py`.

## 3. The `VectorStoreService` Class

This class encapsulates all vector database logic.

### Initialization & Schema Creation
```python
def __init__(self, collection_name=None):
    self.client = get_qdrant_client()
    self._ensure_collection()
```

**`_ensure_collection`**:
This method checks if the collection exists on startup.
-   If **No**: It creates it with specific parameters:
    -   `size=1536`: Matches OpenAI's `text-embedding-3-small`.
    -   `distance=Distance.COSINE`: The standard metric for semantic similarity.
-   If **Yes**: It logs the current point count.

### Adding Documents
```python
def add_documents(self, documents: list[Document]) -> list[str]:
    # 1. Generate UUIDs for each doc
    ids = [str(uuid4()) for _ in documents]
    
    # 2. Upload to Qdrant
    self.vector_store.add_documents(documents, ids=ids)
    return ids
```
This takes the chunked documents (from `DocumentProcessor`) and uploads them. Qdrant indexes the vectors automatically.

### Searching

1.  **`search(query: str, k: int)`**:
    -   Basic semantic search.
    -   Returns list of `Document` objects.
    -   `k` is the number of results (e.g., top 4).

2.  **`search_with_scores(query: str, k: int)`**:
    -   Returns documents *plus* a similarity score (0 to 1).
    -   Useful for filtering out irrelevant results (e.g., only show if score > 0.7).

### Retriever for LangChain
```python
def get_retriever(self, k: int) -> Any:
    return self.vector_store.as_retriever(...)
```
This returns a standard LangChain `Retriever` interface. This is what we pass to the `RAGChain` so it can automatically fetch context during a conversation.

## Usage Example

```python
from app.core.vector_store import VectorStoreService

# 1. Initialize
# Automatically connects and ensures 'rag_documents' collection exists
store = VectorStoreService()

# 2. Add Documents
docs = [Document(page_content="RAG is cool")]
store.add_documents(docs)

# 3. Search
results = store.search("What is RAG?", k=2)
print(results[0].page_content)
```

## Management Methods
-   **`delete_collection`**: Wipes all data. Dangerous!
-   **`get_collection_info`**: returns stats like how many vectors are indexed.
-   **`health_check`**: Verifies connectivity.
