# Logging Configuration Explained

This document provides a detailed breakdown of the `app/utils/logger.py` module, explaining how it works and how to use it effectively.

## 1. Imports

```python
import logging
import sys
from functools import lru_cache
```

-   **`logging`**: The standard Python logging library. It's thread-safe and highly configurable.
-   **`sys`**: Used here to access `sys.stdout` (standard output) so logs print to the console.
-   **`functools.lru_cache`**: A decorator that caches the results of a function. "LRU" stands for Least Recently Used. It's used here to store logger instances so we don't re-create them repeatedly, slightly improving performance.

---

## 2. Setting Up Logging: `setup_logging`

This function is responsible for the global logging configuration. It should be called once at the application startup (e.g., in `main.py`).

```python
def setup_logging(log_level: str = "INFO") -> None:
    # ... configuration code ...
```

### Key Components:

1.  **Formatter**:
    ```python
    formatter = logging.Formatter(
        fmt="[%(asctime)s] [%(levelname)s] [%(name)s] %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    )
    ```
    This defines *how* the logs look.
    -   `%(asctime)s`: Timestamp (e.g., `2025-12-14 09:30:00`).
    -   `%(levelname)s`: Severity (e.g., `INFO`, `ERROR`).
    -   `%(name)s`: Name of the logger (e.g., `app.core.document_processor`).
    -   `%(message)s`: The actual log message.

    **Example Output:**
    ```text
    [2025-12-14 09:30:00] [INFO] [app.main] Application started successfully
    ```

2.  **Root Logger & Handlers**:
    ```python
    root_logger = logging.getLogger()
    root_logger.setLevel(...)

    # Remove existing handlers
    for handler in root_logger.handlers[:]:
        root_logger.removeHandler(handler)

    # Console handler
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(formatter)
    root_logger.addHandler(console_handler)
    ```
    -   **Root Logger**: The parent of all loggers. Settings applied here (like the level) propagate down.
    -   **Handlers**: Destinations for logs. We use `StreamHandler(sys.stdout)` to print logs to the terminal (useful for Docker/Kubernetes logs).
    -   **Cleanup**: We remove existing handlers to prevent duplicate logs (e.g., if `setup_logging` is called twice or if uvicorn set up its own).

3.  **Noise Reduction**:
    ```python
    logging.getLogger("httpx").setLevel(logging.WARNING)
    # ... others ...
    ```
    Libraries like `httpx` or `openai` can be very chatty at `INFO` level. We set them to `WARNING` so we only see their logs if something goes wrong, keeping our main log stream clean.

### Usage Example:

In your `main.py`:

```python
from app.utils.logger import setup_logging
from app.config import get_settings

settings = get_settings()
setup_logging(log_level=settings.log_level)
```

---

## 3. Getting a Logger: `get_logger`

```python
@lru_cache
def get_logger(name: str) -> logging.Logger:
    return logging.getLogger(name)
```

This is a helper to get a logger instance.

-   **Why `@lru_cache`?**: `logging.getLogger(name)` is already efficient, but adding `@lru_cache` makes repeated calls for the same name almost instant by bypassing the logging library's internal lookup logic entirely. It's a micro-optimization.
-   **Naming Convention**: Usually passed `__name__`, which evaluates to the module path (e.g., `app.api.routes.query`). This creates a hierarchy.

### Usage Example:

In `app/services/my_service.py`:

```python
from app.utils.logger import get_logger

# __name__ will be 'app.services.my_service'
logger = get_logger(__name__)

def do_something():
    logger.info("Starting operation...")
    try:
        # ... code ...
        logger.debug("Operation details...")
    except Exception as e:
        logger.error(f"Operation failed: {e}")
```

---

## 4. The Mixin Pattern: `LoggerMixin`

```python
class LoggerMixin:
    @property
    def logger(self) -> logging.Logger:
        return get_logger(self.__class__.__name__)
```

A **Mixin** is a class designed to add specific functionality to other classes through inheritance. This specific mixin adds a `self.logger` property to any class that inherits from it.

-   **Automatic Naming**: It uses `self.__class__.__name__` as the logger name. So if you have `class DocumentProcessor(LoggerMixin)`, the logger name will be `DocumentProcessor`.

### Usage Example:

Without Mixin:
```python
class PDFLoader:
    def __init__(self):
        self.logger = get_logger("PDFLoader") # Manual string or __name__

    def load(self):
        self.logger.info("Loading PDF")
```

With Mixin:
```python
from app.utils.logger import LoggerMixin

class PDFLoader(LoggerMixin):
    def load(self):
        # self.logger is automatically available!
        self.logger.info("Loading PDF")
```

### When to use which?

-   **`get_logger(__name__)`**: Best for **module-level** functions or when you want the logger name to reflect the file structure (e.g., `app.core.rag_chain`). This is generally preferred for debugging as it tells you exactly *where* in the project the code is.
-   **`LoggerMixin`**: Best for **class-based** logic where you think of the "Component" logging (e.g., `VectorStore creates a log`). The logger name will be just the class name (e.g., `VectorStoreService`), losing the module path context, but it's cleaner to write inside the class.

---

## Best Practices Summary

1.  **Call `setup_logging` once** at the application entry point.
2.  **Use specific levels**:
    -   `DEBUG`: Detailed info for diagnosing problems.
    -   `INFO`: Confirmation that things are working as expected.
    -   `WARNING`: Something unexpected happened, but the software is still working.
    -   `ERROR`: Due to a more serious problem, the software has not been able to perform some function.
3.  **Don't print**: Never use `print()` in production code. Use `logger.info()` instead. Logs can be structured, filtered, and sent to aggregation systems (like Datadog, CloudWatch, or LangSmith); prints cannot.
