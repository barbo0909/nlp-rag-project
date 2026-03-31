# Turkish Legal RAG (Retrieval-Augmented Generation)

A production-oriented Turkish legal question-answering system built with progressive baseline → evaluation → optimization methodology.

**Current Status:** ✅ Baseline prototype complete (Notebooks 01–06)  
**Architecture:** Question → Embedding → Hybrid Retrieval (Dense + BM25) → Reranker → LLM → Grounded Answer

---

## 📊 Project Overview

This is **not** a quick demo—it's a methodologically sound research project designed to:

1. Build a working baseline RAG system
2. Quantify component contributions (retrieval, reranking, generation)
3. Identify bottlenecks and failure modes
4. Systematically improve through ablation and tuning

### Key Principle: Baseline → Measure → Optimize

Rather than immediately tuning models, we first built a complete working baseline, identified weaknesses, then systematized improvements.

---

## ✅ Completed (Notebooks 01–06)

| Notebook | Stage | Status | Output |
|----------|-------|--------|--------|
| **01** | Dataset Preparation | ✅ | `merged_legal_qa.csv` (26,961 records) |
| **02** | Retrieval Corpus | ✅ | `retrieval_corpus_full.csv` (27,745 chunks) |
| **03** | Dense Retrieval | ✅ | FAISS index + embeddings |
| **04** | Hybrid Retrieval | ✅ | Dense + BM25 fusion (α=0.5) |
| **05** | Reranker | ✅ | Cross-encoder ranking |
| **06** | Baseline RAG Generation | ✅ | Grounded answers (Gemma-2-9B-IT) |

---

## 📈 Dataset Statistics

### Source 1: Kaggle Dataset
- **Name:** `turkish-law-dataset-for-llm-finetuning`
- **Raw records:** 13,954
- **After cleaning:** 13,607 (347 duplicates removed)
- **Coverage:** Turkish Civil Code, Criminal Code, Criminal Procedure, Constitutional Law, Commercial Law

### Source 2: Hugging Face Dataset  
- **Name:** `Renicames/turkish-law-chatbot`
- **Train split:** 13,354 records (full split loaded, not sample)
- **Test split:** Not included (avoiding evaluation leakage)

### Combined Final Corpus
- **Total records:** 26,961 unique Q&A pairs
- **After chunking (800 chars, 100 overlap):** 27,745 chunks
- **Chunk statistics:**
  - Min length: 53 chars
  - Max length: 800 chars
  - Mean: 376 chars
  - Documents with metadata: 14,155 (source/category/quality)
  - Document without metadata: ~12,806 (HF-sourced, null source)

---

## 🔍 Retrieval Pipeline Architecture

```
Question
   ↓
[Dense Embedding] + [BM25 Score]  ← sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2
   ↓                ↓
Top-20 Dense    Top-20 BM25
   ↓                ↓
   └─────[Hybrid Fusion, α=0.5]─────┘
              ↓
         Top-20 Hybrid
              ↓
      [Cross-Encoder Reranker]  ← cross-encoder/ms-marco-MiniLM-L-6-v2
              ↓
          Top-3 Chunks  ← for generation
              ↓
        [LLM Generation]  ← google/gemma-2-9b-it (float16, A100)
              ↓
       Grounded Answer + Citations
```

---

## 🎯 Key Findings

### Dense Retrieval (Notebook 03)
✅ **Works:** Technical implementation correct, FAISS index functioning  
⚠️ **Issue:** Semantically close but legally irrelevant results; includes noisy/metadata-null chunks

### Hybrid Retrieval (Notebook 04)
✅ **Improvement:** Exact-term matching via BM25 resolves some semantic noise  
✅ **Complementary:** Dense and BM25 top-3 typically have 0-1 overlap  
⚠️ **Still noisy:** Candidate pool still contains bozuk (broken) QA pairs; not a full solution

### Reranker (Notebook 05)
✅ **Effective:** Reshuffles candidate ranking meaningfully (top-1 changed in 3/5 queries)  
✅ **Best improvements:** Queries 2, 4, 5 improved significantly  
⚠️ **Limitation:** English MS MARCO model—no Turkish legal domain knowledge; struggles with very noisy pools

