# Persian Legal RAG Prototype — Case Study Report

**Author submission** · Lawera AI Engineer take-home  
**Corpus:** Iranian Civil Code (قانون مدنی), consolidated text from qavanin.ir  
**Scope:** Single-statute RAG with query analysis, hybrid retrieval, evidence gating, and grounded generation

---

## 1. Goal

Build a small, reproducible RAG system that:

- Loads and structures statute text with legal metadata
- Normalizes Persian text robustly
- Retrieves relevant articles for user questions
- Uses metadata (article identity, amendment status) in ranking and context
- Generates concise answers **only** from selected evidence, or abstains

This is a prototype, not production software. Design choices prioritize clarity, safety, and Persian legal characteristics over scale.

---

## 2. Architecture

```
Civil Code TXT
    → metadata + body split
    → provision extractor (ماده / تبصره / hierarchy / amendment status)
    → Persian normalization
    → article-level chunks (notes merged into parent)
         │
User query → QueryAnalyzer (deterministic)
         │         article #, law title, intent
         ▼
    BM25  +  dense (Jina v3)  →  RRF fusion
         │         + direct article lookup when ماده N is explicit
         ▼
    Cross-encoder rerank (Jina) + amendment down-weight
         ▼
    EvidenceSelector (sufficiency / abstain / top-k)
         ▼
    Local LLM (Ollama) with grounded system prompt
```

| Stage | Choice | Why |
|--------|--------|-----|
| Chunking | One chunk = one article; تبصره merged into parent | Legal answers cite articles; notes belong with the parent |
| Normalization | Explicit digit/letter/diacritic/ZWNJ maps | Arabic/Persian variants break exact match if left raw |
| Dense retrieval | Jina embeddings-v3 (`retrieval.passage` / `retrieval.query`) | Strong multilingual support; task adapters |
| Lexical retrieval | BM25 on normalized tokens | Exact legal terms and article numbers |
| Fusion | Reciprocal Rank Fusion | Robust; no score calibration between systems |
| Reranker | Jina reranker-v2-multilingual | Cross-encoder quality on Persian |
| Query analysis | Regex + known-law list (not an LLM) | Explicit `ماده` / `قانون` are hard constraints |
| Evidence gate | Score floor + exact article match rules | Predictable abstention; testable without an LLM |
| Generation | Local Gemma via Ollama | Privacy for legal text; no API dependency |

**Alternatives considered**

- Fixed-size or sentence chunks → breaks article identity  
- Dense-only or BM25-only → weaker on paraphrases or exact terms  
- LLM-based query parsing → non-deterministic on identifiers that must be exact  
- Always pass top-k to the LLM → encourages hallucination on off-corpus questions  

---

## 3. Implementation highlights

### Ingestion and structure

- Parses the `# ===== METADATA =====` block and statute body  
- Extracts articles, notes (تبصره), hierarchy headers (جلد / باب / فصل / …), and amendment markers (اصلاحی / منسوخه + dates)  
- Validates non-empty input and encoding  

### Persian normalization

- Persian/Arabic digits → ASCII  
- `ك/ي` → `ک/ی`  
- Strip diacritics and tatweel; clean ZWNJ noise  
- Applied to both corpus and queries so BM25 and dense search align  

### Query analysis

Deterministic extraction of:

- Explicit article references (`ماده 30`, `ماده ۲۱۸ مکرر`, …)  
- Known law titles (e.g. قانون مدنی)  
- Intent: `article_lookup` | `legal_question` | `multi_issue` | `unknown`  

Explicit identifiers never depend on generative guessing.

### Retrieval and constraints

- Hybrid BM25 + dense with RRF  
- **Direct metadata lookup** when the user names `ماده N`, so short queries are not lost after keyword stripping  
- Soft boost for matching article/law; abrogated articles down-weighted in the reranker  

### Evidence selection

- Explicit article queries require a matching article number (with chunk-scan fallback)  
- General questions need top reranker score ≥ `abs_floor` (0.25; calibrated on observed on-topic vs off-corpus scores)  
- Machine-readable `reason` codes for logging and evaluation  

### Generation

- System prompt: answer only from provided materials; flag منسوخه; list cited articles; abstain if empty  
- Low temperature; optional multi-model Ollama fallback  

---

## 4. Demonstration queries

