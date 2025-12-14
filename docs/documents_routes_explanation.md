# Document Routes Module Explained

This document details `app/api/routes/documents.py`, which defines the API endpoints for managing the Knowledge Base (uploading, listing, and deleting documents).

## 1. API Structure
We use `APIRouter` to group these endpoints under the `/documents` prefix.
```python
router = APIRouter(prefix="/documents", tags=["Documents"])
```

## 2. Upload Endpoint: `POST /upload`

This is the entry point for adding data to the RAG system.

```python
@router.post("/upload", response_model=DocumentUploadResponse)
async def upload_document(file: UploadFile = File(...)):
```

### Workflow
1.  **Validation**: Checks if a filename exists.
2.  **Processing**:
    ```python
    processor = DocumentProcessor()
    chunks = processor.process_upload(file.file, file.filename)
    ```
    This abstracts away *how* the file is read (PDF, CSV, etc.) and split. The route handler just receives the ready-to-index text chunks.
3.  **Indexing**:
    ```python
    vector_store = VectorStoreService()
    document_ids = vector_store.add_documents(chunks)
    ```
    The chunks are sent to Qdrant.
4.  **Response**: Returns how many chunks were created and their IDs.

### Error Handling
-   **400 Bad Request**: If the file type is unsupported (raised by `DocumentProcessor`) or file is empty.
-   **500 Internal Server Error**: If Qdrant is down or parsing crashes unexpectedly.

## 3. Collection Info: `GET /info`

Used to monitor the state of the vector database.

```python
@router.get("/info", response_model=DocumentListResponse)
async def get_collection_info():
    info = vector_store.get_collection_info()
    return DocumentListResponse(
        collection_name=info["name"],
        total_documents=info["points_count"], # How many chunks total?
        ...
    )
```

## 4. Reset Collection: `DELETE /collection`

**⚠️ DANGER ZONE**
This endpoint wipes the entire Qdrant collection.

```python
@router.delete("/collection")
async def delete_collection():
    vector_store.delete_collection()
    return {"message": "Collection deleted successfully"}
```

## Design Decisions

-   **Service Instantiation**: We instantiate `DocumentProcessor()` and `VectorStoreService()` inside the route handlers.
    -   *Pros*: Simple, easy to understand.
    -   *Cons*: Harder to unit test (mocking required). In larger apps, we might inject these as dependencies (`Depends(get_vector_service)`).
-   **Streaming Uploads**: `UploadFile` uses Python's `SpooledTemporaryFile`. It handles large files efficiently without loading the entire 100MB PDF into RAM at once.
