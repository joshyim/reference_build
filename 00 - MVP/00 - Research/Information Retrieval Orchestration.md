# Information Retrieval: RAG Retrieval Pipeline Components

Research summary covering the retrieval pipeline for a RAG application with prepared keywords, vector embeddings, and a chat interface.

---

## 1. Query Processing

**What it does:** Transforms raw user chat input into one or more retrieval-optimized queries before anything hits the database.

**Why it matters:** Users type conversational, ambiguous, or multi-part questions. The gap between natural language input and what retrieval indexes respond to well is the single largest source of retrieval failure. Teams using query rewriting report 30-45% improvements in retrieval precision without changing any search infrastructure [1].

### Techniques

#### Query Rewriting
- An LLM rephrases the user's question using domain-specific vocabulary while preserving intent
- Example: "How do I fix that login thing?" becomes "Troubleshoot authentication failure on user login"
- Start with clear LLM instructions for rewriting, then A/B test whether rewritten queries outperform raw input

#### Query Expansion / Augmentation
- Adds terms or context when the original query is too short or ambiguous
- Generate 3-5 semantically similar query variants, retrieve for each, merge results -- this dramatically increases the probability of capturing relevant documents
- Particularly useful for embedding-based retrieval where a single query vector may miss relevant regions of the embedding space

#### Query Decomposition
- Breaks complex, multi-part questions into independent sub-queries, each targeting a specific retrieval axis
- Example: "What are the regulatory compliance requirements for remote work in our European offices?" decomposes into: (1) regulatory compliance frameworks, (2) remote work policies, (3) geographic requirements for Europe
- Sub-query results are retrieved independently and then aggregated

#### Query Routing / Intent Classification
- Classifies the query to determine which retrieval strategy or index to use
- Simple factual lookups may go to keyword search; complex multi-hop reasoning may trigger agentic retrieval with iterative search
- RAGRouter [2] and REIC [3] are recent frameworks addressing this
- Agentic RAG approaches, where an agent controls the retrieval loop with routing, grading, and self-correction, push accuracy to 78% on complex multi-hop queries vs. traditional RAG's lower performance

| Technique | Description | Priority |
|---|---|---|
| Query rewriting | Essential for any chat-based RAG system | P0 |
| Query expansion (multi-query) | High-value, implement early | P1 |
| Query decomposition | Essential for systems handling complex, multi-part questions | P0 |
| Query routing | Nice-to-have initially, becomes essential at scale with multiple data sources or retrieval strategies | P2 |

---

## 2. Retrieval Strategies

### Dense Retrieval (Vector / Semantic Search)

**What it does:** Encodes the query into a dense vector using an embedding model, then finds the nearest document vectors by cosine similarity or inner product.

**Why it matters:** Handles synonyms, paraphrasing, and conceptual similarity naturally. A query about "employee termination" will match documents about "firing staff" even without shared keywords.

**Common implementations:**
- Embedding models: OpenAI embeddings, e5-large, BGE, Sentence Transformers
- Vector indexes: HNSW (approximate nearest neighbor), FAISS, or managed services (Pinecone, Weaviate, Qdrant, pgvector)
- Typical retrieval: top-20 to top-50 candidates for downstream re-ranking

**Weakness:** Poor at exact term matching -- product SKUs, contract clause numbers, error codes, proper names

### Sparse Retrieval (Keyword / BM25)

**What it does:** Scores documents based on term frequency, inverse document frequency, and document length normalization. Your prepared keywords map directly to this.

**Why it matters:** Excels at precise term matching. When a user searches for a specific identifier, code, or proper noun, BM25 reliably surfaces the right documents.

**Common implementations:**
- Classic BM25 via Elasticsearch, OpenSearch, or Lucene
- Your pre-prepared keywords can serve as the term index, boosting signal beyond raw document text

### Learned Sparse Retrieval (SPLADE)

**What it does:** Uses a neural model to produce sparse token-weight vectors, combining the interpretability and exact-match strength of sparse retrieval with learned semantic expansion.

**Why it matters:** SPLADE and similar models expand the document and query representations with related terms the original text may not contain, bridging the vocabulary mismatch problem that limits pure BM25, while remaining compatible with inverted index infrastructure [4].

**Implementation:** Can replace or augment BM25 in the sparse leg of hybrid search. Particularly strong when combined with dense retrieval.