| ID | Query type | Observed behaviour |
|----|------------|-------------------|
| paraphrase | Rephrased “شرایط شاهد” | Retrieves ماده ۱۳۱۳; answer lists maturity, reason, justice, faith, legitimate birth + notes |
| same_number_identity | `ماده ۱۰ قانون مدنی` | Article lookup path; returns private-contracts rule when lookup/inject is active |
| multi_issue | Immovable property **and** right of usufruct | Intent `multi_issue`; retrieves ماده ۱۲ and ماده ۴۰ (and related) |
| negated | Is testimony of a professional beggar accepted? | Retrieves ۱۳۱۳; answer correctly **no** (تبصره ۲) |
| insufficient | Armed robbery penalty under Civil Code | Abstains (`low_retrieval_confidence`); Civil Code is not criminal law |
| abrogated_awareness | شرایط شاهد per ماده ۱۳۱۳ مکرر | Retrieves منسوخه text; answer states the article is abrogated |

---

## 5. Evaluation and error analysis

Lightweight checks used:

- **Article hit:** normalized overlap between retrieved and expected article numbers  
- **Abstention:** evidence `sufficient=False` and/or answer text containing abstention markers  

**What worked**

- Hybrid + RRF recovers paraphrases better than either channel alone  
- Merging تبصره into the parent article keeps witness-condition answers complete  
- Evidence floor blocks weak off-corpus hits (~0.13 scores) while keeping on-topic hits (~0.5–0.8)  
- Explicit `مکرر` / منسوخه handling surfaces historical text without presenting it as current law  

**What was weak / failed initially**

- Short “ماده N …” queries lost the article after stopword-like stripping until **direct article lookup** was added  
- Without `abs_floor`, low-confidence criminal questions still passed the evidence gate (LLM sometimes abstained anyway; the gate must not rely on that)  
- Multi-law “same article number, different statute” cannot be fully stress-tested on a single-law corpus  

**Assumptions**

- Corpus is one consolidated Civil Code file (the brief’s synthetic multi-source JSONL was not used; the provided statute TXT was)  
- Reranker score scale is model-specific; `abs_floor=0.25` should be re-calibrated if the reranker changes  

---

## 6. Dependencies and how to run

**Main libraries:** `sentence-transformers`, `transformers`, `rank_bm25`, `chromadb`, `ollama`, `torch`, `pydantic`

**Run**

1. Place the Civil Code TXT on a path listed in the ingestion cell (or edit the path)  
2. `pip install` the packages above  
3. Optional: `ollama pull gemma4` (or another local model)  
4. Open the notebook and run top to bottom (GPU recommended for embeddings and reranker)  

Generation falls back to printing retrieved evidence if no local model is available, so retrieval quality remains inspectable.

**AI tools used during development:** conversational coding assistant for architecture review and notebook assembly. Models: Jina embeddings-v3, Jina reranker-v2-multilingual, local Gemma via Ollama. No employer-owned code or secrets.

---

## 7. What I would change before production

1. **Multi-source corpus** with `authority_tier` (statute / binding precedent / official guidance / commentary) and ranking that prefers controlling authority  
2. **`(law_id, article_number)`** as the primary key so the same article number in another act cannot be substituted  
3. **Temporal graph** of amendments; default “current only” mode; optional historical mode  
4. **Access control** on restricted client documents—exclude before ranking/context  
5. **Deduplication** across editions  
6. **Fixed Persian legal eval set** with Recall@k, citation accuracy, abstention precision/recall, latency  
7. **Infrastructure:** managed or incremental vector index; distilled reranker; citation verification before display  
8. Extend QueryAnalyzer (`known_laws`, negation, historical vs current); keep identifiers deterministic  

**Privacy / cost / latency (prototype)**

| Aspect | This prototype | Production note |
|--------|----------------|-----------------|
| Privacy | Local embeddings + local LLM | Prefer local or contracted hosting for client legal data |
| Cost | One-time GPU indexing; local inference | Cloud APIs scale but raise residency and cost issues |
| Latency | Hybrid + cross-encoder + LLM on GPU | Cache frequent queries; async pre-retrieve |

---

## 8. Deliverables checklist

| Item | Location |
|------|----------|
| Runnable prototype | `Legal_RAG_Case_Study_FIXED.ipynb` (or final notebook name) |
| Architecture & choices | This README §§2–3 |
| Example queries & outputs | Notebook §9 demos; summary in §4 |
| Evaluation / error analysis | Notebook §10; summary in §5 |
| Production notes | §7 |
| Setup instructions | §6 and notebook final markdown |

---

*This work is a focused engineering prototype for evaluation of technical choices, not a substitute for professional legal advice.*
