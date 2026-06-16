# DECISIONS.md

> NOTE TO SELF: this file is graded on honesty, not polish. Fill in real
> numbers and real failures from YOUR runs. Typos are fine. Do not let an AI
> "clean up" this file -- the reviewers say they can tell, and a polished
> version of this reads as fabricated.

## 1. Three things I tried that didn't work

1. **Fixed-size token chunking (512 tokens, no overlap-aware splitting)**
   - [FILL IN: what you observed -- e.g. citations pointed to "chunk 14"
     with no meaningful boundary, retrieval pulled half-sentences, etc.]
   - Why it failed: [FILL IN]

2. **[FILL IN: e.g. hard rerank-score cutoff for refusal]**
   - What I tried: [FILL IN]
   - Why it failed: [FILL IN -- e.g. ms-marco cross-encoder scores aren't
     calibrated to a fixed threshold across query types, false positives on
     short factual questions]

3. **[FILL IN: e.g. asking the LLM to output a confidence score 0-100]**
   - What I tried: [FILL IN]
   - Why it failed: [FILL IN -- e.g. scores were not correlated with actual
     correctness, model was overconfident on hallucinated answers too]

## 2. Chunking strategy and its failure mode

Heading-based (H1-H3) sections, oversized sections split on paragraph
boundaries with ~100 token overlap. See `ingest/chunk.py` docstring.

**Remaining failure mode**: [FILL IN -- e.g. reference pages that are
mostly tables get chunked with little surrounding context; give a concrete
example question from your eval set where this caused a miss]

## 3. How the refusal mechanism works, and where it's wrong

Two-stage retrieval (FAISS -> cross-encoder rerank) feeds top-5 chunks to a
single LLM call that must classify the question as answerable /
unanswerable / false_premise BEFORE answering, using only the provided
context (explicit instruction not to use training knowledge).

**Category it still gets wrong**: [FILL IN -- e.g. false-premise questions
about topics with zero retrieval coverage get misclassified as
"unanswerable" because the model has no evidence to confirm the premise is
wrong. Give the specific eval question ID(s).]

## 4. One more week + $500/month

[FILL IN -- e.g.:
- Swap reranker for a larger model / add a calibrated confidence score
  trained on labeled retrieval data
- Add a second-pass verifier call for answerable questions (cheap model
  checks the answer against citations before returning)
- Expand corpus coverage / handle table-heavy reference pages differently
- Add caching for repeated/similar queries to cut cost
- etc.]

## 5. Shortcut taken due to the 24h limit

[FILL IN -- e.g. did not hand-tune the rerank threshold per category, used
a single system prompt for all three verdict types instead of
category-specific prompts, citation matching in eval is substring-based
rather than verified against actual doc anchors, etc.]