### ColBERT / Late Interaction

**What it does:** Stores per-token embeddings for documents and computes fine-grained token-level similarity at query time (MaxSim operation). Offers a middle ground between bi-encoder speed and cross-encoder accuracy.

**Why it matters:** Better at capturing nuanced relevance than bi-encoders, while remaining much faster than cross-encoders at retrieval time. ColBERTv2 with PLAID engine is production-viable [5].

### Hybrid Retrieval (Dense + Sparse)

**What it does:** Runs sparse and dense retrieval in parallel, then fuses the two ranked result lists into a single list.

**Why it matters:** Real-world improvement of 15-30% better recall than either method alone [6]. Every major production vector database now supports hybrid search (as of March 2026).

#### Fusion Methods

**Reciprocal Rank Fusion (RRF)**
- Scores each document by summing `1/(k + rank)` across all result lists it appears in
- The constant k=60 is the industry standard (from Cormack et al., SIGIR 2009)
- Best starting point: simple, resilient to mismatched score scales, strong results without tuning
- Ideal for prototyping or when retriever score distributions differ significantly

**Linear Combination / Weighted Sum**
- Normalizes scores from each retriever to a common scale, then combines with tunable weights (e.g., 0.7 * dense + 0.3 * sparse)
- Requires calibration to set weights appropriately for your data
- More tunable than RRF but more fragile if score distributions shift

**Learned Fusion**
- Train a small model to predict optimal combination of retriever scores
- Most complex, best for mature systems with evaluation data

| Technique | Description | Priority |
|---|---|---|
| Dense retrieval | Essential — you already have embeddings | P0 |
| Sparse / BM25 | Essential — you already have keywords | P0 |
| Hybrid with RRF | Essential — the simplest high-impact win | P0 |
| SPLADE / ColBERT | Evaluate when optimizing beyond baseline | P2 |
| Learned fusion | For mature systems with training signal | P2 |

---

## 3. Re-ranking

**What it does:** Takes the initial retrieval results (typically top-20 to top-50) and re-scores them with a more computationally expensive but more accurate model, producing a final top-5 to top-10 for the LLM.

**Why it matters:** Databricks testing confirms reranked results reduce LLM hallucinations by 35% compared to raw embedding similarity [7]. The pattern is: retrieve broadly (top-20), rerank precisely (top-5), send only the best to the LLM.

### Cross-Encoder Rerankers

**How it works:** Takes the query and each candidate passage as a single concatenated input, produces a relevance score. Because the model attends across the combined sequence, it captures fine-grained alignment, negation, and subtle constraints.

**Models:** cross-encoder/ms-marco-MiniLM, BGE-reranker, Cohere Rerank v3.5, Jina Reranker

**Trade-off:** High accuracy, but O(n) inference cost per candidate. Practical for re-ranking 20-100 candidates.

### LLM-Based Rerankers

**How it works:** Prompts an LLM to score or rank candidate passages given the query. Can handle nuanced relevance judgments.

**Trade-off:** More expensive per request, higher latency. Best for high-value flows where precision is critical. Can leverage the same LLM you use for generation.

**Models/approaches:** RankGPT (listwise ranking), RankRAG (unified ranking + generation) [8], pointwise scoring with an LLM

### Lightweight / Distilled Rerankers

**How it works:** Smaller, faster models (e.g., FlashRank, zerank-1) distilled from larger cross-encoders or LLMs.

**Trade-off:** Near-cross-encoder quality at much lower latency. zerank-1 delivers +28% NDCG@10 over baseline retrievers [9].

| Technique | Description | Priority |
|---|---|---|
| Cross-encoder re-ranking | Essential for production quality on top-20 results | P0 |
| LLM-based re-ranking | Use for high-stakes queries where precision is critical | P2 |
| Lightweight rerankers | Strong alternative when latency budget is tight | P1 |

---

## 4. Context Assembly

**What it does:** Assembles the final set of retrieved (and re-ranked) chunks into a coherent context block for the LLM prompt.

**Why it matters:** Even with perfect retrieval, poor context assembly -- redundancy, bad ordering, exceeding the context window, losing surrounding context -- degrades generation quality. In production RAG systems, 30-40% of retrieved context is semantically redundant [10].

### Chunk Selection and Deduplication

