# RAG Chain Module Explained

This document details the `app/core/rag_chain.py` module, which orchestrates the Retrieval-Augmented Generation (RAG) process using **LangChain LCEL** (LangChain Expression Language).

## 1. What is LCEL?

LCEL is a declarative way to build AI chains. Instead of writing nested functions, we use the pipe operator (`|`) to flow data from one component to the next.

**Data Flow:**
`Input -> Retriever -> Prompt -> LLM -> Parser -> Output`

## 2. Initialization: `RAGChain`

```python
class RAGChain:
    def __init__(self, vector_store_service=None):
        # 1. Setup Vector Store
        self.vector_store = vector_store_service or VectorStoreService()
        self.retriever = self.vector_store.get_retriever()

        # 2. Setup LLM (OpenAI)
        self.llm = ChatOpenAI(model="gpt-4o-mini", ...)

        # 3. Setup Prompt
        self.prompt = ChatPromptTemplate.from_template(RAG_PROMPT_TEMPLATE)

        # 4. Build the Chain
        self.chain = (
            {
                "context": self.retriever | format_docs,  # Fetch & Format docs
                "question": RunnablePassthrough(),        # Pass through user question
            }
            | self.prompt  # Insert into prompt
            | self.llm     # Send to OpenAI
            | StrOutputParser() # Extract string from response
        )
```

### The Chain Breakdown
1.  **Input Dictionary**:
    -   `"question"`: Getting the user's input directly using `RunnablePassthrough()`.
    -   `"context"`: Taking that same input, passing it to `self.retriever` to find documents, and then piping those documents to `format_docs` to make them a single string.
2.  **`| self.prompt`**: The populated dictionary (context + question) is injected into the `RAG_PROMPT_TEMPLATE`.
3.  **`| self.llm`**: The formatted prompt is sent to OpenAI.
4.  **`| StrOutputParser()`**: The raw `AIMessage` from OpenAI is converted to a simple string.

## 3. Query Methods

The class provides multiple ways to interact with this chain:

### Standard Query (`query`)
```python
def query(self, question: str) -> str:
    return self.chain.invoke(question)
```
-   **Usage**: Simple Q&A.
-   **Output**: Just the answer text.

### Query with Sources (`query_with_sources`)
```python
def query_with_sources(self, question: str) -> dict:
    answer = self.chain.invoke(question)
    source_docs = self.retriever.invoke(question)
    # ... formats sources ...
    return {"answer": answer, "sources": source_docs}
```
-   **Usage**: When you need to show citations.
-   **Note**: This runs retrieval *twice* (once inside the chain, once explicitly). Optimization: You could refactor the chain to return documents, but this implementation keeps the chain logic simple.

### Async Query (`aquery`)
```python
async def aquery(self, question: str) -> str:
    return await self.chain.ainvoke(question)
```
-   **Usage**: Non-blocking calls for high-performance APIs (FastAPI).

### Query with Evaluation (`aquery_with_evaluation`)
```python
async def aquery_with_evaluation(self, question: str) -> dict:
    # 1. Get Answer & Sources
    result = await self.aquery_with_sources(question)
    
    # 2. Lazy Load Evaluator
    evaluator = self.evaluator 
    
    # 3. Run RAGAS Check
    evaluation = await evaluator.aevaluate(...)
    
    return {..., "evaluation": evaluation}
```
-   **Usage**: For monitoring quality in production. It checks if the answer was faithful to the sources.

### Streaming (`stream`)
```python
def stream(self, question: str):
    for chunk in self.chain.stream(question):
        yield chunk
```
-   **Usage**: Real-time typing effect in the UI. Instead of waiting 5s for the full answer, the user sees words appear instantly.

## Prompt Template

The system follows a strict instruction set:
```text
You are a helpful assistant. Answer the question based on the provided context.
If you cannot answer... say "I don't have enough information..."
Do not make up information.
```
This "Grounding" instruction is crucial to prevent hallucinations.
