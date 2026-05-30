# Medical Document Summarization — Extractive vs Abstractive vs LLM+RAG

End-of-year engineering project — ENSA Khouribga, 2025–2026.

This project benchmarks three approaches for summarizing English medical
documents (clinical notes, biomedical abstracts):

1. **Extractive baseline** — TextRank. No hallucination possible, but limited fluency.
2. **Abstractive** — fine-tuned BART / BioBART on (document, summary) pairs.
3. **LLM + RAG** — Mistral / LLaMA served locally via Ollama, augmented with a
   medical knowledge base (normal lab ranges, drug classes, abbreviations).

The core research question is not "which one gets the best ROUGE", but
**which one is most factually faithful to the source document**. We design a
custom faithfulness metric on top of medical NER to detect hallucinated
entities (drugs, dosages, diagnoses) that appear in the summary but not in
the source.

## Status
Phase 1 — environment setup.

## Repository layout
```
medical-summarizer/
├── src/
│   ├── preprocessing/   # cleaning, chunking, tokenization
│   ├── extractive/      # TextRank baseline
│   ├── llm/             # Ollama client, prompt templates
│   ├── rag/             # knowledge base, vector store, retrieval
│   └── evaluation/      # ROUGE, BERTScore, faithfulness via NER
├── data/
│   ├── raw/             # downloaded datasets (gitignored)
│   ├── processed/       # cleaned (document, summary) pairs
│   └── knowledge_base/  # medical reference for RAG
├── notebooks/           # exploration, error analysis
├── scripts/             # one-off CLI scripts (download, run pipeline)
└── tests/
```

## Quick start
See `docs/setup.md`.
