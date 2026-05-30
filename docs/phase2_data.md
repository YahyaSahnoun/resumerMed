# Phase 2 — Data preparation

## Strategy

You downloaded many datasets — that's good defensive thinking, but for this
project we focus on **one primary dataset** to ship something complete.

**Primary**: `PubMed` (ccdv/pubmed-summarization). Article → abstract pairs.
**Secondary** (used later for generalization analysis): `Asclepius` clinical notes.
**Qualitative analysis only**: `MTSamples`.

Everything else (CAS, BioASQ, MEDIQA-AnS, MS2, MeQSum) is set aside — we
mention them in the report as "considered but out of scope to stay focused".

## Steps to run

Make sure your venv is activated. Then:

### 1. Place the data

Put your downloaded PubMed parquet files at:
```
data/raw/PubMed/
    test.parquet
    validation.parquet
    train-0.parquet
    train-1.parquet
    ...
```

We only need `test.parquet` for the benchmark. The train files can stay
there or be deleted to save space.

### 2. Verify the loader works

```bash
python src/preprocessing/load_pubmed.py data/raw/PubMed
```

You should see "Loaded N test examples" with a sample article and abstract
printed. If you get a column-name error, paste the error here and we'll
adapt the loader.

### 3. Compute corpus statistics

These numbers go in the report (section "Dataset").

```bash
pip install matplotlib    # one-time, for the histogram
python scripts/explore_pubmed.py data/raw/PubMed
```

Outputs:
- `data/processed/pubmed_stats.json` — exact numbers to cite
- `data/processed/pubmed_length_distribution.png` — figure for the report

### 4. Build the unified evaluation sample

This is the **most important** step. It produces the single input file all
three approaches will consume in Phases 3, 4, 5.

```bash
python scripts/build_eval_sample.py data/raw/PubMed
```

Output:
- `data/processed/pubmed_eval_sample.jsonl` — 100 documents with a fixed
  random seed, each with an `id`, `article`, and `reference_summary`.

Why JSONL? One record per line, robust to interrupted writes, easy to
inspect with `head` or `wc -l`, supported natively by pandas (`read_json(lines=True)`).

## Why exactly 100 documents?

Mistral on CPU produces a summary in ~30 seconds. With 100 documents and
3 approaches to compare, the full benchmark takes about 100 × 30s × 3 ≈
**2.5 hours of compute**, which is reasonable to re-run a couple of times
during development.

If your machine is faster, you can later bump `SAMPLE_SIZE` in
`build_eval_sample.py` to 200 or 500 for a tighter benchmark — but **only
after** the full pipeline works end to end on 100.

## Sanity checks before moving on

Run these three commands and verify the outputs make sense:

```bash
wc -l data/processed/pubmed_eval_sample.jsonl   # should print 100
head -c 500 data/processed/pubmed_eval_sample.jsonl   # peek at first record
cat data/processed/pubmed_stats.json   # see corpus stats
```

When all three look good, Phase 2 is done.
