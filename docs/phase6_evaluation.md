# Phase 6 — Evaluation

## Three metric families

Each one captures a different aspect of summary quality. The combination is
what makes the project rigorous — none alone is enough.

### 1. ROUGE-1 / ROUGE-2 / ROUGE-L
N-gram and longest-common-subsequence overlap between prediction and gold
reference. Standard in summarization research since 2004. Easy to compute,
easy to compare across papers. **But**: counts wording, not meaning.

### 2. BERTScore
Cosine similarity of contextual BERT embeddings between prediction and
reference, aggregated to a precision/recall/F1. Captures paraphrase that
ROUGE misses. Slower to compute (needs a forward pass through BERT) but
much more aligned with human judgment.

### 3. Faithfulness (custom)
**This is your novel contribution.** ROUGE and BERTScore measure
similarity to the gold reference — but a model that hallucinates a
plausible-sounding fact will still get high ROUGE if the hallucination is
phrased similarly to the reference.

The faithfulness metric ignores the reference and compares the prediction
to the **source article** directly. We extract three categories of risky
entities from the prediction:
- **Numbers and dosages** (e.g., "18 mg/L", "n=240", "p<0.05")
- **Drug names** (matched against a curated list of common drugs)
- **Uppercase abbreviations** (3-6 letters: ICU, COPD, MRI, ...)

For each entity in the prediction, we check whether it appears in the
source article. Faithfulness = supported / total. A score below 1.0
means the prediction introduces at least one piece of information not in
the source — a hallucination signal.

This metric is intentionally interpretable. For the report, the script
outputs the **list of unsupported entities per document**, which gives
you direct qualitative material:

> *"The structured prompt LLM hallucinated a dosage of 50 mg/day in 3 of
> 30 documents, all of which were absent from the source article."*

That kind of sentence wins points.

## How to run

### Install eval libraries

```bash
pip install rouge-score bert-score
```

### Run the full evaluation

```bash
python scripts/run_evaluation.py
```

The script auto-discovers any `predictions_*.jsonl` file in
`data/processed/`. It computes ROUGE, BERTScore, and faithfulness for
each, and outputs:

- A printed table to stdout (paste this into the report)
- `data/processed/evaluation_results.json` (raw numbers)
- `data/processed/evaluation_results.csv` (table format)

### Fair comparison mode

If one approach is missing a few documents (e.g., the LLM timed out on
some long articles), use `--fair-comparison` to restrict the evaluation
to the IDs present in ALL prediction files:

```bash
python scripts/run_evaluation.py --fair-comparison
```

This guarantees a strictly apples-to-apples comparison.

## How to read the results

### Expected pattern

In your final table you should typically see:

| Approach   | ROUGE-L | BERTScore | Faithfulness |
|------------|---------|-----------|--------------|
| TextRank   | mid     | mid       | ~1.0         |
| LLM only   | high    | high      | < 1.0        |
| LLM + RAG  | high    | high      | < 1.0?       |

TextRank should score near-perfect faithfulness by construction (it only
copies sentences from the source). The LLM approaches will sit below
that floor — that's the cost of generation. The interesting question is
**whether RAG helps or hurts faithfulness compared to LLM-only**.

### The key analysis to write up

For each approach, list 3-5 representative unsupported entities (from
the per-document logs). Discuss whether they are:
- **Genuine hallucinations**: the model invented a fact.
- **Paraphrases**: the fact is in the source but phrased differently
  (e.g., source says "twice daily", prediction says "BID" — they mean
  the same thing). These are limits of the string-match metric.
- **From the RAG context**: only relevant for the RAG approach — if a
  drug name appears in the prediction because it was in the retrieved
  knowledge base but NOT in the source article, that's a RAG-induced
  hallucination.

This 3-way analysis (genuine / paraphrase / RAG-induced) is exactly the
kind of nuanced discussion that distinguishes a B from an A.

## Limitations to acknowledge

- String matching misses semantic equivalence ("BID" vs "twice a day").
  A future improvement is using scispaCy NER with UMLS linking.
- The drug list is hand-curated and US/UK-centric.
- We treat all entities as equally risky, but a hallucinated dosage is
  much worse than a hallucinated abbreviation.
- The metric is monolingual (English).

Write these limitations into the report. Acknowledging them shows
methodological maturity.

## When Phase 6 is done

You have `evaluation_results.csv` with one row per approach and columns
for every metric. This table IS the headline result of your project.
Drop it into the report's "Results" section, and the bulk of the writing
is just narrating what the numbers say.
