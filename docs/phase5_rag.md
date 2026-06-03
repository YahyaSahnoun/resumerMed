# Phase 5 — LLM + RAG

## The core conceptual point (must be in the report)

RAG in summarization does NOT serve to retrieve the article's content —
the article is already in front of us. RAG injects **external medical
reference knowledge** so the LLM can interpret terms in context (normal
lab ranges, drug classes, disease definitions) without hallucinating.

Worded another way: RAG gives the model a small medical handbook to consult
**while** summarizing the article, not instead of it. The summary content
still comes from the article. The handbook only helps the model parse what
the article is saying.

This distinction is the analytical hook of your project: RAG can help by
contextualizing terms, but it can also **introduce new information absent
from the source**, which is itself a kind of hallucination. Your report
should measure both effects.

## Architecture

```
article → retrieve top-5 KB entries → RAG prompt → LLM → summary
```

- **Knowledge base**: 36 hand-curated entries in `data/knowledge_base/medical_kb.json`,
  covering lab values, vital signs, diseases, drug classes, drugs,
  procedures, methodology terms, and common abbreviations.
- **Vector store**: ChromaDB persistent client (no server), cosine similarity,
  `all-MiniLM-L6-v2` embeddings.
- **Retrieval**: top-5 entries per article. The query is the first 1500
  characters of the article (the tail of PubMed articles is mostly
  references and discussion — noise for retrieval).
- **Prompt**: identical structure to Phase 4 `structured`, but with a
  REFERENCE KNOWLEDGE block clearly delimited and explicit instructions
  about its role.

## Steps to run (local)

```bash
pip install chromadb sentence-transformers

# One-time: build the vector store
python -m src.rag.build_vector_store

# Smoke test on 3 documents
python scripts/run_rag.py --model phi3:mini --limit 3

# Full run
python scripts/run_rag.py --model phi3:mini
```

Output files:
- `data/processed/predictions_rag_<model>_top5.jsonl` — the predictions.
- `data/processed/retrieval_log_rag_<model>_top5.jsonl` — per-article log
  of which KB entries were retrieved. Use this for error analysis.

## Steps to run (Colab — much faster on T4 GPU)

Open `notebooks/medical_summarizer_pipeline.ipynb` in Colab, set runtime
to T4 GPU, upload `pubmed_eval_sample.jsonl`, and run the cells in order.
The whole pipeline (Phases 3 + 4 + 5) runs in ~25-40 minutes.

The notebook uses `transformers` with 4-bit quantization instead of Ollama
because Ollama doesn't fit Colab's environment cleanly. The prompts and
algorithm are otherwise identical.

## Ablation experiments to include in the report

These three comparisons are the analytical core of your project:

1. **LLM no-RAG vs LLM+RAG (same prompt, same model)**.
   - ROUGE: does RAG improve overlap with the gold abstract?
   - Faithfulness: does RAG introduce facts absent from the article?
2. **Retrieval quality**.
   - Show the retrieval log for 3-5 articles. Are the retrieved entries
     actually relevant to the article topic? Cite distances.
3. **Top-K sensitivity (bonus)**.
   - Re-run with `--top-k 3` and `--top-k 10` and report ROUGE+faithfulness.
     Often more is not better — too much retrieved context dilutes the prompt.

## Limitations to acknowledge in the report (this earns marks)

- The KB has only 36 entries. A real clinical KB has tens of thousands.
- The retriever uses general-purpose embeddings, not biomedical ones.
  Switching to `pritamdeka/S-PubMedBert-MS-MARCO` would be the obvious
  next step.
- We retrieve at the article level. Chunking the article and retrieving
  per chunk would be more granular but increases cost.
- We do not yet evaluate whether retrieved facts contradict the article
  (e.g., article says "creatinine 0.9" but retrieval says normal range
  is 0.7-1.3 — that's consistent, but the LLM might rephrase it
  incorrectly).

## When Phase 5 is done

You should have **five** prediction files in `data/processed/`:

- `predictions_textrank_tfidf.jsonl`
- `predictions_textrank_embeddings.jsonl`
- `predictions_llm_<model>_basic.jsonl`
- `predictions_llm_<model>_structured.jsonl`
- `predictions_rag_<model>_top5.jsonl`

All in the same JSONL schema. Phase 6 (evaluation) will compute ROUGE,
BERTScore, and faithfulness for each, side by side.
