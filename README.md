Here's the README filled in properly — but leave the metrics table blank until you actually run the eval. I've filled everything else.

TruthGate
A RAG-based question answering system over the Kubernetes official documentation that answers questions with source citations, refuses to answer when the information isn't in the docs, and detects questions built on false premises.
Built as a take-home assessment. The core challenge is not retrieval — it's knowing when to shut up.

Corpus
Official Kubernetes documentation scraped from the kubernetes/website GitHub repo (content/en/docs/), covering:

Concepts
Tasks
Tutorials
Reference
Setup

Scraped as raw markdown (not rendered HTML) to avoid nav/footer noise and preserve natural heading structure for citation.

Architecture
GitHub markdown → chunk by headings → embed (local) → FAISS index
                                                            ↓
question → embed → FAISS top-20 → cross-encoder rerank top-5 → LLM → verdict + answer + citations
Every component choice is justified below.
Chunking — heading-based, not fixed-window
Split documents on H1/H2/H3 headings. Each chunk = one section, with metadata carrying file_path, heading_path, and url. Oversized sections (>800 tokens) are split on paragraph boundaries with 100-token overlap.
Why not fixed-window sliding chunks (the default in most tutorials)? Fixed windows split mid-concept constantly, citations point to meaningless positions ("paragraph 14"), and retrieval quality is worse because chunk boundaries don't align with semantic units. Fixed-window was the v1 approach — see DECISIONS.md.
Embeddings — local, free
BAAI/bge-small-en-v1.5 (384-dim, ~130MB, CPU-friendly). Strong performance on MTEB retrieval benchmarks relative to its size. No API cost, no latency added by a remote call.
Vector store — FAISS flat index
IndexFlatIP (exact inner product on L2-normalized vectors = cosine similarity). Corpus is 2,000–4,000 chunks — exact search is fast (<50ms on CPU) at this size. Approximate indexing (HNSW, IVF) buys nothing here and adds tuning complexity.
Reranking — local cross-encoder
cross-encoder/ms-marco-MiniLM-L-6-v2. The bi-encoder (embedding model) scores query and document independently — fast but coarse. The cross-encoder takes (query, document) as a single input and scores them jointly, which is significantly more accurate. Too slow to run over the full corpus, hence the two-stage funnel: cheap recall (FAISS top-20) → expensive precision (reranker top-5).
Generation — small paid LLM
Claude Haiku (or Gemini Flash / GPT-4o-mini). Single call per query. Forced JSON output with verdict, answer, citations. Cheap enough to stay well under the $0.02/query budget target.
Refusal & false-premise detection
The LLM is instructed to follow three steps in order before answering:

Check the premise — does the question assume something the docs contradict? → false_premise
Check coverage — do the retrieved chunks actually support an answer, or are they just topically related? → unanswerable
Answer — only if steps 1 and 2 are clear → answerable with citations

This lives entirely in the system prompt + JSON output format. The model is explicitly told not to use its training knowledge to fill gaps. See core/pipeline.py for the full prompt and known failure modes.

Setup
bash# 1. Clone and install
git clone <your-repo-url>
cd truthgate
make setup

# 2. Add your API key
cp .env.example .env
# edit .env — paste your ANTHROPIC_API_KEY (or GEMINI_API_KEY)

# 3. Scrape, chunk, and index the corpus (~10-15 min first run)
make ingest

# 4. Ask a question
python main.py "How does a Kubernetes Deployment manage Pod replicas?"

# 5. Interactive mode
python main.py --interactive

# 6. Run the full eval harness
make eval
Or with Docker (no local Python setup needed):
bashdocker compose up

Project Structure
truthgate/
├── ingest/
│   ├── scrape.py        # scrapes k8s docs markdown from GitHub
│   ├── chunk.py         # heading-based chunker
│   └── build_index.py   # embeds chunks + builds FAISS index
├── retrieval/
│   └── retriever.py     # FAISS search + cross-encoder reranker
├── core/
│   └── pipeline.py      # main QA pipeline: classify + answer + log cost/latency
├── eval/
│   ├── questions.json   # 60 hand-written eval questions
│   ├── run_eval.py      # eval harness
│   ├── results.json     # per-question outputs (generated, gitignored until run)
│   └── metrics.json     # summary metrics table (generated)
├── data/                # corpus + index (gitignored, rebuilt via make ingest)
├── main.py              # CLI entrypoint
├── Makefile
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── DECISIONS.md
└── README.md

Eval Results

Run make eval to regenerate. Numbers below are from the committed run.

MetricValueUnanswerable recall—Unanswerable precision—False-premise detection rate—Answerable verdict rate—Citation match rate—Mean cost per query (USD)—p95 latency (s)—
Full per-question outputs: eval/results.json

Budget

Target: ≤ $0.02 average cost/query, p95 latency ≤ 8s
Actual: (fill in after eval run)
Embeddings and reranking are local (free). Only LLM generation hits the API.


What It Gets Wrong
Documented honestly in DECISIONS.md. Short version:

False-premise questions about topics absent from retrieved chunks get misclassified as unanswerable — the model can't flag a wrong assumption it has no evidence about
Reference pages that are mostly tables (e.g. kubectl flag lists) chunk poorly and hurt retrieval for "what does --flag X do" questions
The eval's citation matching is substring-based, not verified against actual doc anchors — it's a proxy, not ground truth


Favorite Eval Question
(fill in after writing your 60 questions — one line: which question and what it taught you)

DECISIONS.md
See DECISIONS.md for: three things that failed, chunking failure modes, how refusal works and where it breaks, what I'd do with more time, and one documented shortcut.
