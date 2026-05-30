# Phase 4 — LLM + prompt (no RAG yet)

## Why no RAG yet

The whole point of Phase 5 (RAG) is to show that retrieval *improves* what
the LLM does on its own. If you start with RAG enabled, you can't prove
the improvement comes from retrieval rather than from the prompt or the
model itself. So:

| Phase | Approach | What it isolates |
|---|---|---|
| 3 | TextRank (no LLM) | Hallucination floor (zero by construction) |
| 4 | LLM alone | What the model knows without help |
| 5 | LLM + RAG | Marginal value of the medical knowledge base |

Each phase adds exactly one moving part.

## The prompt is the whole game

The prompt template is in `src/llm/prompts.py`, not buried in the runner
script. That's deliberate — prompts are version-controlled artifacts that
need to be discussed in the report.

Two variants are shipped:

**`basic`** — minimal instruction:
> "Summarize the following medical article in about 200 words."

No structure constraint, no anti-hallucination guardrails. This is the
"out of the box" baseline for the LLM.

**`structured`** — explicit guardrails + section structure:
- System prompt forbids inventing facts, numbers, or conclusions absent
  from the article.
- User prompt requires four sections (Background / Methods / Results /
  Conclusions), matching PubMed abstract structure.

Comparing these two variants gives you a **mini ablation study** in the
report: "switching from the basic to the structured prompt reduced
hallucinated entities by X% with no ROUGE penalty" — that kind of sentence
is exactly what an evaluator wants to see.

## Determinism

We pin `temperature=0.2` and `seed=42`. Re-running the same article should
produce the same summary (modulo tiny floating-point differences across
Ollama versions). Mention this in the report — it makes your numbers
reproducible.

## Run it

Make sure the Ollama daemon is running and the `mistral` model is pulled.

```bash
# Quick smoke test (5 documents only, ~2-3 min)
python scripts/run_llm.py --prompt basic --limit 5

# Inspect what came out
head -c 800 data/processed/predictions_llm_mistral_basic.jsonl
```

If the smoke test produces sensible-looking summaries, run the full thing:

```bash
# Variant 1: basic prompt (~50 min on CPU)
python scripts/run_llm.py --prompt basic

# Variant 2: structured prompt (~50 min on CPU)
python scripts/run_llm.py --prompt structured
```

Total compute for Phase 4: about 2 hours. You can leave it running
overnight. The script writes after every document and supports resume —
if you Ctrl-C and re-run, it skips the IDs already done.

## On smaller machines

If `mistral` is too heavy for your machine (slow generation, swapping):

```bash
ollama pull phi3:mini
python scripts/run_llm.py --prompt structured --model phi3:mini --limit 5
```

`phi3:mini` is ~2 GB and runs about 3x faster. Quality drops a bit but
it's still usable for a benchmark. Note the model name in the report.

## Sanity checks before moving on

```bash
# Both prediction files should have 100 lines
wc -l data/processed/predictions_llm_mistral_basic.jsonl
wc -l data/processed/predictions_llm_mistral_structured.jsonl

# Look at one example
python -c "
import json
with open('data/processed/predictions_llm_mistral_structured.jsonl') as f:
    rec = json.loads(f.readline())
print('--- ARTICLE (first 500 chars) ---')
# (article isn't in the prediction file; you'd join with the eval sample to see it)
print('--- GENERATED SUMMARY ---')
print(rec['prediction'])
print()
print('--- GOLD REFERENCE ---')
print(rec['reference'])
"
```

Read 3-4 of these by hand. Note any hallucination you spot (a number, a
finding, a drug name in the prediction that isn't in the article). Write
them down — these become qualitative examples for the report.

## What goes in the report about Phase 4

- The two prompt variants, quoted verbatim from `src/llm/prompts.py`.
- Average generation time per document (already in `elapsed_seconds`).
- One full worked example showing: article (truncated) → basic summary →
  structured summary → reference. Comment on differences.
- A list of 3-5 hallucinations you found by manual inspection (this is
  cheap evidence and very persuasive).
- ROUGE / BERTScore / faithfulness numbers come in Phase 6 — leave
  placeholders.

## When Phase 4 is done

You have two more prediction files in `data/processed/`:
- `predictions_llm_mistral_basic.jsonl`
- `predictions_llm_mistral_structured.jsonl`

Combined with Phase 3, you now have **four** prediction files in the
shared format. We're halfway through the comparison — move on to Phase 5
to add the RAG layer.
