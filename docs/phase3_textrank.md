# Phase 3 — Extractive baseline (TextRank)

## Why this is the first thing you ship

Before any neural network, you need a **floor**: a method so simple it can't
possibly hallucinate, against which everything fancy will be measured. That's
the role of the extractive baseline. If your LLM with RAG doesn't beat
TextRank on factual fidelity, then the RAG isn't doing what you say it does.

## What TextRank does

1. Split the article into sentences.
2. Build a graph: one node per sentence, edge weights = similarity between pairs.
3. Run PageRank — sentences that are similar to many other important
   sentences get a high score.
4. Pick the top N sentences, restore them in document order, concatenate.

The result is a summary made entirely of original sentences from the article.
**Zero hallucination by construction.**

We implement two variants:

| Variant | Similarity | Pros | Cons |
|---|---|---|---|
| `tfidf` | TF-IDF + cosine | Fast, no model download, baseline historique | Misses paraphrases |
| `embeddings` | sentence-transformers (MiniLM) + cosine | Captures semantic similarity | Slower, ~80 MB model download |

Both variants will appear side by side in the final benchmark table.

## Run it

Install the extra library needed for the embeddings variant:
```bash
pip install sentence-transformers
```

Then, from the project root:

```bash
# Variant 1: classic TF-IDF
python scripts/run_textrank.py --method tfidf

# Variant 2: sentence embeddings
python scripts/run_textrank.py --method embeddings
```

Each run takes ~30s to 2 min for 100 documents. Output files:

```
data/processed/predictions_textrank_tfidf.jsonl
data/processed/predictions_textrank_embeddings.jsonl
```

## The shared prediction format

Every approach in this project writes predictions in the **same JSONL format**:

```json
{
  "id": "pubmed_000",
  "approach": "textrank_tfidf",
  "prediction": "...generated summary...",
  "reference": "...gold abstract...",
  "elapsed_seconds": 0.42
}
```

This is critical: Phase 6 (evaluation) doesn't care which approach produced
the file — it loads any prediction file matching this schema and computes
ROUGE / BERTScore / faithfulness. So the final benchmark table is generated
by a single script that takes all five prediction files as input.

## Sanity checks

```bash
# Verify the output has exactly 100 lines (one per eval doc)
wc -l data/processed/predictions_textrank_tfidf.jsonl

# Peek at one prediction
python -c "import json; print(json.dumps(json.loads(open('data/processed/predictions_textrank_tfidf.jsonl').readline()), indent=2)[:1000])"
```

## What to put in the report about Phase 3

- The algorithm: PageRank on a sentence-similarity graph.
- The two variants and what differs.
- A worked example on **one** real PubMed article: show the input (truncated),
  the TF-IDF summary, the embeddings summary, and the gold reference. Comment
  on what each variant captured or missed. This single qualitative example
  is worth more than five paragraphs of theory.
- Generation time per document, averaged across the 100-doc sample (already
  in the `elapsed_seconds` field).
- ROUGE/BERTScore numbers (computed in Phase 6 — you'll come back to this
  section to fill them in).

## When Phase 3 is done

You have **two prediction files** in `data/processed/`, each containing
100 lines. You're ready for Phase 4 — the LLM + prompt approach.
