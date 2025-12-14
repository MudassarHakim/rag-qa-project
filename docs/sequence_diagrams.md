# System Sequence Diagrams

This document visualizes the core workflows of the RAG Q&A System using sequence diagrams.

## 1. Document Ingestion Flow

This flow illustrates how a document travels from the user's upload to being indexed in the vector database.

```mermaid
sequenceDiagram
    participant User
    participant API as FastAPI (Main)
    participant Processor as DocumentProcessor
    participant Splitter as TextSplitter
    participant Embedder as EmbeddingService
    participant Qdrant as Qdrant Vector DB

    User->>API: POST /documents/upload (file)
    activate API
    
    Note right of API: Validates file type<br/>(PDF, TXT, CSV)

    API->>Processor: process_upload(file)
    activate Processor
    
    Processor->>Processor: Save temp file
    Processor->>Processor: Load file (PyPDF/CSV/Text)
    
    Processor->>Splitter: split_documents(documents)
    activate Splitter
    Splitter-->>Processor: returns chunks (List[Document])
    deactivate Splitter
    
    deactivate Processor
    API->>Processor: returns chunks

    API->>Embedder: embed_documents(chunks)
    activate Embedder
    Embedder->>Qdrant: Generate Embeddings (OpenAI)
    Embedder-->>API: returns vectors
    deactivate Embedder

    API->>Qdrant: add_documents(chunks, vectors)
    activate Qdrant
    Note right of Qdrant: Indexes vectors<br/>for similarity search
    Qdrant-->>API: returns success
    deactivate Qdrant

    API-->>User: 200 OK (Upload Summary)
    deactivate API
```

## 2. RAG Query Flow

This flow shows how a user's question is answered using retrieved context.

```mermaid
sequenceDiagram
    participant User
    participant API as FastAPI (Main)
    participant Chain as RAGChain
    participant VectorDB as VectorStoreService
    participant OpenAI as OpenAI LLM
    participant Ragas as RagasEvaluator

    User->>API: POST /query (question)
    activate API

    API->>Chain: query(question)
    activate Chain

    Note right of Chain: Step 1: Retrieval

    Chain->>VectorDB: get_retriever()
    activate VectorDB
    VectorDB->>VectorDB: embed_query(question)
    VectorDB->>VectorDB: similarity_search()
    VectorDB-->>Chain: returns relevant documents
    deactivate VectorDB

    Note right of Chain: Step 2: Generation

    Chain->>Chain: Format context (docs -> string)
    Chain->>Chain: Create Prompt (Context + Question)

    Chain->>OpenAI: Invoke LLM
    activate OpenAI
    OpenAI-->>Chain: returns Answer
    deactivate OpenAI

    opt With Evaluation
        Chain->>Ragas: evaluate(question, answer, context)
        activate Ragas
        Ragas-->>Chain: returns scores
        deactivate Ragas
    end

    Chain-->>API: returns result (answer + sources)
    deactivate Chain

    API-->>User: 200 OK (Answer)
    deactivate API
```
