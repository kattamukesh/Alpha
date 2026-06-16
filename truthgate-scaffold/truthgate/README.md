# TruthGate

A RAG QA system over the **Kubernetes documentation** that answers questions
with citations, refuses when the answer isn't in the docs, and detects
false-premise questions.

## Corpus

Official Kubernetes documentation, scraped from the `kubernetes/website`
GitHub repo (`content/en/docs/`), covering concepts, tasks, tutorials,
reference, and setup sections. ~[FILL IN: N] markdown files / ~[FILL IN: N]
chunks after processing.

## Architecture

```
scrape (GitHub markdown) -> chunk (heading-based) -> embed + FAISS index
                                                              |
question -> embed -> FAISS top-20 -> cross-encoder rerank top-5 -> LLM (classify + answer) -> JSON result
```

- **Chunking**: heading-based sections (H1-H3), oversized sections split on
  paragraph boundaries with overlap. See `ingest/chunk.py` for rationale
  and known failure mode.
- **Embeddings**: `BAAI/bge-small-en-v1.5` (local, CPU, free).
- **Vector store**: FAISS `IndexFlatIP` (exact cosine similarity over
  L2-normalized vectors) -- justified by corpus size in `ingest/build_index.py`.
- **Reranking**: `cross-encoder/ms-marco-MiniLM-L-6-v2` (local, free).
- **Generation**: [FILL IN: model used], single call per query, structured
  JSON output with verdict + answer + citations.

## Refusal & False-Premise Mechanism

See `core/pipeline.py` docstring and `DECISIONS.md` for the full design
rationale and known weak spots.

## Setup

```bash
make setup          # install deps, create .env
# edit .env and add your ANTHROPIC_API_KEY (or OPENAI_API_KEY)
make ingest          # scrape docs, chunk, build index (~10-15 min)
make eval            # run the 60-question eval harness
make run             # interactive CLI
```

Or with Docker:

```bash
docker compose up
```

## Eval Results

Run `make eval` to regenerate. Latest numbers (run on [FILL IN: date]):

| Metric | Value |
|---|---|
| Unanswerable recall | [FILL IN] |
| Unanswerable precision | [FILL IN] |
| False-premise detection rate | [FILL IN] |
| Answerable verdict rate | [FILL IN] |
| Citation match rate | [FILL IN] |
| Mean cost per query | [FILL IN] |
| p95 latency | [FILL IN] |

Full per-question results: `eval/results.json`.

## Cost & Latency Budget

- Target: <= $0.02 average cost/query, p95 latency <= 8s.
- Actual: [FILL IN -- be honest, this is graded on honesty]

## Project Structure

```
truthgate/
├── ingest/          # scraper, chunker, index builder
├── retrieval/       # FAISS + reranker
├── core/            # QA pipeline (classify + answer)
├── eval/            # 60 hand-written questions + harness
├── main.py          # CLI entrypoint
├── DECISIONS.md      # design decisions, failures, tradeoffs
└── Makefile
```

## Favorite Eval Question

[FILL IN: one line -- which question and what it taught you]
