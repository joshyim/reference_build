# Basic Context Assembly for RAG

## What This Document Covers

Basic context assembly is the step between retrieval and LLM generation in a RAG pipeline. After your vector search returns the top-k most relevant chunks, you need to assemble those chunks into a prompt that the LLM can use to generate a grounded answer. This document covers the mechanics, trade-offs, and practical patterns for an MVP implementation.

---

## 1. What Basic Context Assembly Involves

The pipeline is:

1. **User sends a query**
2. **Query is converted to an embedding** — the user's text query is run through the same embedding model used during ingestion, producing a vector representation that can be compared against stored document vectors
3. **Vector search** returns the top-k most similar chunks from your vector store
4. **Context assembly** concatenates those chunks into a single text block
5. **Prompt construction** combines system instructions + assembled context + user query into a final prompt
6. **LLM generates** a response grounded in the provided context

The assembly step (4-5) is where you make decisions about how many chunks to include, how to order them, how to format them, and how to frame them for the LLM. At MVP, this is literally string concatenation with a prompt template wrapped around it.

---

## 2. Top-k Selection

### How to Choose k

There is no universal correct k. It depends on your chunk size, context window, and query nature. Practical guidance:

- **Start with k=5.** Most common default in production systems and frameworks (LangChain, LlamaIndex, etc.). Five chunks of 300-500 tokens each gives you 1,500-2,500 tokens of context — manageable and usually sufficient for single-topic queries.
- **k=3** works for narrow factoid lookups where you expect one chunk to contain the answer and want minimal noise.
- **k=10** is reasonable when queries are broad or when your chunks are small (200 tokens or less). However, more chunks means more noise and higher cost.
- **Beyond k=10**, you are almost certainly introducing irrelevant content unless you add a reranker. Chroma's "context rot" research (July 2025) found that retrieval performance degrades as context length increases, even on straightforward tasks [1].

### Similarity Score Thresholds

Top-k alone always returns k chunks, even if the bottom results are barely relevant. A score threshold adds a quality floor:

- **Cosine similarity ranges** vary by embedding model and domain:
  - **0.7-0.8** for technical/structured documents
  - **0.5-0.6** for narrative/conversational content
  - **0.3** as an absolute floor
- **For MVP:** Start with top-k only (k=5) and skip thresholds. Add a threshold later when you observe the LLM using irrelevant chunks. When you do, start at 0.5 for cosine similarity and tune from there.
- **Edge case:** If threshold filtering leaves you with zero chunks, return an "I don't have enough information" response or fall back to the LLM's parametric knowledge with a disclaimer.

### The Trade-off

| Too few chunks (k=1-2) | Too many chunks (k=10+) |
|---|---|
| May miss the answer entirely | Introduces noise and irrelevant content |
| Fast and cheap | Higher token cost |
| Low risk of confusing the LLM | "Context rot": LLM performance degrades |
| Works for simple factoid queries | Can cause hallucination from tangential content |

---

## 3. Chunk Ordering in the Prompt

### The "Lost in the Middle" Problem

Liu et al. (2023) [2] demonstrated a U-shaped performance curve: LLMs perform best when relevant information appears at the **beginning** or **end** of the context, and performance degrades significantly for information placed in the **middle** of long contexts.

### Ordering Strategies

| Strategy | How it works | When to use |
|---|---|---|
| **Relevance-first** (descending) | Most relevant chunk first, least relevant last | Default for MVP. Simple and generally effective. |
| **Reverse relevance** (ascending) | Least relevant first, most relevant last | Exploits recency bias — LLM pays most attention to what it read last. |
| **Lost-in-the-middle mitigation** | Most relevant at start AND end, least relevant in the middle | Best for larger k (7+) where middle chunks risk being ignored. |
| **Chronological/document order** | Ordered by source document position | Use when chunks come from the same document and narrative flow matters. |

### MVP Recommendation

**Use relevance-first (descending) ordering.** Simplest, default in every framework, and with k=5 the "lost in the middle" effect is minimal because there is not much middle. The problem becomes material at ~10+ chunks.

---

## 4. Prompt Structure / Template

### The Three-Part Structure

Every RAG prompt has three components in this order:

1. **System instructions** — Role, behavior rules, how to use the context
2. **Retrieved context block** — The assembled chunks
3. **User query** — The actual question