### Baseline Generation (Notebook 06)
✅ **Functional:** Grounded answers produced with source attribution  
✅ **Valid responses:** Queries 2, 5 produced solid answers  
⚠️ **Issues:**
- Over-cautious: "Verilen bağlamda yeterli bilgi yoktur" for Query 1 despite decent retrieval
- Citation quality: Supporting sources include generic "Kaynak 1" for metadata-null records
- Formatting: Some answers split across code blocks (needs prompt refinement)

---

## ⚠️ Critical Known Issues

### 1. **source = null Records (12K+ chunks)**
- HF dataset has no source/category/quality_score metadata
- Creates "Kaynak 1" placeholder citations instead of real source names
- **Impact:** Citation accuracy, faithfulness evaluation, supporting evidence quality

### 2. **Data Quality Heterogeneity**
- Kaggle records: Well-structured, source-attributed, some duplicates
- HF records: Cleaner duplicates but missing metadata
- Result: Noisy candidate pools with wrong QA pairings in corpus (especially Query 3)

### 3. **Domain Mismatch in Models**
- Embedding: Multilingual but general-purpose (not Turkish legal)
- Reranker: English MS MARCO-specific (not Türkçe hukuk)
- Generation: Instruction-tuned Gemma (good, but not legal-specific)
- **Solution needed:** Domain adaptation / fine-tuning

### 4. **No Gold Benchmark**
- Currently evaluating on ad-hoc test queries
- No verified ground-truth Q&A pairs for formal evaluation
- **Next step:** Create gold benchmark with manual verification

---

## 📋 Models Used

| Component | Model | Parameters | Notes |
|-----------|-------|------------|-------|
| Embedding | `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` | 33M | 384-dim output, L2-normalized |
| Reranker | `cross-encoder/ms-marco-MiniLM-L-6-v2` | 22M | English domain; not Türkçe optimized |
| Generation | `google/gemma-2-9b-it` | 9B | float16 on A100; instruction-tuned |

---

## 📁 Project Structure

```
nlp-rag-project/
├── data/
│   ├── raw/                          # Original Kaggle CSV
│   ├── processed/                    # Normalized datasets
│   │   ├── kaggle_normalized.csv
│   │   ├── hf_normalized.csv
│   │   ├── merged_legal_qa.csv
│   │   └── merged_legal_qa.jsonl
│   └── retrieval/
│       └── retrieval_corpus_full.csv   # Final 27,745 chunks
├── notebooks/
│   ├── 01_dataset_preparation.ipynb           # ✅ Merge Kaggle + HF
│   ├── 02_retrieval_corpus_preparation.ipynb  # ✅ Chunking
│   ├── 03_baseline_dense_retrieval.ipynb      # ✅ FAISS
│   ├── 04_hybrid_retrieval_baseline.ipynb     # ✅ Dense + BM25
│   ├── 05_reranker_baseline.ipynb             # ✅ Cross-encoder reranking
│   ├── 06_baseline_rag_generation.ipynb       # ✅ Gemma generation
│   ├── 07_retrieval_evaluation.ipynb          # ⏳ Next: Recall@k, MRR, nDCG
│   ├── 08_qa_evaluation.ipynb                 # ⏳ Next: EM, F1, BLEU, cite accuracy
│   └── 09_hallucination_analysis.ipynb        # ⏳ Next: Analyze false info
├── outputs/
│   ├── dense_retrieval/           # FAISS index, embeddings
│   ├── hybrid_retrieval/          # Hybrid fusion rankings
│   ├── reranker/                  # Reranker scores & rankings
│   └── rag_generation/            # Generated answers + metadata
├── src/                           # (Reserved for Python modules)
├── requirements.txt
└── README.md
```

---

## 🔄 Retrieval Test Queries (5 Queries)

All evaluation uses the same 5 Turkish legal queries:

1. **Haksız zenginleşme ile ilgili hükümler nelerdir?**
   - Domain: Tort Law (Unjust Enrichment)
   - Expected: Turkish Civil Code §946-973

2. **Miras bırakanın tasarruf özgürlüğü nasıl sınırlandırılır?**
   - Domain: Inheritance Law
   - Expected: Forced heirship (saklı pay)

3. **Ceza muhakemesinde tutuklama şartları nelerdir?**
   - Domain: Criminal Procedure
   - Expected: CPC §100-103

4. **Anayasa'ya göre devletin şekli nedir?**
   - Domain: Constitutional Law
   - Expected: Turkish Constitution, Republic structure