- After re-ranking, deduplicate chunks that are semantically overlapping (exact match, high cosine similarity between chunk embeddings, or originating from the same source passage)
- Deduplicate before assembling context, not after -- avoids wasting context window on redundant information
- Diversity-aware selection: ensure retrieved chunks cover different aspects of the query rather than all confirming the same point

### Context Window Management

- Start with 512-token chunks with 10-20% overlap as a baseline [10]
- Budget your context window: reserve tokens for system prompt, chat history, and generation; fill remaining space with retrieved context
- Prioritize higher-ranked chunks; truncate or drop lowest-ranked ones if the window fills
- Monitor actual token usage -- overstuffing the context window degrades generation quality even on models with large windows

### Ordering Strategies

- **Relevance-first ordering:** Place the most relevant chunks at the top. LLMs tend to attend more strongly to information at the beginning and end of the context ("lost in the middle" phenomenon)
- **Chronological / logical ordering:** When chunks come from a sequential document, preserve their original order to maintain coherence
- **Sandwich ordering:** Place highest-relevance chunks at the start and end, lower-relevance in the middle, to counteract the lost-in-the-middle effect

### Parent-Child Chunk Relationships

**How it works:** Index small "child" chunks (100-500 tokens) for precise retrieval matching, but when a child is retrieved, fetch its "parent" chunk (500-2000 tokens) to provide richer context to the LLM [11].

**Why it matters:** Combines the precision of small-chunk retrieval with the coherence of large-chunk context. Particularly effective for technical documents, legal texts, and research papers where sections reference ideas defined elsewhere.

**Hierarchical chunking** builds a tree of chunks at multiple granularities (document > section > paragraph > sentence). Queries can match at any level; the system can expand upward for more context or drill down for specificity.

### Late Chunking

**How it works:** Embeds the full document first so every token captures complete document context, then pools tokens within chunk boundaries. Each chunk's embedding retains awareness of pronouns, references, and document-level context.

**Impact:** 10-12% improvement in retrieval accuracy on documents with anaphoric references [12].

| Technique | Description | Priority |
|---|---|---|
| Deduplication & context window budgeting | Essential for avoiding redundancy and managing token limits | P0 |
| Relevance-based ordering | Essential — counteracts lost-in-the-middle effect | P0 |
| Parent-child retrieval | Implement when chunk context is insufficient | P1 |
| Hierarchical chunking | For complex document structures | P2 |
| Late chunking | Evaluate for documents heavy in cross-references | P2 |

---

## 5. Conversational Retrieval

**What it does:** Handles the unique challenges of multi-turn chat: resolving references to previous turns, incorporating conversation history, and maintaining topic coherence across a session.

**Why it matters:** In chat interfaces, follow-up queries like "What about that?" or "Does this parameter have a default?" are incomplete without conversational context. The MTRAG benchmark shows dramatic performance degradation when systems fail to maintain context across turns [13].

### Coreference Resolution

- Replace pronouns and references with their explicit referents before retrieval
- "What about that?" becomes "What about the premium plan's pricing?" based on conversation history
- "And the timing?" becomes "What is the timing for the deployment we discussed?"
- Typically done via an LLM call that takes recent chat history + current query and outputs a standalone, resolved query

### Chat History Integration

**Approaches:**

1. **Query rewriting with history:** Pass the last N turns to an LLM with the instruction "Rewrite this follow-up query as a standalone question." This is the standard approach (used by LangChain's `create_history_aware_retriever`, Haystack's conversational pipelines, OpenSearch conversational search)

2. **Semantic memory retrieval:** Store conversation history as vector embeddings. Retrieve only the turns semantically relevant to the current query rather than concatenating the full transcript. Keeps context windows lean and focused

3. **Sliding window:** Include the last 3-5 turns as context. Simple but loses long-range context

4. **Summary-based:** Periodically summarize older conversation into a condensed context block. Balances coverage with token efficiency

### Follow-Up Query Handling

- Detect whether a query is a follow-up (shares topic, uses ellipsis/anaphora) vs. a topic change
- For follow-ups: enrich with conversation context before retrieval
- For topic changes: reset retrieval context to avoid polluting results with stale information
- TREC CAsT benchmarks treat query rewriting and neural re-ranking as essential (not optional) requirements for multi-turn retrieval [14]

### Session Context Management

