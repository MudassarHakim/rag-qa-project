# RAGAS Evaluator Module Explained

This document details the `app/core/ragas_evaluator.py` module, which uses the **RAGAS** (Retrieval Augmented Generation Assessment) library to measure the quality of our RAG responses.

## 1. What is RAGAS?

RAGAS provides metrics to evaluate RAG pipelines *without* needing human-labeled "ground truth" answers. It uses an LLM (metrics-as-judge) to score:
1.  **Faithfulness**: Is the answer derived *only* from the retrieved context? (Hallucination check).
2.  **Answer Relevancy**: Does the answer actually address the user's question?

## 2. Initialization: `RAGASEvaluator`

```python
class RAGASEvaluator:
    def __init__(self):
        # ... logic ...
        self.metrics = [faithfulness, answer_relevancy]
```

-   **Separate Models**: We can configure a different LLM for evaluation than for generation (e.g., use GPT-4o for evaluation even if generation uses GPT-3.5) via settings like `ragas_llm_model`.
-   **Metrics**: We initialize the specific metrics we want to track.

## 3. The Evaluation Process: `aevaluate`

This is the main entry point. It runs asynchronously to avoid blocking the main API thread.

```python
async def aevaluate(self, question, answer, contexts) -> dict:
    # 1. Prepare Data
    dataset = self._prepare_dataset(question, answer, contexts)

    # 2. Run Evaluation (in thread pool)
    result = await asyncio.to_thread(self._evaluate_with_timeout, dataset)

    # 3. Return Scores
    return {
        "faithfulness": 0.95,
        "answer_relevancy": 0.88,
        ...
    }
```

### Key Components

1.  **Dataset Preparation (`_prepare_dataset`)**:
    RAGAS requires a HuggingFace `Dataset` format. We convert our raw strings into this format:
    ```python
    data = {
        "question": [question],
        "answer": [answer],
        "contexts": [[context_1, context_2]] 
    }
    ```

2.  **Thread Pool (`asyncio.to_thread`)**:
    The core `evaluate()` function from RAGAS is synchronous (blocking). If we ran it directly in an `async def` function, it would freeze the entire FastAPI server for several seconds. `asyncio.to_thread` offloads it to a separate thread, keeping the API responsive.

3.  **Error Handling**:
    If evaluation fails (e.g., OpenAI API timeout), we catch the exception and return `None` for scores instead of crashing the user's request. This ensures the user still gets their answer even if the background evaluation fails.

## Usage Example

```python
from app.core.ragas_evaluator import RAGASEvaluator

evaluator = RAGASEvaluator()

# Typically called after generating an answer
scores = await evaluator.aevaluate(
    question="What is the capital of France?",
    answer="The capital is Paris.",
    contexts=["Paris is the capital and most populous city of France."]
)

print(scores["faithfulness"]) # e.g., 1.0
```

## Metrics Explained

### Faithfulness (0.0 - 1.0)
-   **High Score**: The answer is fully supported by the retrieved contexts.
-   **Low Score**: The answer contains claims not found in the source documents (hallucinations).

### Answer Relevancy (0.0 - 1.0)
-   **High Score**: The answer directly addresses the prompt.
-   **Low Score**: The answer is vague, repetitive, or off-topic.
