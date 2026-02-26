# Engineering Conversational RAG Agent - Developer Guide

This project is an AI assistant platform designed to help teams find reliable answers quickly from large collections of internal documents. Instead of giving generic chatbot responses, it reads trusted sources, explains answers in plain language, and can show where information came from. It is built to support many users at once, continue conversations naturally, and stay dependable even when one AI provider is unavailable by automatically switching to backups. The system is also packaged for easy local use, monitored for performance, and structured so knowledge updates can be tracked and reproduced over time.
<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/6abb214e-17f8-43ad-ab47-c473dd628a85" />

This project was deployed on revit with wpf and on Nextjs.
## What Is Implemented

- High-concurrency WebSocket serving with per-connection and per-IP limits (`server/connection_manager.py`).
- Multi-step agent workflow in `agents/<agent_module>/workflow.py`:
  - Strategy step (`safe / category / think_hard`)
  - Router step (`vector RAG / BM25 / no RAG`)
  - ReAct path for complex queries
  - Concept-design path with JSON validation loop and repair
- Hierarchical RAG stack in `agents/<agent_module>/components.py`:
  - Hierarchical chunking (`4096, 2048, 1024`)
  - Auto-merging retriever + BM25 retriever
  - Optional Qdrant vector store support
  - GraphRAG component class and optional request-level switch (`use_graph_rag`)
- Provider-agnostic model layer in `agents/<agent_module>/shared_models.py`:
  - LLM providers: `google`, `cerebras`, `openrouter`, `test`
  - Embedding providers: `ollama`, `vllm`
  - Factory-based model selection and instance caching
  - Fallback chains with quota/rate-limit detection and retry/backoff
- Observability:
  - Prometheus metrics endpoint at `/metrics`
  - Runtime monitor for RAM/VRAM tracking (`monitor/monitor.py`)
- Data lineage:
  - DVC-tracked artifacts (`data.dvc`, `index.dvc`)
- Packaging/deployment:
  - Docker Compose setup (`compose.yaml`)
  - PyInstaller standalone packaging (`../build3.ps1`)

## Project Structure

- `server/`: FastAPI app, config, validation models, connection/session lifecycle.
- `agents/`: RAG components, workflow orchestration, prompt templates, provider/embedding factories.
- `monitor/`: runtime memory monitor + Prometheus scrape config.
- `tests/`: unit tests, concurrency/load script, dataset/evaluation helpers.
- `src/`: dataset generation utilities for RAG evaluation.
- `compose.yaml`: local `fastapi + ollama` stack.
- `main.py`: server entrypoint; optionally starts Ollama and Prometheus subprocesses.

## Prerequisites

- Python 3.10+
- `pip`
- Optional:
  - Ollama (local embedding/runtime)
  - vLLM-compatible GPU setup (if using `vllm` embeddings)
  - Docker + Docker Compose
  - DVC (for pulling tracked `data/` and `index/`)

## Configuration

Settings are loaded from:
1. environment variables
2. `server_config.json` (resolved from executable/project base path)

Main config file: `dev/server_config.json`

Important fields:
- `persist_dir` (default in config: `mock_index`)
- `dataset_dir` (default in config: `mock_data`)
- `prompts_file` (`agents/<agent_module>/prompts.yaml`)
- connection and message limits (`max_connections`, `max_connections_per_ip`, etc.)

## Running Locally

1. Install dependencies:
```bash
pip install -r requirements.txt
```

2. Start server:
```bash
python main.py
```

3. Optional dev mode:
```bash
python main.py --dev
```

Server default: `http://0.0.0.0:8000`

## Docker Compose

Run backend + Ollama:
```bash
docker-compose -f compose.yaml up --build
```

Endpoints:
- FastAPI: `http://localhost:8000`
- Ollama: `http://localhost:11434`

## API Endpoints

### HTTP

- `GET /`: service info + endpoint summary
- `GET /health`: health + connection stats
- `GET /health/ready`: readiness check
- `POST /initialize`: initialize workflow for a user/provider
- `GET /check_providers`: available providers + setup guides
- `GET /metrics`: Prometheus metrics

`POST /initialize` body:
```json
{
  "key": "YOUR_PROVIDER_API_KEY",
  "provider": "google",
  "username": "alice"
}
```

### WebSocket

- `WS /run_ws`

Request payload:
```json
{
  "query": "Your question",
  "username": "alice",
  "use_graph_rag": false
}
```

Streamed response event types:
- `token`
- `log`
- `sources`
- `error`
- `qouta_error`
- `done`

## DVC Data/Index Reproducibility

This repo tracks vector index/data artifacts via DVC:
- `dev/data.dvc` -> `data/`
- `dev/index.dvc` -> `index/`

Typical workflow:
```bash
dvc pull
```

## Build and Portable Packaging

Use PyInstaller script from project root:
```powershell
.\build3.ps1
```

Current packaging behavior:
- Core app is built as `installers/chatbot_installer/<app_name>/<app_name>.exe`
- `index`, `data`, `mock_index`, and `mock_data` are copied post-build into `dist` output
- `prompts.yaml` and `server_config.json` are explicitly copied


## Testing and Evaluation

### Unit tests
```bash
pytest tests/unit_tests
```

### Concurrency/load simulation
```bash
python tests/test_concurrency.py
```

### RAG evaluation scripts
- `tests/rag_test.py`: RAGAS `answer_correctness` and `context_recall.with_params(k=5)` (Recall@5)
- `src/rag_dataset.py`: dataset generation pipeline for RAG evaluation

## Notes

- Workflows are user-scoped (`username`) and must be initialized before use.
- Global embeddings are configured during lifespan startup; LLM selection is done per workflow via provider factories.
- GraphRAG support exists but vector RAG remains the default operational path.