5. **Aile yurdu ile ilgili malik üzerindeki sınırlamalar nelerdir?**
   - Domain: Family Law / Property Rights
   - Expected: Turkish Civil Code family home restrictions

---

## 🚀 Next Steps (Notebooks 07–09, Not Yet Started)

### Phase 1: Evaluation (CRITICAL—currently missing)

**Notebook 07: Retrieval Evaluation**
- Compute Recall@5, Recall@10
- Calculate MRR (Mean Reciprocal Rank)
- Calculate nDCG (Normalized Discounted Cumulative Gain)
- *Requires:* Gold benchmark with annotated relevant chunks

**Notebook 08: QA Evaluation**
- Exact Match (EM) and F1 against gold answers
- BLEU and ROUGE scores
- Citation Accuracy (does supporting source match text?)
- Faithfulness (is answer grounded in retrieval?)

**Notebook 09: Hallucination Analysis**
- Identify contradictions between generated answer and retrieved context
- Flag unsupported claims
- Measure citation quality

### Phase 2: Optimization (Following 07–09)

1. **Embedding Tuning**
   - Hard negative mining on legal corpus
   - Domain-adapted fine-tuning on Q&A pairs

2. **Reranker Improvement**
   - Fine-tune on Turkish legal Q&A pairs
   - Or switch to stronger multilingual model

3. **LLM Optimization**
   - Instruction tuning with legal examples
   - Improved grounded prompting
   - Handling of source-null records

4. **Data Cleaning**
   - Identify and remove bozuk QA pairs
   - Move source-null HF records to separate track
   - Consider source validation / augmentation

5. **Fully Optimized System**
   - Compare tuned vs. baseline (ablation)
   - Measure improvement per component
   - Generate final benchmark results

---

## 🏗️ Methodology: Why This Approach?

We deliberately did **not** start with fine-tuning. Instead:

1. ✅ **Built a working baseline first** (notebooks 01–06)
2. ⏳ **Measure the baseline** (notebooks 07–09)
3. ⏳ **Identify bottlenecks** (from evaluation)
4. ⏳ **Tune systematically** (embedding → reranker → LLM)
5. ⏳ **Quantify improvements** (ablation studies)

**Why?** Because without measuring the baseline, you can't tell which tuning actually helped.

---

## 💻 Running the Notebooks

### Prerequisites
```bash
pip install -r requirements.txt
```

### In Google Colab
Each notebook includes:
- Google Drive mount
- Configurable paths
- Dependency installation

```python
# Drive mount (automatic in Colab)
from google.colab import drive
drive.mount('/content/drive')
```

### Data Loading
All notebooks load from:
```
/content/drive/My Drive/nlp-rag-project/
```

Ensure this path exists in your Google Drive.

---

## 📊 Key Metrics (Current Baseline)

| Metric | Value | Notes |
|--------|-------|-------|
| Dense Recall@5 | Low | Noisy results; semantic drift |
| Hybrid Recall@5 | Higher | Improvement from BM25 |
| Reranker improvement | +3/5 queries | Top-1 reshuffled; quality varies |
| Citation accuracy | Low | Many "Kaynak N" placeholders |
| Generation quality | Mixed | 2 strong, 1 moderate, 1 weak, 1 strong |

---

## 🎓 Design Principles

1. **Grounded Generation:** Model answers only from retrieved context
2. **Citation Tracking:** Every answer includes source attribution (to extent possible)
3. **Reproducibility:** All steps logged, outputs saved, notebooks self-contained
4. **Progressive Evaluation:** Baseline → measurement → tuning
5. **Methodological Rigor:** Avoid evaluation leakage (test split not used for training)

---

## 📝 License & Attribution

- **Kaggle Dataset:** `batuhankalem/turkish-law-dataset-for-llm-finetuning`
- **HF Dataset:** `Renicames/turkish-law-chatbot`
- **Models:** Sentence-Transformers, Cross-Encoder, Google Gemma-2

---
---

## 🔗 Continue Reading

- **Detailed architecture:** See [notebooks/README_DETAILED.md](notebooks/) (future)
- **Evaluation plan:** See [notebooks/07_retrieval_evaluation.ipynb](notebooks/) (in progress)
- **Results & findings:** Check `outputs/*/` after running notebooks