- Maintain a session state that tracks: current topic, key entities mentioned, user preferences expressed
- Use this state to bias retrieval (e.g., if the user has been asking about "deployment," boost documents related to deployment)
- Consider a TTL (time-to-live) on session context to handle topic drift

| Technique | Description | Priority |
|---|---|---|
| Coreference resolution via query rewriting | Essential for any chat interface | P0 |
| Chat history-aware retrieval (last N turns) | Essential for multi-turn coherence | P0 |
| Semantic memory for long conversations | Valuable for extended sessions | P2 |
| Follow-up vs. topic change detection | Improves quality for multi-turn conversations | P2 |
| Session state tracking | Adds polish with topic/entity tracking | P2 |

---

## 6. Metadata Filtering

**What it does:** Applies structured filters (date ranges, document types, departments, access permissions) to narrow the candidate set before or during vector/keyword search.

**Why it matters:** Without metadata filtering, retrieval may surface documents the user shouldn't see, outdated versions, or irrelevant categories. Pre-filtering reduces the search space and improves both relevance and security.

### Pre-Retrieval Filtering (Recommended)

- Filter the candidate set first, then run vector/keyword search on the narrowed set
- More efficient: smaller search space means faster retrieval
- Ensures access control is enforced at the retrieval boundary, not as a post-hoc check
- Supported natively by all major vector databases (Pinecone, Weaviate, Qdrant, pgvector, Elasticsearch)

### Post-Retrieval Filtering

- Retrieve first, then filter results by metadata
- Simpler to implement but wasteful: you may retrieve top-20 documents and discard half for access control, leaving fewer relevant results
- Use only when pre-filtering would eliminate too many candidates (very restrictive filters on small datasets)

### Access Control Lists (ACLs)

- Tag each document chunk with user/role/group permissions at ingestion time
- At query time, inject the user's permissions as a metadata filter
- Databricks' framework for RAG ACLs [15] and Amazon Bedrock's metadata filtering [16] are production reference implementations
- Critical for enterprise deployments where data segregation is a compliance requirement

### LLM-Extracted Metadata Filters

- Use LLM function calling to extract structured filter criteria from natural language queries
- Example: "Show me the Q3 2025 financial reports for the APAC region" extracts `{date_range: "Q3 2025", category: "financial reports", region: "APAC"}`
- Combine with Pydantic models for type-safe filter extraction [17]

### Graph-Based Metadata Filtering

- Use knowledge graph relationships to expand or refine metadata filters
- Neo4j's approach combines graph traversal with vector search for multi-hop metadata resolution [18]

| Technique | Description | Priority |
|---|---|---|
| Basic metadata filtering (date, type, category) | Essential for relevance and narrowing search space | P0 |
| Access control / ACL filtering | Essential for multi-tenant or enterprise systems | P0 |
| LLM-extracted filters from natural language | Implement when users expect natural language filtering | P1 |
| Graph-based metadata | For complex interconnected metadata | P2 |

---

## 7. Evaluation and Feedback

**What it does:** Measures retrieval quality quantitatively and incorporates user signals to improve the system over time.

**Why it matters:** Without evaluation, you cannot know whether pipeline changes help or hurt. Without feedback loops, the system cannot adapt to actual user needs.

### Retrieval-Level Metrics

| Metric | What it measures | When to use |
|---|---|---|
| **Recall@k** | Proportion of relevant documents in the top-k results | Primary metric -- are you finding the right documents? |
| **Precision@k** | Proportion of top-k results that are relevant | Are you avoiding noise in retrieved context? |
| **MRR (Mean Reciprocal Rank)** | Average reciprocal rank of the first relevant result | When the top-1 result matters most |
| **NDCG@k** | Rank-aware relevance score with position discounting | Best overall metric -- accounts for both relevance and ranking position |
| **MAP (Mean Average Precision)** | Average precision across all relevant documents | When you care about the full ranked list |

**Important caveat (2025 research):** Classical IR metrics like NDCG and MRR assume monotonically decreasing document utility with rank position. Recent work shows this assumption fails for RAG, where heterogeneous position discounting and the LLM's attention patterns affect actual downstream quality [19]. Use retrieval metrics as a proxy, but always validate with end-to-end metrics.

### End-to-End / Generation Metrics

