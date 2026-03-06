# Experiment Log - SHL Assessment Recommendation Engine

## Experiment 1: Baseline FAISS-only Retrieval
**Date:** March 1, 2025
**Approach:** Pure semantic search using FAISS with OpenAI text-embedding-3-large
**Config:**
- Embedding model: text-embedding-3-large
- Top-K per query: 50
- Single query (raw user input)
- No reranking

**Results:**
| Metric | Value |
|--------|-------|
| Recall@10 (train) | 0.38 |

**Observations:**
- Semantic search alone misses keyword-specific assessments (e.g., "Selenium", "HTMLCSS")
- Works well for descriptive queries but fails on exact product name matches
- Single query doesn't cover all skill dimensions in complex JDs

---

## Experiment 2: Multi-query FAISS Retrieval
**Date:** March 2, 2025
**Approach:** LLM generates 10 search queries per user query, run all through FAISS
**Config:**
- Query analyzer: GPT-4.1 generates 10 search queries
- FAISS top-K per query: 50
- Max-score fusion across queries
- No reranking

**Results:**
| Metric | Value |
|--------|-------|
| Recall@10 (train) | 0.45 |

**Observations:**
- Multi-query improves coverage significantly (+7%)
- Still misses exact keyword matches (BM25 needed)
- Query analyzer sometimes generates too generic queries

---

## Experiment 3: Hybrid FAISS + BM25
**Date:** March 3, 2025
**Approach:** Added BM25 keyword search alongside FAISS, with score fusion
**Config:**
- FAISS semantic search + BM25Okapi keyword search
- Linear score fusion: 0.5 * FAISS + 0.5 * BM25
- Basic tokenization (lowercase + split on non-alphanumeric)
- No reranking

**Results:**
| Metric | Value |
|--------|-------|
| Recall@10 (train) | 0.52 |

**Observations:**
- BM25 captures exact keyword matches that FAISS misses
- Score fusion weights need tuning (equal weights not optimal)
- BM25 benefits from name boosting in corpus

---

## Experiment 4: Hybrid + LLM Reranker
**Date:** March 3, 2025
**Approach:** Added GPT-4.1 reranker after hybrid retrieval
**Config:**
- Hybrid retrieval (FAISS + BM25)
- LLM reranker selects top 10 from 50 candidates
- Basic reranker prompt (just select most relevant)
- Score fusion: 0.6 * FAISS + 0.4 * BM25

**Results:**
| Metric | Value |
|--------|-------|
| Recall@10 (train) | 0.55 |

**Observations:**
- LLM reranker adds +3% by understanding query intent better
- But reranker sometimes over-indexes on personality tests
- Need domain-specific guidance in reranker prompt

---

## Experiment 5: Enhanced Query Analyzer with SHL Catalog Knowledge
**Date:** March 4, 2025
**Approach:** Injected SHL product catalog naming patterns into query analyzer prompt
**Config:**
- Query analyzer generates 15-20 queries (up from 10)
- Catalog-aware: knows "Automata", "Verify", "OPQ", "JFA" naming conventions
- Mix of keyword queries (for BM25) and descriptive queries (for FAISS)
- Top-K per query: 20, TOP_K_TO_LLM: 70

**Results:**
| Metric | Value |
|--------|-------|
| Recall@10 (train) | 0.58 |

**Observations:**
- Catalog knowledge helps generate precise keyword queries
- "Automata Fix", "Automata SQL" now retrieved correctly
- Coverage improved for role-specific solutions (JFA, Short Form)

---

## Experiment 6: Name-boosted BM25 + Compound Tokenizer
**Date:** March 4, 2025
**Approach:** Boost assessment names in BM25 corpus, split compound words
**Config:**
- BM25 corpus: name repeated 5x + full text
- Compound tokenizer: "htmlcss" -> ["htmlcss", "html", "css"]
- Guaranteed top-2 slots per query (FAISS + BM25) to prevent misses

**Results:**
| Metric | Value |
|--------|-------|
| Recall@10 (train) | 0.62 |

**Observations:**
- Name boosting critical for assessments like "HTMLCSS", "CSS3"
- Compound tokenizer catches assessments with concatenated names
- Guaranteed slots ensure diverse retrieval per query

---

