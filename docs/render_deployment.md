# Deploying to Render.com

Yes, this project is perfectly suited for **Render.com** (Web Service). It uses standard Python/FastAPI patterns.

## Prerequisites
1.  A [Render.com](https://render.com) account.
2.  Your GitHub repository connected to Render.
3.  **API Keys**: You will need your `OPENAI_API_KEY`, `QDRANT_URL`, and `QDRANT_API_KEY` ready.
4.  **Important**: Since we use **Qdrant Cloud** (external database), you don't need to host a database on Render. The app just connects to it.

## Method 1: Blueprint (Recommended)

1.  Create a file named `render.yaml` in the root of your repository (content provided below).
2.  Push it to GitHub.
3.  In Render dashboard, click **"New +"** -> **"Blueprint"**.
4.  Select your repo. Render will automatically detect the configuration.

## Method 2: Manual Setup

1.  Click **"New +"** -> **"Web Service"**.
2.  Connect your `rag-qa-project` repo.
3.  **Settings**:
    -   **Name**: `rag-qa-app`
    -   **Runtime**: `Python 3`
    -   **Build Command**: `pip install -r requirements.txt`
    -   **Start Command**: `uvicorn app.main:app --host 0.0.0.0 --port 10000`
4.  **Environment Variables**:
    -   Click "Advanced" -> "Add Environment Variable".
    -   Add `OPENAI_API_KEY`, `QDRANT_URL`, `QDRANT_API_KEY`.
    -   Add `PYTHON_VERSION` = `3.9.18` (or your preferred version).

## Sample `render.yaml`

Save this file as `render.yaml` in your project root.

```yaml
services:
  - type: web
    name: rag-qa-app
    env: python
    buildCommand: pip install -r requirements.txt
    startCommand: uvicorn app.main:app --host 0.0.0.0 --port 10000
    envVars:
      - key: PYTHON_VERSION
        value: 3.9.18
      - key: OPENAI_API_KEY
        sync: false  # You will enter this in the dashboard
      - key: QDRANT_URL
        sync: false
      - key: QDRANT_API_KEY
        sync: false
```

## Post-Deployment Issues
-   **Health Check**: Render verifies if your app is up. If it fails, check the "Logs" tab.
-   **Port**: Render automatically sets the `PORT` env var (usually 10000). Uvicorn must listen on `0.0.0.0`.