| Metric | What it measures |
|---|---|
| **Faithfulness / Groundedness** | Does the generated answer stay faithful to retrieved context? |
| **Answer Relevance** | Does the answer address the user's actual question? |
| **Hallucination Rate** | Percentage of generated claims not supported by retrieved context |
| **Factual Consistency** | Agreement between generated answer and ground truth |
| **BERTScore / ROUGE** | Textual similarity to reference answers (when available) |

### Evaluation Strategy

1. **Offline evaluation:** Build a test set of (query, relevant_documents, expected_answer) triples. Run retrieval and generation, measure metrics. Gate pipeline changes on metric improvements
2. **LLM-as-judge:** Use an LLM to evaluate retrieval relevance and answer quality at scale when human labels are unavailable. Frameworks: RAGAS, DeepEval, TruLens
3. **Node-level evaluation:** Evaluate each pipeline stage independently (retrieval quality, re-ranking quality, generation quality) to isolate bottlenecks
4. **CI/CD gates:** Automate evaluation in your deployment pipeline -- reject changes that degrade key metrics

### User Feedback Integration

- **Explicit feedback:** Thumbs up/down, relevance ratings on answers. Map negative feedback back to the specific retrieved chunks to identify retrieval failures
- **Implicit feedback:** Click-through on sources, time spent reading, follow-up question patterns (a follow-up reformulation often signals the first answer was inadequate)
- **Feedback-to-training loop:** Aggregate feedback signals to fine-tune embedding models, re-ranking models, or adjust retrieval weights
- **Continuous monitoring:** Track retrieval and generation metrics in production over time to detect drift (new content not being retrieved, changing query patterns)

| Technique | Description | Priority |
|---|---|---|
| Recall@k and NDCG@k measurement | Core retrieval quality metrics | P0 |
| End-to-end faithfulness/hallucination evaluation | Measures generation quality against retrieved context | P0 |
| Offline test set with CI/CD gating | Essential for gating production deployments | P0 |
| LLM-as-judge for scalable evaluation | Scalable alternative when human labels are unavailable | P1 |
| User feedback collection (thumbs up/down) | Essential signal for identifying retrieval failures | P0 |
| Feedback-to-training loop | Becomes essential at scale for continuous improvement | P2 |
| Continuous production monitoring | Essential for detecting drift in production | P0 |

---

## Architecture Summary

The full retrieval pipeline flows as:

```
User Chat Input
    |
    v
[1. Query Processing]
    - Coreference resolution (chat history)
    - Query rewriting / expansion
    - Query decomposition (if multi-part)
    - Intent classification / routing
    |
    v
[2. Metadata Filtering]
    - Access control (ACL)
    - Category / date / type filters
    - LLM-extracted structured filters
    |
    v
[3. Retrieval]
    - Dense retrieval (vector search on embeddings)
    - Sparse retrieval (BM25 on prepared keywords)
    - Fusion (RRF or weighted combination)
    |
    v
[4. Re-ranking]
    - Cross-encoder or lightweight reranker
    - Top-20 -> Top-5
    |
    v
[5. Context Assembly]
    - Deduplication
    - Parent chunk expansion
    - Context window budgeting
    - Relevance-based ordering
    |
    v
[6. LLM Generation]
    - Retrieved context + system prompt + chat history
    |
    v
[7. Evaluation & Feedback]
    - Metric tracking
    - User feedback capture
    - Continuous monitoring
```

## Priority Implementation Order

For a system that already has keywords and vector embeddings with a chat interface:

1. **Start here:** Hybrid retrieval (BM25 on keywords + vector search) with RRF fusion
2. **Add immediately:** Query rewriting with coreference resolution for chat history
3. **Add next:** Cross-encoder re-ranking on top-20 results
4. **Then:** Metadata filtering and access control
5. **Then:** Context assembly with deduplication and parent-child chunks
6. **Then:** Evaluation framework with offline test set
7. **Optimize:** Query decomposition, learned sparse retrieval, LLM-based re-ranking, feedback loops

---

## Sources