## Experiment 7: Role-aware Reranker with Examples
**Date:** March 5, 2025
**Approach:** Added role-specific selection examples and rules to reranker prompt
**Config:**
- Reranker prompt includes 12 example role -> assessment battery mappings
- Rules: limit personality tests, exhaust role packages, prefer specific over generic
- Category coverage enforcement (skill + cognitive + personality + solutions)
- Hit-count scoring: reward assessments found by multiple queries

**Results:**
| Metric | Value |
|--------|-------|
| Recall@10 (train) | 0.66 |

**Observations:**
- Example batteries teach the LLM the selection pattern
- Role-package exhaustion helps (e.g., all "Entry Level Sales" variants selected)
- Hit-count bonus rewards consensus across multiple search queries

---

## Experiment 8: BM25-only Boost + Final Tuning
**Date:** March 5, 2025
**Approach:** Special scoring for BM25-only matches, final weight tuning
**Config:**
- BM25-only boost: if BM25 > 0.3 and FAISS < 0.1, use BM25-weighted scoring
- Fusion weights: relevance 0.7 + breadth 0.3
- Within relevance: FAISS 0.6 + BM25 0.4
- Hit threshold: bonus kicks in at 3+ query hits

**Results:**
| Metric | Value |
|--------|-------|
| **Recall@10 (train)** | **0.68** |

**Per-query breakdown:**
| Query | Recall@10 |
|-------|-----------|
| Java developer + collaboration | 1.00 |
| Entry-level sales graduates | 0.56 |
| COO cultural fit (China) | 0.83 |
| Radio station sound manager | 0.60 |
| Content writer (English + SEO) | 0.60 |
| QA Engineer (Selenium, JS, SQL) | 0.78 |
| ICICI Bank Admin Assistant | 0.67 |
| Marketing Manager | 0.40 |
| Consultant (cognitive screening) | 0.40 |
| Senior Data Analyst (SQL, Python) | 0.60 |

**Observations:**
- BM25-only boost helps niche assessments that have no semantic similarity
- Marketing Manager and Consultant queries remain challenging (specialized bundles)
- Overall architecture: QueryAnalyzer(GPT-4.1) -> Hybrid(FAISS+BM25) -> Reranker(GPT-4.1)

---

## Approaches Considered but Not Used

### Sentence Transformers (Local Embeddings)
- Tried all-MiniLM-L6-v2 for embeddings
- Recall@10 was ~15% lower than OpenAI text-embedding-3-large
- Decided OpenAI embeddings justified the API cost for better quality

### Gemini as LLM Provider
- Tested Gemini 2.0 Flash as alternative to GPT-4.1
- Similar quality but slower response times
- Simplified to OpenAI-only to reduce complexity

### Cross-encoder Reranking
- Considered using a cross-encoder model for reranking
- LLM-based reranking with domain-specific prompts outperformed generic cross-encoders
- Cross-encoder couldn't leverage SHL catalog knowledge

### RAG with Full Document Retrieval
- Considered chunking assessment descriptions for RAG
- Not needed since assessments are short metadata records
- FAISS on full assessment text + BM25 was sufficient

---

## Final Architecture
```
User Query
    |
    v
[QueryAnalyzer Agent] -- GPT-4.1, temp=0.6
    | generates 15-20 search queries + extracts skills/duration/domain
    v
[Retriever Agent] -- Hybrid FAISS + BM25
    | FAISS semantic search (text-embedding-3-large)
    | BM25 keyword search (name-boosted, compound tokenizer)
    | Max+Sum score fusion with BM25-only boost
    | Returns top 70 candidates
    v
[Reranker Agent] -- GPT-4.1, temp=0.5
    | Role-aware selection with 12 example batteries
    | Balances skill tests + cognitive + personality + solutions
    | Returns top 10 recommendations
    v
Final Recommendations (10 assessments)
```

## Key Hyperparameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Embedding model | text-embedding-3-large | Best quality embeddings |
| LLM model | GPT-4.1 | For query analysis + reranking |
| TOP_K_PER_QUERY | 20 | FAISS + BM25 per search query |
| TOP_K_TO_LLM | 70 | Candidates sent to reranker |
| TOP_K_FINAL | 10 | Final recommendations |
| Search queries per input | 15-20 | Generated by query analyzer |
| BM25 name boost | 5x | Name repeated 5 times in corpus |
| Score fusion (relevance) | FAISS 0.6 + BM25 0.4 | Semantic weighted slightly higher |
| Score fusion (final) | Relevance 0.7 + Breadth 0.3 | Max-score matters more than sum |
| Hit count threshold | 3+ queries | Bonus for multi-query consensus |
