# Document Processor Explained

This document provides a detailed breakdown of the `app/core/document_processor.py` module, which is responsible for loading various file formats and splitting them into chunks for the RAG pipeline.

## 1. Core Components & Imports

```python
from langchain_community.document_loaders import CSVLoader, PyPDFLoader, TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
```

-   **Loaders**: We use LangChain's specialized loaders for PDF, CSV, and Text files.
-   **Splitter**: `RecursiveCharacterTextSplitter` is used to break large documents into smaller semantic chunks. It tries to keep related text together (e.g., paragraphs) by splitting on separators like `\n\n`, `\n`, etc.

---

## 2. Initialization: `DocumentProcessor`

```python
class DocumentProcessor:
    SUPPORTED_EXTENSIONS = {".pdf", ".txt", ".csv"}

    def __init__(self, chunk_size=None, chunk_overlap=None):
        # Configures the text splitter with size and overlap
        self.text_splitter = RecursiveCharacterTextSplitter(...)
```

-   **Settings**: It reads `chunk_size` (e.g., 1000 characters) and `chunk_overlap` (e.g., 200 characters) from the app configuration.
-   **Overlap**: Essential for RAG. It ensures context isn't lost at the boundaries of chunks.

---

## 3. Loading Files

The class has specific methods for each file type, unified by a dispatcher.

### Specific Loaders
-   **`load_pdf`**: Uses `PyPDFLoader` to extract text from PDF pages. Each page usually becomes one `Document`.
-   **`load_text`**: Uses `TextLoader` for plain text files.
-   **`load_csv`**: Uses `CSVLoader`. Each row in the CSV usually becomes a separate `Document`.

### The Dispatcher: `load_file`

```python
def load_file(self, file_path: Union[str, Path]) -> list[Document]:
    extension = file_path.suffix.lower()
    # ... checks support ...
    loaders = {
        ".pdf": self.load_pdf,
        ".txt": self.load_text,
        ".csv": self.load_csv,
    }
    return loaders[extension](file_path)
```
This method automatically chooses the right loader based on the file extension. This is the **Strategy Pattern** in action.

---

## 4. Handling Uploads: `load_from_upload`

This is critical for web APIs (FastAPI) because uploaded files arrive as streams in memory, but many LangChain loaders expect a *file path* on disk.

```python
def load_from_upload(self, file: BinaryIO, filename: str) -> list[Document]:
    # 1. Create a temporary file
    with tempfile.NamedTemporaryFile(delete=False, suffix=extension) as tmp_file:
        tmp_file.write(file.read())
        tmp_path = tmp_file.name

    try:
        # 2. Load from the temp file path
        documents = self.load_file(tmp_path)
        
        # 3. Restore original filename metadata
        for doc in documents:
            doc.metadata["source"] = filename
            
        return documents
    finally:
        # 4. Clean up (delete) the temp file
        Path(tmp_path).unlink(missing_ok=True)
```

**Key Steps:**
1.  **Write to Disk**: Save the upload to a temp file so `PyPDFLoader` (etc.) can read it.
2.  **Process**: Call `load_file`.
3.  **Fix Metadata**: The loader will think the source is `/tmp/xyz123.pdf`. We overwrite this with the actual filename `report.pdf`.
4.  **Cleanup**: Always delete the temp file in a `finally` block to prevent disk clutter.

---

## 5. Splitting: `split_documents`

```python
def split_documents(self, documents: list[Document]) -> list[Document]:
    return self.text_splitter.split_documents(documents)
```

If you load a 50-page PDF, you might get 50 `Document` objects (one per page). If a page has 5000 characters, it's too big for an LLM context or vector search.
`split_documents` breaks these into smaller chunks (e.g., 1000 chars) while preserving metadata.

---

## Usage Examples

### Case 1: Processing a Local File
```python
processor = DocumentProcessor()

# One-step: Load -> Split
chunks = processor.process_file("data/annual_report.pdf")

print(f"Generated {len(chunks)} chunks.")
# Generated 45 chunks.
```

### Case 2: Processing an API Upload (FastAPI)
```python
@app.post("/upload")
async def upload_document(file: UploadFile):
    processor = DocumentProcessor()
    
    # file.file is the file-like object
    chunks = processor.process_upload(
        file=file.file, 
        filename=file.filename
    )
    
    # Now index these chunks into vector DB...
    return {"chunks": len(chunks)}
```