### Where to Place What

- **System message:** Instructions on how to use the context (e.g., "Answer only based on the provided context. If the context doesn't contain the answer, say so."). Include citation format requirements here, before the model sees any chunks.
- **User message:** The retrieved context block followed by the user's actual query.

This separation — instructions in system, context + query in user — produces more consistent results than putting everything in one message.

### MVP Prompt Template

```
# System message
You are a helpful assistant that answers questions based on the provided context.
Use ONLY the information in the context below to answer. If the context does not
contain enough information to answer the question, say "I don't have enough
information to answer that."

# User message
Context:
---
[Chunk 1]
---
[Chunk 2]
---
[Chunk 3]
---

Question: {user_query}
```

### Including Metadata

For MVP, include at minimum the **source identifier** (document title or filename) with each chunk. This helps the LLM attribute information and helps users verify answers.

```
Context:
---
[Source: Product Requirements - Q3 2025.pdf]
The new authentication flow requires OAuth 2.0 with PKCE...
---
[Source: Architecture Decision Record #14]
We selected PostgreSQL for the user database because...
---
```

### Delimiter Strategy

Use clear, consistent separators between chunks:
- **Triple dashes** `---` (simple, works well)
- **XML-style tags** `<document>...</document>` (Claude parses these well)
- **Numbered headers** `[Document 1]`, `[Document 2]`

The key is consistency. Dumping chunks with no delimiters forces the model to guess boundaries, which degrades quality.

---

## 5. Chunk Format

### Should Chunks Include Source/Title?

**Yes.** Snowflake's finance RAG research found that appending document-level context (e.g., "This is from XYZ Corp 10-Q filing") to each chunk helped models track which information came from where, reducing confusion when multiple sources were present [3].

However, per-chunk LLM-generated summaries actually hurt performance in the same study — keep it simple with basic metadata.

### Labeling Approach for MVP

Use numbered chunks with source metadata:

```
[1] (Source: onboarding-guide.pdf, Section: Getting Started)
New employees should complete the IT setup checklist within their first
three days. This includes requesting VPN access, setting up 2FA, and...

[2] (Source: it-policies.pdf, Section: VPN Access)
VPN access requires manager approval and takes 24-48 hours to provision.
Submit requests through the IT portal at...
```

**Why numbered?** Enables citation in the LLM's response (e.g., "According to [1], new employees should...") and makes debugging easier.

### Does Formatting Affect LLM Performance?

Yes. Preserving markdown structure (headers, bullet lists) in chunks gives the model structural cues. Do not strip formatting from chunks during ingestion unless you have a specific reason to.

---

## 6. Context Length Considerations

### The Budget Problem

Your context window must accommodate four competing demands:

| Component | Typical Token Budget | Notes |
|---|---|---|
| System prompt | 200-500 tokens | Instructions, role definition |
| Retrieved context | 1,500-4,000 tokens | Your k chunks |
| Chat history | 0-3,000 tokens | Previous turns (if multi-turn) |
| Generation buffer | 500-2,000 tokens | Space for the LLM's response |

### Rules of Thumb for MVP

- **Keep total assembled context under 8,000 tokens** even if your model supports more. Quality over quantity.
- **Reserve at least 1,000 tokens for generation.** If the model runs out of space, responses get truncated mid-sentence.
- **For MVP without chat history:** ~300 tokens for system prompt + ~3,000 tokens for retrieved context + ~1,000 tokens for generation = ~4,300 tokens total. Fits comfortably in any modern model.
- **When you add multi-turn chat (P1.1):** Chat history competes directly with retrieved context. Cap history at 3-5 recent turns and summarize older turns.

### MVP Approach

Do not build a dynamic token budgeter for MVP. Instead:
1. Fix your chunk size at ~300-500 tokens during ingestion
2. Fix k=5
3. This gives you a predictable ~1,500-2,500 tokens of context every time
4. Add your system prompt and query — well under any modern model's limit
5. Revisit budgeting when you add chat history or encounter long-context issues

---

## 7. When Basic Assembly Breaks Down

### Symptoms That You Have Outgrown Basic Assembly

