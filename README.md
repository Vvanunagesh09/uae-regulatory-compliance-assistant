# UAE Regulatory & Compliance Assistant

A Retrieval-Augmented Generation (RAG) assistant that answers questions about DIFC / UAE regulations **only from official PDF documents**, with citations, a confidence score, and the ability to say "I don't know".

Built in Google Colab. A free CPU runtime is enough (a T4 GPU is optional).

## Features

- **Grounded answers**: every sentence carries a `[n]` citation pointing to the source article and page
- **Hybrid retrieval**: dense embeddings + BM25, fused with Reciprocal Rank Fusion (RRF)
- **Cross-encoder reranking** for more precise evidence selection
- **Structure-aware chunking** that splits by Article / Section instead of fixed word counts
- **Abstention**: refuses out-of-scope questions when the best relevance score is below a threshold
- **Guardrail** against prompt-injection attempts
- **Verification** without an extra LLM call: citation check + lemma overlap with the cited evidence
- **Confidence score** = 0.4 x reranker probability + 0.2 x dense/BM25 agreement + 0.4 x verified-sentence ratio
- Only **1 LLM call per question**, with automatic fallback to another Groq model on rate limits

## Documents used

| Document | Pages |
|---|---|
| DIFC Companies Law | 102 |
| DIFC Contract Law (Law No. 6 of 2004, updated 2024) | 41 |
| DIFC Data Protection Law | 54 |

## Pipeline

```
PDFs -> Extraction -> Cleaning & normalisation -> Linguistic processing (spaCy)
     -> Chunking (structure-aware + fixed baseline) -> Metadata
     -> Indexing (dense + BM25) -> Hybrid retrieval (RRF) -> Cross-encoder reranking
     -> Evaluation -> Visualisation -> RAG answering (citations, verification, abstention)
```

1. **Extraction**: PyMuPDF, page by page (page numbers kept for citations)
2. **Cleaning**: Unicode (NFKC) normalisation, header/footer removal, de-hyphenation, paragraph reflow
3. **Linguistic processing**: spaCy sentences, tokens, lemmas (legal words like *shall, may, must, not* are kept)
4. **Chunking**: Article / Section detection with contextual headers, plus a fixed-size baseline for comparison
5. **Indexing**: `BAAI/bge-small-en-v1.5` embeddings + BM25 over lemmatised tokens
6. **Retrieval**: top-20 hybrid candidates, then cross-encoder reranking to top-4 evidence
7. **Generation**: Groq LLM answers only from the numbered evidence

## Tech stack

Python, PyMuPDF, spaCy, sentence-transformers, rank-bm25, scikit-learn, pandas, NumPy, Matplotlib, Groq API (Llama / GPT-OSS models), Google Colab

**Models:** `BAAI/bge-small-en-v1.5` (embeddings), `cross-encoder/ms-marco-MiniLM-L-6-v2` (reranker)

## Results

Retrieval evaluation on 8 questions (structured chunking vs fixed-size chunking):

| Chunking | Retriever | Hit@1 | MRR | nDCG@10 |
|---|---|---|---|---|
| Structured | BM25 | 1.000 | 1.000 | 0.886 |
| Structured | Dense | 1.000 | 1.000 | 0.952 |
| Structured | Hybrid | 1.000 | 1.000 | 0.937 |
| Structured | Hybrid + Rerank | 1.000 | 1.000 | 0.951 |
| Fixed | BM25 | 0.875 | 0.896 | 0.737 |
| Fixed | Dense | 0.875 | 0.875 | 0.685 |
| Fixed | Hybrid | 0.875 | 0.888 | 0.748 |
| Fixed | Hybrid + Rerank | 0.875 | 0.938 | 0.716 |

End-to-end test:
- Gold article found in the evidence for **8 / 8** in-scope questions
- **3 / 3** out-of-scope trap questions correctly refused
- Prompt-injection attempt correctly blocked

> **Limitation:** the evaluation uses only 8 LLM-generated questions, so the numbers are indicative, not statistically strong.

## How to run

1. Open `UAE_Regulatory_Compliance_Assistant.ipynb` in Google Colab
2. Get a free API key from [console.groq.com](https://console.groq.com)
3. In Colab, open the **Secrets** (key icon) panel, add `GROQ_API_KEY`, and enable notebook access
4. Upload the 3 PDFs when the ingestion cell asks (or place them in a Google Drive folder and set `data_dir`)
5. Run all cells

Never paste your API key directly into the notebook.

## Example

**Q:** What information must be included in the directors' report for a public company each financial year?

**A:** Returns the required items (directors' names, principal activities, recommended dividend, business review, audit-information statement), with citations to *Companies Law, Section 126, p.78* and a confidence score of 0.94.

## Disclaimer

This tool gives an automated summary of retrieved regulatory text. It is **not legal advice**. Always check the cited articles.

## Author

**V Anu**: AI/ML instructor and B.Tech (CS & Data Science) student
