# Main Application Module Explained

This document details `app/main.py`, the entry point for our FastAPI application. This is where everything comes together to launch the server.

## 1. Environment Loading

```python
from dotenv import load_dotenv
load_dotenv()
```
**CRITICAL**: We call `load_dotenv()` *before* importing anything else. This ensures that environment variables (like `OPENAI_API_KEY`) are loaded from the `.env` file before modules like LangChain try to access them.

## 2. Lifespan Manager

FastAPI's modern way to handle startup and shutdown logic.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- Startup Logic ---
    setup_logging(settings.log_level)
    logger.info(f"Starting {settings.app_name}...")
    
    yield  # Application runs here
    
    # --- Shutdown Logic ---
    logger.info("Shutting down application")
```
-   **Startup**: We configure the global logging system here.
-   **Shutdown**: A place to close database connections or cleanup resources (e.g., closing the Qdrant client if needed).

## 3. Application Setup

```python
app = FastAPI(
    title=settings.app_name,
    version=__version__,
    lifespan=lifespan,
    # ...
)
```
-   **CORS**: We add `CORSMiddleware` to allow requests from any origin (`*`). In production, you'd restrict this to your specific frontend domain.
-   **Static Files**: `app.mount("/static", ...)` allows serving the simple `index.html` frontend directly from the `static/` folder.

## 4. Router Integration

This is where we plug in the endpoints defined in other files.

```python
app.include_router(health.router)
app.include_router(documents.router)
app.include_router(query.router)
```
Any new features (e.g., User Management) would need their router included here to be active.

## 5. Global Exception Handler

```python
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled exception: {exc}", exc_info=True)
    return JSONResponse(status_code=500, ...)
```
**Safety Net**: If *any* part of the code raises an error that isn't caught, this handler catches it, logs the full stack trace (vital for debugging), and returns a clean "Internal Server Error" JSON response to the user instead of crashing the server or leaking raw traceback data.

## 6. Running the App

```python
if __name__ == "__main__":
    uvicorn.run("app.main:app", ...)
```
This block allows you to run the app directly with `python app/main.py`. However, in production, we typically use the command `uvicorn app.main:app` externally.