[1] [The Query Rewriting Revolution - RAG About It](https://ragaboutit.com/the-query-rewriting-revolution-how-smart-prompt-engineering-is-eliminating-rag-retrieval-failures/)
[2] [RAGRouter: Learning to Route Queries to Multiple Sources](https://arxiv.org/pdf/2505.23052)
[3] [REIC: RAG-Enhanced Intent Classification at Scale - EMNLP 2025](https://aclanthology.org/2025.emnlp-industry.74.pdf)
[4] [SPLADE for Better LLM RAG Retrieval](https://ai.gopubby.com/how-to-use-splade-for-better-llm-rag-retrieval-55f78652de3b)
[5] [Production Retrievers in RAG: Hybrid Search + Re-Ranking](https://machine-mind-ml.medium.com/production-rag-that-works-hybrid-search-re-ranking-colbert-splade-e5-bge-624e9703fa2b)
[6] [Hybrid Search Done Right: BM25 + HNSW + RRF](https://ashutoshkumars1ngh.medium.com/hybrid-search-done-right-fixing-rag-retrieval-failures-using-bm25-hnsw-reciprocal-rank-fusion-a73596652d22)
[7] [Ultimate Guide to Choosing the Best Reranking Model in 2026 - ZeroEntropy](https://www.zeroentropy.dev/articles/ultimate-guide-to-choosing-the-best-reranking-model-in-2025)
[8] [RankRAG: Unifying Context Ranking with RAG in LLMs](https://arxiv.org/html/2407.02485v1)
[9] [Reranking in RAG: Cross-Encoders, Cohere Rerank & FlashRank](https://medium.com/@vaibhav-p-dixit/reranking-in-rag-cross-encoders-cohere-rerank-flashrank-c7d40c685f6a)
[10] [Manage RAG Context Windows: Chunk Strategy Guide 2026](https://markaicode.com/rag-context-window-chunk-strategy/)
[11] [Parent-Child Chunking in LangChain for Advanced RAG](https://medium.com/@seahorse.technologies.sl/parent-child-chunking-in-langchain-for-advanced-rag-e7c37171995a)
[12] [Chunking Strategies for RAG: Early, Late, and Contextual Chunking](https://medium.com/@visrow/chunking-strategies-for-rag-early-late-and-contextual-chunking-explained-with-code-71b88e4709f9)
[13] [Conversational RAG Systems: Building Multi-Turn Dialogue](https://zenvanriel.com/ai-engineer-blog/conversational-rag-systems/)
[14] [5 Proven Strategies to Improve Chatbot Response Accuracy with RAG](https://www.chatrag.ai/blog/2026-01-02-5-proven-strategies-to-improve-chatbot-response-accuracy-with-rag-in-2025)
[15] [Mastering RAG Chatbot Security: ACL and Metadata Filtering - Databricks](https://community.databricks.com/t5/technical-blog/mastering-rag-chatbot-security-acl-and-metadata-filtering-with/ba-p/101946)
[16] [Access Control for Vector Stores using Metadata Filtering - AWS](https://aws.amazon.com/blogs/machine-learning/access-control-for-vector-stores-using-metadata-filtering-with-knowledge-bases-for-amazon-bedrock/)
[17] [Streamline RAG with Intelligent Metadata Filtering - AWS](https://aws.amazon.com/blogs/machine-learning/streamline-rag-applications-with-intelligent-metadata-filtering-using-amazon-bedrock/)
[18] [Graph-based Metadata Filtering for Vector Search in RAG - Neo4j](https://neo4j.com/blog/developer/graph-metadata-filtering-vector-search-rag/)
[19] [Redefining Retrieval Evaluation in the Era of LLMs](https://arxiv.org/html/2510.21440v1)
[20] [RAG Evaluation Metrics Explained: Recall@K, MRR, Faithfulness](https://langcopilot.com/posts/2025-09-17-rag-evaluation-101-from-recall-k-to-answer-faithfulness)
[21] [How to Build a RAG Pipeline from Scratch in 2026 - kapa.ai](https://www.kapa.ai/blog/how-to-build-a-rag-pipeline-from-scratch-in-2026)
[22] [RAG Is Not Dead: Advanced Retrieval Patterns That Actually Work in 2026](https://dev.to/young_gao/rag-is-not-dead-advanced-retrieval-patterns-that-actually-work-in-2026-2gbo)
[23] [Optimizing RAG with Hybrid Search & Reranking - Superlinked](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking)
[24] [RAG Evaluation: 2026 Metrics and Benchmarks - Label Your Data](https://labelyourdata.com/articles/llm-fine-tuning/rag-evaluation)
