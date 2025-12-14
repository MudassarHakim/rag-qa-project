# Embeddings Module Explained

This document explains the `app/core/embeddings.py` module, which is responsible for converting text into vector representations (embeddings) using OpenAI's models. These embeddings are the core of semantic search in RAG.

## 1. Core Concept: What are Embeddings?

Embeddings are lists of floating-point numbers (vectors) that represent the *meaning* of text.
-   **Similar meaning** = **Close distance** in vector space.
-   **Different meaning** = **Far distance**.

We use `OpenAIEmbeddings` via LangChain to generate these vectors.

---

## 2. Configuration & Caching: `get_embeddings`

```python
@lru_cache
def get_embeddings() -> OpenAIEmbeddings:
    settings = get_settings()
    
    embeddings = OpenAIEmbeddings(
        model=settings.embedding_model,      # e.g., text-embedding-3-small
        openai_api_key=settings.openai_api_key,
    )
    return embeddings
```

### Why `@lru_cache`?
Creating the `OpenAIEmbeddings` client might involve reading config or setting up connections. Since the configuration doesn't change during runtime, we cache the instance using `functools.lru_cache`. This ensures we reuse the same client object throughout the application, saving memory and initialization time.

### Configuration
It uses `get_settings()` to fetch:
-   **`embedding_model`**: Typically `text-embedding-3-small` (efficient and cheap) or `text-embedding-3-large` (higher accuracy).
-   **`openai_api_key`**: Credentials to call OpenAI's API.

---

## 3. The Service Class: `EmbeddingService`

While `get_embeddings()` returns the raw LangChain object, `EmbeddingService` provides a wrapper around it. This is useful for:
1.  **Abstraction**: If we switch from OpenAI to HuggingFace later, we just change the internal implementation here, and the rest of the app (`Class EmbeddingService`) stays the same.
2.  **Logging**: We add debug logs to track when embedding generation happens (which costs money/tokens).

### Methods

1.  **`embed_query(text: str)`**
    -   Used for the **User's Question**.
    -   Generates a single vector.
    -   Example: "How do I reset my password?" -> `[0.12, -0.45, ...]`

2.  **`embed_documents(texts: list[str])`**
    -   Used for the **Knowledge Base**.
    -   Takes a list of text chunks.
    -   Generates a list of vectors.
    -   This is batched automatically by LangChain for efficiency.

---

## Usage Examples

### Direct Usage
```python
from app.core.embeddings import get_embeddings

embeddings = get_embeddings()
vector = embeddings.embed_query("Hello world")
print(len(vector)) 
# Output: 1536 (dimension for text-embedding-3-small)
```

### Service Usage (Recommended)
```python
from app.core.embeddings import EmbeddingService

service = EmbeddingService()

# 1. Embed a question
query_vector = service.embed_query("What is RAG?")

# 2. Embed a batch of documents (e.g., during upload)
texts = ["RAG stands for...", "It combines retrieval..."]
doc_vectors = service.embed_documents(texts)
```

## Why 1536 Dimensions?
The `text-embedding-3-small` model outputs vectors with 1536 dimensions. This means every piece of text is represented by 1536 numbers. When we set up Qdrant (our vector DB), we must specify this exact dimension size, or the storage will fail.
