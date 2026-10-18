# RAG API

A small retrieval-augmented generation service: drop text files in `docs/`, ask a question over HTTP, get an answer grounded in those files. Everything runs locally. No API keys, no hosted model.

The interesting part isn't the model. It's that the RAG quality is tested in CI, so editing a document that breaks an answer fails the build.

## How it works

```
docs/*.txt ──► embed_docs.py ──► Chroma (./db)
                                     │
POST /query?q=... ──► FastAPI ──► top match ──► Ollama (tinyllama) ──► answer
```

1. `embed_docs.py` clears the collection and embeds every `.txt` file in `docs/` into a persistent Chroma store
2. `POST /query` retrieves the closest document and passes it to a local `tinyllama` model through Ollama as context
3. The answer comes back as JSON

## Run it

You'll need Python 3.11+ and [Ollama](https://ollama.com) running locally.

```bash
ollama pull tinyllama
pip install -r requirements.txt uvicorn
python embed_docs.py
uvicorn app:app --reload
```

```bash
curl -X POST "http://127.0.0.1:8000/query?q=What%20is%20Kubernetes?"
```

## Testing RAG in CI

LLM output isn't deterministic, which makes it hard to test. So CI runs the API with `USE_MOCK_LLM=1`: retrieval still happens for real, but the endpoint returns the retrieved context instead of calling the model. That makes the test about the part that can actually regress when someone edits `docs/`: does the right document come back for the right question?

The GitHub Actions workflow triggers on any change to `docs/`, `app.py` or `embed_docs.py`, rebuilds the embeddings, starts the API in mock mode and runs [`semantic_test.py`](semantic_test.py), which asserts on key concepts in the answers.

## Kubernetes

`deployment.yaml` and `service.yaml` run the API as a single-replica Deployment behind a NodePort Service, reaching Ollama on the host through `host.docker.internal:11434`. Built for a local cluster (Docker Desktop or Minikube) with a locally built `rag-app` image.

## Known issues

- The `Dockerfile` still copies `embed.py` and `k8s.txt` from before the move to `docs/`, so `docker build` currently fails. Fix: copy `docs/` and `embed_docs.py` instead.
- Retrieval returns only the single best match (`n_results=1`); multi-document context and chunking are the obvious next step.

## Tech

Python · FastAPI · Chroma · Ollama · tinyllama · Docker · Kubernetes · GitHub Actions
