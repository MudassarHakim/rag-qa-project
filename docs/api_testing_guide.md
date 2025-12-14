# API Testing Guide

This guide provides a comprehensive list of endpoints to verify the system's functionality, along with sample `curl` commands and expected workflows.

**Base URL**: `http://localhost:8000` (Default)

## 1. System Health Checks

Verify the application is running and connected to services.

### Basic Liveness Check
Checks if the API server is up.
```bash
curl -X GET http://localhost:8000/health
```
**Expected Response:**
```json
{
  "status": "healthy",
  "timestamp": "2024-05-20T10:00:00.000000",
  "version": "0.1.0"
}
```

### Readiness Check
Checks connections to Qdrant and other dependencies.
```bash
curl -X GET http://localhost:8000/health/ready
```
**Expected Response:**
```json
{
  "status": "ready",
  "qdrant_connected": true,
  "collection_info": { ... }
}
```

---

## 2. Document Management

Flow: Upload a document -> Verify it's indexed.

### Upload Document
Upload a PDF, TXT, or CSV file.
```bash
# Replace 'sample.pdf' with your actual file path
curl -X POST http://localhost:8000/documents/upload \
  -H "accept: application/json" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@sample.pdf"
```

### Check Collection Info
Verify the document count increased.
```bash
curl -X GET http://localhost:8000/documents/info
```

### Delete Collection (Reset)
**Warning**: Deletes all documents.
```bash
curl -X DELETE http://localhost:8000/documents/collection
```

---

## 3. Querying & RAG

Flow: Ask a question -> Get Answer -> (Optional) Stream Answer.

### Standard Query
Ask a question and get the full answer at once.
```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "question": "What does this document say about X?",
    "include_sources": true,
    "enable_evaluation": false
  }'
```

### Streaming Query (Real-time)
Receive the answer token-by-token.
```bash
curl -N -X POST http://localhost:8000/query/stream \
  -H "Content-Type: application/json" \
  -d '{
    "question": "Summarize the key points."
  }'
```

### Debug Search (Retrieval Only)
See what the vector DB finds without generating an answer.
```bash
curl -X POST http://localhost:8000/query/search \
  -H "Content-Type: application/json" \
  -d '{
    "question": "specific keyword"
  }'
```

---

## 4. Testing Evaluation (RAGAS)

Validation of RAG quality using "Faithfulness" (hallucination check) and "Answer Relevancy".

### Limitations
-   **Latency**: Evaluation adds 5-10 seconds to the request time as it invokes the LLM multiple times.
-   **Cost**: Consumes more OpenAI tokens.

### Run Evaluation Query
Set `enable_evaluation: true` in your request.

```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "question": "What is the capital of France?",
    "include_sources": true,
    "enable_evaluation": true
  }'
```

**Expected Response with Evaluation:**
```json
{
  "question": "What is the capital of France?",
  "answer": "The capital of France is Paris.",
  "sources": [...],
  "processing_time_ms": 12500.5,
  "evaluation": {
    "faithfulness": 1.0,        // 1.0 = Fully supported by sources
    "answer_relevancy": 0.98,   // 0.98 = Highly relevant to question
    "evaluation_time_ms": 4500.0,
    "error": null
  }
}
```

### Interpreting Scores
-   **Faithfulness < 0.7**: The model might be hallucinating info not found in the documents.
-   **Answer Relevancy < 0.7**: The model might be dodging the question or giving generic answers.

---

## Sample End-to-End Workflow

1.  **Check Health**: Ensure `GET /health/ready` returns `status: ready`.
2.  **Upload Data**: Run `POST /documents/upload` with your knowledge base file.
3.  **Verify Index**: Run `GET /documents/info` and check `total_documents > 0`.
4.  **Ask Question**: Run `POST /query` with a relevant question.
5.  **Check Quality**: (Optional) Set `"enable_evaluation": true` in the query to see RAGAS scores.
