# API Schemas Explained

This document details the `app/api/schemas.py` module, which uses **Pydantic** to define the data structures (models) for our API. These schemas ensure that data sent to and received from the API is valid and well-structured.

## 1. What is Pydantic?

Pydantic is a data validation library. We define classes (Models), and Pydantic automatically validates incoming JSON data against them. If the data is wrong (e.g., missing field, wrong type), it raises an error automatically.

## 2. Health & Status Schemas

These are used for system monitoring endpoints.

```python
class HealthResponse(BaseModel):
    status: str
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    version: str
```
-   **`default_factory=datetime.utcnow`**: If we don't provide a timestamp, Pydantic generates the current time automatically when the model is created.

## 3. Query Schemas (The Core Logic)

These define how users interact with the RAG system.

### Request: `QueryRequest`
```python
class QueryRequest(BaseModel):
    question: str = Field(..., min_length=1, max_length=1000)
    include_sources: bool = True
    enable_evaluation: bool = False
```
-   **Validation**: `min_length=1` ensures the user can't send an empty question. `max_length` prevents huge payloads that could crash the LLM.
-   **Flags**:
    -   `include_sources`: Toggles whether to show the retrieved documents.
    -   `enable_evaluation`: Toggles whether to run RAGAS (slower, but gives quality metrics).

### Response: `QueryResponse`
This is what the user *sees*.

```python
class QueryResponse(BaseModel):
    question: str
    answer: str
    sources: Optional[list[SourceDocument]]
    processing_time_ms: float
    evaluation: Optional[EvaluationScores]
```
It's a nested structure. `sources` is a list of `SourceDocument` objects, and `evaluation` is an `EvaluationScores` object. Pydantic handles serializing this complex nesting into JSON automatically.

## 4. Evaluation Schemas

```python
class EvaluationScores(BaseModel):
    faithfulness: Optional[float] = Field(None, ge=0.0, le=1.0)
    # ...
```
-   **Constraints**: `ge=0.0` (Greater or Equal) and `le=1.0` (Less or Equal) ensure that scores are always percentages between 0 and 1. If RAGAS returned 1.5 (impossible), Pydantic would catch it.

## 5. Usage in FastAPI

In `app/api/routes/query.py`, we use these schemas like this:

```python
@router.post("/query", response_model=QueryResponse)
async def ask_question(request: QueryRequest):
    # 'request' is already validated and converted to a Python object
    print(request.question) 
    
    # ... logic ...
    
    # We return a dict or object that matches QueryResponse
    return {
        "question": request.question,
        "answer": "...",
        ...
    }
```
FastAPI uses the type hints (`request: QueryRequest`) to:
1.  Read the JSON body.
2.  Validate it.
3.  Generate the Swagger/OpenAPI documentation automatically.