| Symptom | Likely Cause | Solution |
|---|---|---|
| LLM answers using wrong/irrelevant chunks | Top-k retrieval is too noisy | Add a **reranker** (cross-encoder) |
| Same information repeated across chunks | Overlapping chunks from similar documents | Add **deduplication** |
| Answer requires info split across chunk boundaries | Chunks too small, or answer spans sections | **Parent-child retrieval** |
| Chunks lack context to be understood alone | Chunk is mid-paragraph with no surrounding info | **Contextual chunking** or prepend document summaries |
| Model ignores middle chunks | Too many chunks, lost-in-the-middle | Reduce k, add reranking, or use start-and-end ordering |
| LLM misses exact keywords, IDs, or codes | Vector search misses lexical matches | Add **hybrid search** (vector + BM25) |
| Token costs are spiking | Too much context per query | **Context budgeting** with dynamic allocation |

### The Typical Upgrade Path

1. **MVP:** Basic top-k with concatenation (you are here)
2. **First upgrade:** Hybrid search (vector + keyword) and a reranker
3. **Second upgrade:** Parent-child chunking, deduplication, metadata filtering
4. **Third upgrade:** Dynamic context budgeting, query routing, multi-step retrieval

Do not jump ahead before you have evidence from real queries that the current stage is failing.

---

## 8. Simple Implementation Patterns

### Pattern A: Minimal Python (No Framework)

```python
def assemble_context(query: str, vector_store, embedding_model, k: int = 5) -> str:
    # 1. Embed the query
    query_embedding = embedding_model.encode(query)

    # 2. Retrieve top-k chunks
    results = vector_store.query(
        query_embeddings=[query_embedding],
        n_results=k
    )

    # 3. Assemble context (relevance-first — results are pre-sorted)
    context_parts = []
    for i, (doc, metadata) in enumerate(zip(results["documents"][0], results["metadatas"][0])):
        source = metadata.get("source", "Unknown")
        context_parts.append(f"[{i+1}] (Source: {source})\n{doc}")

    context_block = "\n---\n".join(context_parts)

    # 4. Build the prompt
    system_prompt = (
        "You are a helpful assistant. Answer the question using ONLY the provided context. "
        "If the context does not contain enough information, say so. "
        "Cite sources using [1], [2], etc."
    )

    user_prompt = f"Context:\n---\n{context_block}\n---\n\nQuestion: {query}"

    return system_prompt, user_prompt
```

### Pattern B: LangChain (Stuff Documents Chain)

```python
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer the question based only on the following context:\n\n{context}"),
    ("human", "{input}"),
])

document_chain = create_stuff_documents_chain(llm, prompt)
retrieval_chain = create_retrieval_chain(retriever, document_chain)

response = retrieval_chain.invoke({"input": "How do I reset my password?"})
```

The "stuff" strategy stuffs all retrieved documents into the prompt at once. This is the right one for MVP. Alternatives (map-reduce, refine, map-rerank) exist for when context exceeds the window, which should not happen at MVP if you control k and chunk size.

### Which Pattern for MVP

**Use Pattern A** (manual concatenation + direct API call). ~20 lines of code, full control, no framework abstraction hiding behavior. Use LangChain if you are already invested in that ecosystem, but for MVP the framework adds dependency without adding value.

---

## Key Takeaways for MVP

1. **k=5, chunk size 300-500 tokens, relevance-first ordering.** Start here.
2. **Use clear delimiters and numbered sources** in your context block.
3. **System instructions in system message, context + query in user message.** Keep them separate.
4. **Keep total context under 8k tokens.** Shorter is better, even with large context windows.
5. **Include basic metadata** (source name) with each chunk. Skip everything else for now.
6. **Do not add a reranker, deduplication, or dynamic budgeting** until you have evidence of specific failure modes from real usage.
7. **The prompt template matters more than you think.** Explicit instructions to "only use provided context" and "say when you don't know" dramatically reduce hallucination.

---

## Sources

[1] [Context Rot: How Increasing Input Tokens Impacts LLM Performance — Chroma Research (July 2025)](https://research.trychroma.com/context-rot)
[2] [Lost in the Middle: How Language Models Use Long Contexts — Liu et al. (2023)](https://arxiv.org/abs/2307.03172)
[3] [Long-Context Isn't All You Need: How Retrieval & Chunking Impact Finance RAG — Snowflake](https://www.snowflake.com/en/engineering-blog/impact-retrieval-chunking-finance-rag/)
