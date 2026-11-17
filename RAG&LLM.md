# RAG & MCP

A practical study guide for AI applications

Embeddings • Search • Vector databases • Chunking • Caching  
Resilience • RAG pipeline • Model Context Protocol  
Examples throughout: SQL Server DBA knowledge base and HR policy chatbot  
Prepared from your RAG2.0 notes, reorganized and expanded  
16 November 2026

---

## How to use these notes

Read the guide in order the first time. Each section starts with the simple meaning, then explains what happens, why it matters, and how it applies to a real SQL Server DBA assistant.

- First pass: understand the bold terms and the end-of-section summaries.
- Second pass: follow the HR policy chatbot example through the full RAG pipeline.
- Practice: answer the review questions without looking at the notes.
- Build: implement a small project using a few PDF documents before scaling it up.

> Key idea: RAG finds trusted information and gives it to an LLM; MCP gives an AI application a standard way to interact with external tools and data.

---

## Contents

- Part 1 — Retrieval-Augmented Generation (RAG)
  - 1. What RAG is and why it is used
  - 2. Embeddings and vector representations
  - 3. Similarity measures and semantic search
  - 4. Keyword search: TF-IDF and BM25
  - 5. Hybrid search and reranking
  - 6. Vector databases and indexing
  - 7. Document ingestion, chunking and metadata
  - 8. The complete RAG workflow
  - 9. Caching in AI and databases
  - 10. Failure handling and production resilience
  - 11. RAG quality, security and operations
- Part 2 — Model Context Protocol (MCP)
  - 12. What MCP is and how it differs from agents
  - 13. JSON-RPC and MCP architecture
  - 14. Tools, Resources and Prompts
  - 15. Transports, hosts, clients and servers
  - 16. Building an MCP server with Python
  - 17. Roots, Sampling, Elicitation and protocol updates
  - 18. Security and operational best practices
- Part 3 — Revision
  - 19. End-to-end examples, glossary and interview questions

---

# Part 1 — Retrieval-Augmented Generation (RAG)

## 1. What is RAG?

RAG stands for Retrieval-Augmented Generation. It is an approach in which an application first retrieves relevant information from a knowledge source and then gives that information to a Large Language Model (LLM) so it can answer more accurately and with source support.

Think of an HR chatbot that has access to 500 company policy PDFs. If an employee asks, "How many days of paternity leave can I take?", the application searches the policies, finds the relevant section, and then gives that section to the LLM to answer based on the document.

### Without RAG vs With RAG

| Without RAG | With RAG |
|---|---|
| The LLM mainly relies on information learned during training and the prompt. | The LLM receives relevant information retrieved from your own documents. |
| It may not know a company's latest internal policy. | It can answer from the policy text retrieved for that question. |
| It may confidently produce unsupported information. | The application can require the answer to be supported by retrieved sources. |

### Why use RAG?

- Use private or organization-specific information that was not part of the model's training.
- Keep information more current by updating the knowledge base rather than retraining the LLM.
- Provide source references so users can verify answers.
- Reduce the chance of unsupported answers when retrieval, prompting and validation are designed well.

### What RAG does not guarantee

- RAG does not guarantee correctness. It can retrieve the wrong passage or the LLM can misinterpret it.
- RAG does not automatically make documents secure. Permissions must be enforced during retrieval.
- RAG is not model training or fine-tuning. It supplies external context at answer time.

> Key idea: RAG = retrieve relevant evidence + augment the prompt with that evidence + generate an answer.

---

## 2. Embeddings and vector representations

### 2.1 What is an embedding?

An embedding model converts text (or another supported input such as an image) into a list of numbers called a vector. The numbers are arranged so that items with related meanings tend to be near each other in vector space.

#### Example

| Text | What the embedding should capture |
|---|---|
| "I love learning Python." | Programming / learning context |
| "Python programming is interesting." | A related programming meaning |
| "I like eating biryani." | Food context; less related to the first two |

The example is conceptual. The model does not store a human-readable label such as "programming" in each coordinate. The meaning is distributed across the vector.

### 2.2 What does 384-dimensional mean?

A 384-dimensional embedding contains 384 numeric values. Each position is one dimension (coordinate) in the vector space. These values are not milliseconds, percentages or direct measurements. They are learned numerical features representing meaning.

Example only (first five values shown):

```text
[0.25, -0.18, 0.73, 0.42, -0.11, ...]
```

Actual vector: 384 values

Dense vector means most or all positions contain numeric values. A sparse vector, by contrast, has many zero values and only a relatively small number of non-zero values.

### 2.3 Same model for documents and questions

For a vector search system, document chunks and user questions should normally be embedded using the same embedding model and compatible configuration. The query vector is then compared with the stored document vectors.

- Choose an embedding model based on language, domain, quality, latency, cost and supported context length.
- Record the model name/version and vector dimension with the index.
- If you change the embedding model, plan to regenerate document embeddings and rebuild or maintain a separate index.

> Key idea: A 384-dimensional vector simply has 384 values. More dimensions do not automatically mean better retrieval.

---

## 3. Similarity measures and semantic search

### 3.1 Semantic search

Semantic search finds content based on meaning rather than requiring the exact same words. For example, a search for "How can I improve database speed?" may retrieve a passage titled "SQL Server performance tuning" even if it does not contain that exact phrase.

### 3.2 How similarity is calculated

The search system compares the query vector with stored vectors. Common comparison measures include cosine similarity, dot product and Euclidean distance. Which one is appropriate depends on the embedding model and system design.

#### Dot product example

For two 3-dimensional vectors, multiply values in matching positions and add the results:

```text
A = [2, 3, 4]
B = [1, 2, 3]
Dot product = (2×1) + (3×2) + (4×3) = 20
```

The real calculation uses every dimension in the vectors. A raw dot product is affected by vector magnitude. If both vectors are normalized to length 1, their dot product equals cosine similarity.

#### Cosine similarity

Cosine similarity measures the angle between two vectors. It focuses on direction rather than magnitude. In many embedding search systems, a higher cosine similarity indicates a closer semantic match.

### 3.3 Top-k retrieval

After scoring candidate chunks, the retriever returns the top-k results. If k=5, it returns the five highest-ranked candidates. The best value depends on document structure, question type, context size and application goals.

| Search question | Illustrative result | Score |
|---|---|---:|
| How do I improve SQL Server performance? | SQL Server performance tuning | 0.95 |
| How do I improve SQL Server performance? | Database query optimization | 0.89 |
| How do I improve SQL Server performance? | PostgreSQL backup | 0.30 |

Scores above are illustrative, not universal thresholds. Similarity scores from different models or distance metrics should not be compared as if they were on the same scale.

### 3.4 Clustering

Clustering groups vectors that are close to one another. It is useful for discovering themes in a large collection, organizing documents, identifying duplicate-like content or exploring a dataset.

> Key idea: Embedding creates vectors; similarity search compares vectors; clustering groups related vectors.

---

## 4. Keyword search: TF-IDF and BM25

Keyword search ranks documents using the words in the query. It is especially useful when exact terms matter, such as error numbers, SQL object names, product codes, policy identifiers or phrases.

### 4.1 TF-IDF

TF-IDF means Term Frequency–Inverse Document Frequency.

- Term Frequency (TF): how often a term appears in a document.
- Inverse Document Frequency (IDF): how rare or informative the term is across the document collection.

A word appearing in almost every document, such as "the", is usually less informative than a rare term such as "deadlock". TF-IDF gives more weight to terms that are frequent in a particular document but rare across the full collection.

### 4.2 BM25

BM25 is a widely used keyword-ranking function related to TF-IDF. It considers term frequency, how rare a term is across the collection, and document length. It includes practical controls so repeated terms do not dominate too much.

### 4.3 Compare the search approaches

| Method | What it matches | Strong use case | Limitation |
|---|---|---|---|
| TF-IDF | Term frequency and rarity | Basic keyword scoring | Less sophisticated ranking than BM25 in many search tasks |
| BM25 | Query terms, rarity, term frequency and length normalization | Search engines, exact terms and document retrieval | May miss relevant passages expressed with different words |
| Dense embeddings | Semantic relationships / meaning | Natural-language questions and paraphrases | May miss exact identifiers or rare terms |

Example: Query = "SQL Server backup failure". BM25 can find a page containing those exact words. Embeddings may also find "Why did my database backup job not complete?" even if the wording differs.

> Key idea: Use keyword search when exact words matter; use semantic search when meaning matters.

---

## 5. Hybrid search and reranking

### 5.1 Hybrid search

Hybrid search combines keyword retrieval (often BM25) with vector retrieval. It is useful because each method catches things the other can miss.

```text
User question
   ├── BM25 keyword search ─────┐
   └── Embedding/vector search ─┤
                               └── Combine and rank candidates
```

The two result lists need to be combined carefully. Their raw scores may be on different scales, so systems often use a rank-fusion method such as Reciprocal Rank Fusion (RRF), or a calibrated score combination.

### 5.2 Reranking

A reranker is a second-stage model that examines the query and each retrieved candidate together, then reorders the candidates by relevance. A common design is:

1. Retrieve a wider candidate set quickly, for example 20–50 chunks.
2. Use a reranker to score query–chunk pairs more precisely.
3. Pass only the best few chunks to the LLM.

This can improve relevance, but adds latency and computational cost. Evaluate whether the improvement is worth it.

### 5.3 Metadata filtering

Metadata is descriptive information stored with each chunk, for example document name, page, department, policy type, effective date, version, access group and source URL. Metadata filters can narrow retrieval before or after ranking.

Example: for an employee in the India entity, filter to the India policy version and the employee's permitted access groups. Do not rely on the LLM to enforce access rules.

---

## 6. Vector databases and indexing

### 6.1 What does a vector database store?

A vector database stores vectors and the information needed to find and use them. A record commonly includes:

- Vector: the embedding values.
- Chunk text or a reference to the original text.
- ID: a unique identifier for the chunk.
- Metadata: source document, page, version, permissions and other fields.

If 500 policy PDFs each produce 10 chunks, the collection contains about 5,000 chunk vectors, not just 500 document vectors.

### 6.2 Why indexing is needed

Without an index, an exact nearest-neighbor search may compare the query with every vector. This is called brute-force or flat search. It can be perfectly reasonable for small collections, but it becomes expensive at scale.

An index organizes vectors to make candidate retrieval faster. Many indexes use Approximate Nearest Neighbor (ANN) search: they trade a small amount of recall for faster search.

### 6.3 Common index approaches

| Index | Simple explanation | Trade-off / note |
|---|---|---|
| Flat / brute force | Compare query with every vector | Exact result; can become expensive at scale |
| HNSW | Builds a multi-layer graph; navigate through nearby nodes | Fast high-recall search in many workloads; uses memory and needs tuning |
| IVF | Partitions vectors into clusters/lists and searches selected lists | Can be memory-efficient; search quality depends on selected lists/probes and training |
| LSH | Hashes similar vectors so likely neighbours fall into related buckets | Useful in some approximate-search settings; not the default choice in every vector DB |

The best index depends on vector count, dimensions, update rate, memory, latency target, recall target and database implementation. Avoid treating fixed counts such as 5,000 or 100,000 vectors as universal guidelines.

### 6.4 HNSW in plain language

Imagine a map with express roads and local roads. HNSW starts at a high-level sparse graph to move quickly toward the target area, then descends through denser levels to find close vectors. It does this efficiently without scanning the entire dataset.

### 6.5 IVF in plain language

Imagine dividing 5,000 vectors into 50 groups. A query first identifies the most promising groups, then searches within those groups instead of scanning everything. Searching more groups can improve recall at the cost of more work.

> Key idea: Embedding model = creates vectors. Vector database = stores/searches them. Index = accelerates nearest-neighbor search.

---

## 7. Document ingestion, chunking and metadata

### 7.1 Ingestion pipeline

Ingestion is the process of preparing source material for retrieval. A practical pipeline is:

1. Collect source files (PDF, DOCX, HTML, text, database records, etc.).
2. Extract text and structure; use OCR when a PDF is scanned as an image.
3. Clean extraction problems while preserving headings, tables, page numbers and important formatting.
4. Split the content into meaningful chunks.
5. Attach metadata and access-control information.
6. Generate embeddings for each chunk.
7. Store chunk text, vector and metadata in the search system.
8. Record document version and ingestion status so updates can be managed.

### 7.2 What is chunking?

Chunking divides a long document into smaller sections. The system normally embeds and retrieves these chunks rather than embedding an entire 100-page document as one unit.

Example policy sections:

- Chunk 1: Eligibility
- Chunk 2: Paternity leave entitlement
- Chunk 3: Application process
- Chunk 4: Required documents

### 7.3 Chunk size and overlap

Chunk size is the amount of content in each chunk. It may be measured in characters, tokens, sentences or structural units. Chunk overlap repeats a small part of one chunk in the next so that meaning spanning boundaries is preserved.

| Situation | Effect |
|---|---|
| Too small | A rule may be separated from its condition or exception. |
| Balanced | A coherent paragraph or section stays together. |
| Too large | Several unrelated topics are mixed together. |

Example with overlap: Chunk 1 contains words 1–100; Chunk 2 contains words 81–180. Words 81–100 appear in both chunks.

A chunk size of 1,000 characters and overlap of 200 characters can be a starting experiment in a simple text-splitting exercise, but it is not a universal best setting. Token-based chunking and structure-aware chunking are often better in real systems.

### 7.4 Chunking strategies

| Strategy | How it works | Best fit |
|---|---|---|
| Character / word-based | Split after a chosen number of characters or words | Simple text and quick prototypes |
| Sentence-based | Keep sentence boundaries intact | Narrative text and FAQs |
| Paragraph-based | Split by paragraph boundaries | Well-structured documents |
| Recursive splitting | Try larger separators first (sections, paragraphs, sentences), then smaller ones | General-purpose documents |
| Structure-aware | Use headings, sections, tables, pages or document layout | Policies, manuals and technical documentation |
| Semantic chunking | Split when the subject or meaning changes | Documents with uneven topic boundaries |

### 7.5 Preserve context

- Keep section headings with the text they describe.
- Do not separate a table from its column headings.
- Keep a rule together with its conditions, exceptions and effective date.
- Store source filename, page number, section and document version.
- For long answers, consider retrieving adjacent chunks or parent sections when needed.

> Key idea: A good chunk is not merely a fixed number of characters. It is a self-contained piece of information that can answer a question.

---

## 8. The complete RAG workflow

### 8.1 Two phases: indexing and answering

| Offline / ingestion phase | Online / query phase |
|---|---|
| Load documents and extract text. | Receive the user's question. |
| Clean and split into chunks. | Check identity, permissions and query constraints. |
| Attach metadata and generate embeddings. | Generate a query embedding (if using vector search). |
| Store chunks, vectors and metadata; build indexes. | Search using vector, BM25 or hybrid retrieval. |
| Track versions and update changed documents. | Optionally rerank and select the best chunks. |
|  | Build a prompt with question + retrieved context. |
|  | LLM generates a grounded response; return citations/sources. |

### 8.2 HR policy chatbot example

```text
INGESTION
500 policy PDFs → text extraction → chunks → metadata → embeddings → vector/keyword indexes

QUESTION
Employee: “How many days of paternity leave are allowed?”
   ↓
Search policy chunks
   ↓
Retrieve relevant policy section
   ↓
Augment prompt with source text
   ↓
LLM answers using only retrieved evidence
```

### 8.3 A useful grounded prompt

```text
Answer the question using only the supplied policy context.
If the context does not contain the answer, say that the policy information
was not found. Do not invent an entitlement. Mention relevant policy
source sections when available.
```

A prompt helps guide behaviour, but it is not a substitute for retrieval quality, permissions, validation or monitoring.

### 8.4 Important RAG terms

| Term | Meaning |
|---|---|
| Retriever | Component that finds candidate chunks. |
| Top-k | Number of candidates selected. |
| Context window | Amount of input/output tokens the LLM can handle. |
| Reranker | Second-stage model that reorders retrieved candidates. |
| Grounding | Making the answer rely on supplied evidence. |
| Citation / provenance | Information identifying where a retrieved fact came from. |
| Hallucination | A generated claim that is unsupported or fabricated. |
| Ingestion | Preparing and indexing source documents. |
| Re-indexing | Rebuilding or updating the searchable representation. |

---

## 9. Caching in AI and databases

Caching stores reusable results so the system does not repeat expensive work unnecessarily. Different caches store different things; do not confuse them.

| Cache type | What it stores | What it can avoid |
|---|---|---|
| LLM response cache | Previously generated final answers | Repeated LLM generation for a matching or sufficiently similar request |
| SQL Server plan cache | Compiled execution plans | Repeated query compilation |
| Embedding cache | Text-to-vector results | Repeated embedding generation for identical inputs |
| Query embedding cache | Previously generated vectors for user queries | Embedding API/model work for repeated queries |
| Vector-search / retrieval cache | Previously retrieved chunk IDs or search results | Repeated nearest-neighbour retrieval |
| Prompt cache (provider feature) | Reusable prompt-prefix processing, depending on provider | Repeated processing of shared prompt prefixes |

### 9.1 Cache hit and cache miss

- Cache hit: a suitable result is already available, so the application reuses it.
- Cache miss: no suitable result is available, so the application performs the operation and may store the result.

### 9.2 Exact-match vs semantic cache

An exact-match cache reuses a result when the key matches exactly (often after normalizing the input). A semantic cache uses similarity to decide whether a previous question is close enough to reuse the answer.

### 9.3 Example flow

1. User asks a question.
2. The application checks whether a query embedding is cached; otherwise it generates one.
3. The application checks whether suitable retrieval results are cached; otherwise it searches.
4. The application checks whether a safe, valid final answer is cached; otherwise it calls the LLM.
5. Store reusable results with appropriate expiry and version information.

This is an illustrative sequence. Real systems may order cache checks differently and may not use every cache.

### 9.4 Cache invalidation and safety

- Include document version, effective date and tenant/user permission scope in cache keys where relevant.
- Expire or invalidate cached answers when source policies change.
- Do not share one user's private result with another user.
- Avoid caching sensitive data unless retention, encryption and access rules permit it.
- Measure cache hit rate, latency saved, stale-result rate and correctness.

> Key idea: A cache makes a system faster only when the cached result is still correct, current and authorized.

---

## 10. Failure handling and production resilience

A production RAG application may depend on three major services: the vector/keyword retrieval store, the embedding service and the LLM. Design for partial failure instead of assuming every dependency is available.

### 10.1 Failure scenarios

| Service failure | What stops working | Possible fallback |
|---|---|---|
| Vector database unavailable | Vector similarity retrieval | Replica/secondary region; separately maintained BM25 index; safe cached retrieval |
| Embedding service unavailable | New query vectors cannot be generated | Cached query vector where safe; preconfigured compatible embedding service and its own matching document index; BM25 |
| LLM unavailable | Natural-language answer generation | Secondary LLM; verified cached answer; return retrieved source excerpts with a clear notice |

### 10.2 Critical compatibility rule

Do not generate a query vector with Model B and compare it with document vectors generated by Model A just because both vectors have the same number of dimensions. Different models generally use different vector spaces and semantic mappings.

### 10.3 Cascading fallback

```text
Level 1: Vector / hybrid retrieval + primary LLM
   ↓ if a dependency fails
Level 2: Available retrieval path + available LLM
   ↓ if LLM is unavailable
Level 3: Previously verified, fresh and authorized answer
```

A fallback must exist independently. BM25 cannot search a vector database that is down unless a separate keyword index and source text are available.

### 10.4 Graceful degradation

Graceful degradation means the application continues to provide a reduced but useful service when part of the system fails. For example, if the LLM is unavailable, show the original matching policy excerpts instead of inventing an answer.

### 10.5 Reliability techniques

| Technique | Meaning | Example |
|---|---|---|
| Health check | Check whether a dependency is reachable and responding properly | Monitor vector DB connection and query latency |
| Timeout | Stop waiting after a defined period | Do not let one stalled LLM request hang the whole app |
| Retry with exponential backoff | Retry transient failures with increasing delays | 1 second, then 2, then 4 seconds, with jitter |
| Circuit breaker | Temporarily stop calling a repeatedly failing service | Open after repeated failures; periodically test recovery |
| Bulkhead | Isolate resources so one dependency cannot exhaust everything | Separate worker pools for embedding and LLM calls |
| Failover | Switch to a standby service or provider | Use a secondary LLM endpoint |
| Observability | Collect logs, metrics and traces | Track errors, latency, tokens, retrieval quality and cache hits |

### 10.6 Circuit breaker states

- Closed: requests flow normally.
- Open: requests are blocked temporarily because the dependency is failing.
- Half-open: a limited test request checks whether the dependency has recovered.

Retry only transient errors. Repeatedly retrying invalid credentials, malformed requests or incompatible vector dimensions will not fix them.

### 10.7 Non-negotiable fallback rules

- Security: every fallback must enforce the same access permissions.
- Freshness: do not return an old policy answer after the policy has changed.
- Accuracy: if no reliable source is available, say so; do not ask the LLM to guess.
- Transparency: tell the user when the system is in a degraded mode.

---

## 11. RAG quality, security and operations

### 11.1 Evaluate retrieval separately from generation

A fluent answer can still be wrong if the retriever supplied the wrong evidence. Evaluate both stages.

| Area | What to check | Example measure |
|---|---|---|
| Retrieval | Did the correct source chunk appear in the results? | Recall@k, Precision@k, MRR or nDCG |
| Reranking | Did relevant chunks move above irrelevant ones? | Ranking metrics and human review |
| Generation | Is the answer supported by the retrieved context? | Faithfulness / groundedness review |
| Answer usefulness | Did it answer the user's actual question? | Human rating, task success, answer relevance |
| Operations | Is it fast and reliable? | Latency, error rate, cost per query, uptime |

### 11.2 Security checklist

- Authenticate users and enforce document-level permissions before returning chunks.
- Treat retrieved documents as untrusted data. A document may contain malicious instructions; do not let retrieved text override system/developer instructions.
- Protect API keys and credentials using a secret manager or secure environment configuration.
- Use least-privilege database accounts; make read-only access the default for a DBA assistant.
- Require explicit approval for destructive or production-changing actions.
- Log access and actions while avoiding unnecessary storage of sensitive content.

### 11.3 Practical monitoring dashboard

| Component | Useful monitoring |
|---|---|
| Document ingestion | Files processed, failures, OCR errors, chunks created, stale documents |
| Embedding service | Availability, latency, token/compute cost, failures |
| Vector / keyword store | Availability, query latency, index size, recall checks |
| Retriever / reranker | Top-k quality, filter behaviour, reranking latency |
| LLM | Latency, token usage, errors, refusal rate, groundedness |
| Cache | Hit ratio, stale entries, invalidation count |
| End-to-end app | Success rate, p50/p95 latency, user feedback, cost per answer |

---

# Part 2 — Model Context Protocol (MCP)

## 12. What is MCP?

MCP stands for Model Context Protocol. It is an open protocol that standardizes how an AI application connects to external tools, data sources and services. It provides a common interface so an AI app can discover and use external capabilities in a consistent way.

Example: An AI assistant can explain SQL Server queries from its model knowledge. If connected to an approved SQL Server MCP server, it may also request live database status or a list of failed SQL jobs.

### Without a database connection vs With an approved MCP connection

| Without a database connection | With an approved MCP connection |
|---|---|
| The assistant can explain how to check database status. | The assistant can ask an MCP tool to retrieve the actual status. |
| It cannot see live server information by itself. | The MCP server can execute an approved read-only query and return the result. |

> Key idea: MCP is a standard connection and interaction protocol. It is not an LLM, database, agent or automation platform by itself.

### 12.1 AI agent vs MCP

| AI agent / host orchestration | MCP |
|---|---|
| Interprets the user's goal and decides what steps may be needed. | Defines a standard way to discover and call external capabilities. |
| Plans, reasons and selects tools (depending on the agent design). | Exposes tools, resources and prompts through a protocol. |
| Controls the overall task flow. | Carries requests and responses between client and server. |

Analogy: The agent is like a manager deciding what needs to be done; MCP is the agreed communication interface; the SQL Server is the system that supplies the requested information.

An agent can use MCP, but an MCP server does not automatically make an application an agent.

---

## 13. JSON-RPC and MCP architecture

### 13.1 What is JSON-RPC?

JSON-RPC 2.0 is a lightweight remote procedure call protocol. One application sends a JSON request naming a method and parameters; the other returns a result or an error.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_database_status",
    "arguments": {"database": "EmployeeDB"}
  }
}
```

Illustrative response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{"type": "text", "text": "EmployeeDB is ONLINE"}]
  }
}
```

| Field | Meaning |
|---|---|
| jsonrpc | JSON-RPC protocol version, normally 2.0 |
| id | Request identifier used to match the response |
| method | Operation being requested |
| params | Arguments for that operation |

A response carries the matching request ID and either a result or an error. Notifications are messages that do not expect a response.

### 13.2 Host, Client and Server

| Component | Meaning | Example |
|---|---|---|
| MCP Host | The AI application the user interacts with; it manages the overall experience and may run multiple clients. | Cursor or a compatible AI desktop application |
| MCP Client | The component inside the host that communicates with a particular MCP server. | A client connection managed by the host |
| MCP Server | A program/service exposing capabilities such as database queries, files or repository operations. | A SQL Server MCP server |

```text
User
 ↓
MCP Host (AI application)
 ↓ manages
MCP Client
 ⇄ JSON-RPC over a supported transport
MCP Server
 ↓ uses approved driver/API
SQL Server / files / GitHub / other service
```

A host can connect to several servers, for example one for files, one for GitHub and one for a database. Each server has a focused responsibility.

---

## 14. Tools, Resources and Prompts

These are the three main server-side primitives. Memorize the simple distinction: Tools do; Resources provide; Prompts guide.

| Primitive | Purpose | SQL Server DBA example | Common methods |
|---|---|---|---|
| Tools | Perform an operation or action | Check database status; list failed jobs; run an approved read-only query | tools/list, tools/call |
| Resources | Expose information for the AI application to read | Database schema, configuration, runbook or error-log excerpt | resources/list, resources/read |
| Prompts | Provide reusable prompt templates or workflows | A template for analysing blocking or preparing a health check | prompts/list, prompts/get |

### 14.1 Tools: take care with actions

A tool can read data or perform a side effect. A tool definition should clearly describe its name, purpose, inputs and expected output. The server must validate arguments and enforce permissions; otherwise, the tool may become a risky or destructive capability.

For a DBA assistant, start with read-only tools such as `get_database_status` and `get_failed_jobs`. Add write operations only after implementing strong authorization, confirmation, auditing and rollback safeguards.

### 14.2 Resources: context and information

Resources are data that an MCP server makes available to the client. They are commonly identified by URIs. Examples include a schema description, a runbook, or a configuration file. A resource is primarily a source of context, not a command.

### 14.3 Prompts: reusable templates

A prompt is a reusable interaction template. For example: “Review the supplied SQL Server wait statistics, identify likely bottlenecks, and explain what evidence supports each recommendation.”

> Key idea: Tools = do something. Resources = read information. Prompts = reusable instructions.

---

## 15. Transports, hosts, clients and servers

### 15.1 Transport choices

| Transport | How it works | Typical use |
|---|---|---|
| stdio | The host launches a local server process and communicates through standard input/output streams. | Local tools running on the same computer |
| Streamable HTTP | The client communicates with a remote MCP server over HTTP; responses may be streamed using SSE. | Remote or cloud-hosted MCP servers |

HTTP+SSE was used by earlier MCP versions for remote communication. Streamable HTTP replaced that older transport in the 2025-03-26 specification. Legacy implementations may still support HTTP+SSE.

### 15.2 What are Cursor and Claude?

Cursor and Claude applications are examples of AI applications that can act as MCP hosts when the relevant MCP integration is supported. They are not MCP server types. Exact support depends on the app and version.

### 15.3 MCP server categories by connected system

- Filesystem server: approved file operations.
- GitHub server: repository, issue and pull-request information/actions.
- PostgreSQL or SQL Server server: approved database operations.
- Google Drive server: access to permitted documents.
- Browser/search server: retrieve web information.
- Slack server: access to permitted messages or actions.

These are categories based on what the server connects to, not transport types.

---

## 16. Building an MCP server with Python

You do not need to implement JSON-RPC parsing and transport handling from scratch. An MCP SDK provides building blocks for registering server capabilities and connecting them over a supported transport.

| Requirement | Why it is needed |
|---|---|
| Python | Language used to implement the server logic |
| MCP Python SDK | Provides MCP server/client abstractions and protocol handling |
| Database driver | Connects Python to SQL Server, for example an appropriate ODBC-based driver |
| MCP host | The AI application that connects to the server |
| Transport | stdio for local process communication or Streamable HTTP for remote service |
| Authentication and authorization | Controls who can connect and what operations they may perform |

### 16.1 Suggested first DBA project

Build a small server with one read-only tool called `get_database_status`.

1. Create a dedicated SQL Server login with minimum read-only permissions.
2. Write a Python function that connects to the database and returns a limited status result.
3. Register the function as an MCP tool using the Python SDK.
4. Connect the server to a compatible MCP host such as Cursor.
5. Ask: "Check the status of EmployeeDB."
6. Verify that the returned value matches the result from SQL Server.
7. Add logging, timeout handling, input validation and safe error messages.

### 16.2 Example tools to add later

| Tool name | Purpose | Suggested permission level |
|---|---|---|
| get_database_status | Return database state and basic properties | Read-only |
| get_failed_jobs | List recent failed SQL Agent jobs | Read-only |
| get_database_size | Return data/log file sizes | Read-only |
| get_blocking_summary | Return current blocking summary | Read-only |
| get_backup_status | Show latest backup details | Read-only |
| execute_readonly_query | Run a restricted SELECT query | Carefully restricted read-only |

> Key idea: For a first MCP project, build safe read-only tools. Do not start with DROP, DELETE, ALTER or production write actions.

---

## 17. Roots, Sampling, Elicitation and protocol updates

These features describe ways an MCP client/host can provide context or user interaction to an MCP server. They are not the same category as the server-side Tools, Resources and Prompts.

### 17.1 Roots

Roots are URI-based locations that a client can expose to indicate relevant filesystem/workspace locations. For example, a client may identify a SQL log directory or DBA scripts directory as relevant.

```text
file:///C:/SQLLogs/
file:///C:/DBA/Scripts/
```

Important: a root is context or a location hint, not a security boundary. The server must independently enforce filesystem permissions and path restrictions.

### 17.2 Sampling

Sampling historically allowed an MCP server to ask the client/host to use its LLM to generate content. Example: a server collects SQL performance metrics and requests AI analysis through the client.

Protocol-version note: In the 2026-07-28 MCP specification, Sampling is marked deprecated for new implementations; the guidance is to integrate directly with an LLM provider API instead. You may still see older examples in tutorials and legacy code.

### 17.3 Elicitation

Elicitation allows a server to request additional information or confirmation from the user through the client. It is useful when the request is incomplete or potentially sensitive.

Example: User asks, "Delete old backup files." The system should clarify the folder and date range, show exactly what will be deleted, and obtain appropriate confirmation before proceeding.

| Elicitation mode | Purpose | Example |
|---|---|---|
| Form elicitation | Collect structured, non-sensitive information from the user | Ask which database or date range to inspect |
| URL elicitation | Send the user to a secure external flow | OAuth authorization or another secure credential-entry flow |

Do not ask users to enter passwords or API secrets into ordinary elicitation forms. Use secure authorization mechanisms.

### 17.4 What changed in the 2026-07-28 specification?

The source document's statement that Roots and Sampling were both deprecated is not accurate for the official 2026-07-28 specification. The official documentation still describes Roots as a client feature. Sampling is deprecated for new implementations.

The 2026-07-28 specification also changes the protocol architecture: it removes the old initialize/notifications/initialized handshake and protocol-level HTTP sessions, and introduces a stateless design.

| Concept | Current study takeaway |
|---|---|
| Roots | Still documented as a client feature; provides workspace/location context. Not a security boundary. |
| Sampling | Deprecated for new implementations in protocol version 2026-07-28; understand it for legacy systems. |
| Elicitation | Still supported; used to collect input or confirmation. |
| MRTR | Newer interaction pattern for cases where a request needs additional input before it can complete. |
| Initialize handshake | Older lifecycle concept; removed in the 2026-07-28 stateless protocol revision. |

Always check the protocol version and SDK version used by the host/server you are implementing. MCP has evolved, and older tutorials may describe earlier lifecycle methods.

### 17.5 Multi Round-Trip Requests (MRTR) in simple terms

A tool may need one more piece of information before it can finish. In the newer stateless protocol design, the server can return an input-required result. The client gathers the requested input, and the request continues with that input.

Example: a tool needs the user to choose a database and confirm a maintenance action. It asks for the missing input; the host gathers it; the request continues with that input.

---

## 18. MCP security and operational best practices

- Use least privilege. Give a database MCP server only the permissions it needs.
- Prefer read-only operations initially. Separate diagnostic tools from change tools.
- Validate all tool arguments on the server, including database names, paths, query limits and date ranges.
- Require explicit approval for destructive or production-changing operations.
- Authenticate remote clients and authorize each operation.
- Protect secrets; never hard-code passwords or expose them in prompts, logs or ordinary forms.
- For local HTTP servers, bind to localhost where appropriate and validate Origin headers to reduce DNS-rebinding risk.
- Use TLS for remote connections and follow the protocol's authorization guidance.
- Log who requested an action, what tool ran, when it ran and the outcome.
- Treat content returned from files, databases and websites as untrusted input.
- Set timeouts, rate limits and output-size limits.
- Keep SDKs and server dependencies updated; test compatibility with the host's protocol version.

> Key idea: MCP standardizes communication; it does not automatically make a tool safe. Security is still the responsibility of the host, server and connected system.

---

# Part 3 — Revision and practice

## 19. End-to-end examples

### 19.1 Example A: HR policy chatbot using RAG

1. HR uploads policy PDFs.
2. Ingestion extracts text and keeps page/section details.
3. Documents are split into coherent chunks.
4. Metadata is attached: country, department, effective date, policy type and access group.
5. Embeddings are generated and stored with the chunks.
6. A search retrieves relevant passages for the employee's question.
7. The LLM answers only from the retrieved evidence.
8. The response includes citations to the source sections when possible.

### 19.2 Example B: SQL Server DBA assistant using MCP

1. DBA asks: "Show me failed SQL Agent jobs."
2. AI host interprets the request.
3. MCP client discovers the server's available tools.
4. Host selects the approved `get_failed_jobs` tool.
5. Tool runs an approved read-only query against SQL Server.
6. Result is returned to the AI host.
7. Host summarizes the findings and explains the current state.

### 19.3 RAG vs MCP

| RAG | MCP |
|---|---|
| A pattern for answering with retrieved knowledge. | A protocol for connecting AI applications to external capabilities. |
| Main flow: retrieve relevant content, then generate. | Main flow: discover and invoke tools/resources/prompts. |
| Typical components: loaders, chunkers, embeddings, indexes, retriever, LLM. | Typical components: host, client, server, JSON-RPC, transport. |
| Example: answer from SQL Server runbooks. | Example: ask SQL Server for current database status. |

They can be used together. An MCP server could expose a search tool for a RAG knowledge base, while the AI host uses the returned information to answer a question.

---

## 20. Quick glossary

| Term | Plain-English meaning |
|---|---|
| LLM | Large Language Model that generates and interprets text. |
| Embedding | Numerical vector representation produced by an embedding model. |
| Dimension | One numeric coordinate in a vector. |
| Dense vector | Vector where most positions contain values. |
| Sparse vector | Vector with many zero values; often used in lexical representations. |
| Semantic search | Search by meaning rather than exact wording. |
| Cosine similarity | Similarity measure based on the angle between vectors. |
| Dot product | Multiply corresponding vector coordinates and sum the products. |
| TF-IDF | Keyword weighting using term frequency and inverse document frequency. |
| BM25 | Keyword-based document ranking function. |
| Hybrid search | Combines keyword and vector retrieval. |
| Reranker | Second-stage model that reorders candidate results. |
| Vector database | System for storing and searching embeddings and associated data. |
| HNSW | Graph-based approximate nearest-neighbour index. |
| IVF | Cluster/partition-based vector index. |
| LSH | Hashing approach that groups likely similar vectors. |
| Chunk | A smaller section of a source document. |
| Chunk overlap | Repeated content between adjacent chunks. |
| Metadata | Information describing a chunk, such as source, page, version or permissions. |
| Top-k | Number of top search results returned. |
| Cache hit | A reusable result was found in cache. |
| Cache miss | No suitable cached result was found. |
| Circuit breaker | Stops calls temporarily to a failing dependency. |
| Graceful degradation | Continues with reduced functionality when part of the system fails. |
| MCP | Model Context Protocol; standard interface for AI applications and external capabilities. |
| JSON-RPC | JSON-based request/response protocol. |
| Host | AI application managing the user experience and MCP clients. |
| Client | MCP component that communicates with a server. |
| Server | Program exposing MCP capabilities. |
| Tool | Callable operation exposed by an MCP server. |
| Resource | Data exposed for reading by an MCP client. |
| Prompt | Reusable interaction template exposed by a server. |
| stdio | Local process communication using standard input/output. |
| Streamable HTTP | HTTP transport for remote MCP communication. |
| Elicitation | Request for additional user input or confirmation. |
| Sampling | Legacy MCP capability for requesting LLM generation from the client; deprecated for new implementations in 2026-07-28. |
| Roots | Client-provided workspace/location context; not a security boundary. |
| MRTR | Multi Round-Trip Requests; request more input and continue a request. |

---

## 21. Interview-style questions and short answers

| Question | Short answer |
|---|---|
| What is RAG? | RAG retrieves relevant external information and supplies it to an LLM so the answer is grounded in that context. |
| What is an embedding? | A numerical vector representation of text or other supported content, used to compare semantic relationships. |
| What does 384 dimensions mean? | The embedding contains 384 numeric coordinates; they are not time or percentage measurements. |
| What is semantic search? | Finding information by meaning, even when the query and document use different words. |
| What is cosine similarity? | A vector similarity measure based on the angle between two vectors. |
| What is dot product? | Multiply corresponding vector coordinates and sum the products; with unit-normalized vectors it equals cosine similarity. |
| What is the difference between TF-IDF and BM25? | Both are keyword-ranking approaches; BM25 adds practical ranking controls such as term-frequency saturation and document-length normalization. |
| What is hybrid search? | Combining keyword search such as BM25 with vector-based semantic search. |
| What is a vector database? | A system that stores embeddings and associated metadata and retrieves similar vectors. |
| What is HNSW? | A graph-based approximate nearest-neighbour index that navigates through layers to find nearby vectors. |
| What is IVF? | An index that partitions vectors into groups and searches selected groups. |
| Why do we chunk documents? | To retrieve focused, meaningful sections instead of passing entire large documents to the LLM. |
| Why is chunk overlap used? | To preserve context that might otherwise be lost at chunk boundaries. |
| What is a reranker? | A second-stage model that reorders retrieved candidates by relevance to the query. |
| What does an embedding cache store? | Previously generated text embeddings so duplicate inputs do not need to be embedded again. |
| What does an LLM response cache store? | Previously generated answers, subject to freshness and permission checks. |
| What if the embedding provider is down? | Use a safe cached query vector, a separately indexed compatible fallback model, or keyword search. |
| Can Model B query vectors search Model A document vectors? | Not generally. Different models use different vector spaces; use a matching index or another retrieval method. |
| What is graceful degradation? | Providing a reduced but useful service when one or more dependencies are unavailable. |
| What is MCP? | An open protocol that standardizes how AI applications connect to external tools, resources and prompts. |
| What is JSON-RPC? | A JSON-based protocol for sending method requests and receiving results or errors. |
| What are the MCP architecture components? | Host, Client and Server. |
| What are MCP Tools? | Callable operations that allow an AI application to perform actions through a server. |
| What are MCP Resources? | Data sources that an AI application can read through a server. |
| What are MCP Prompts? | Reusable prompt templates exposed by a server. |
| Is Cursor an MCP server? | No. A compatible Cursor application acts as an MCP host. |
| What is stdio? | A transport for local process communication through standard input and output. |
| What is Streamable HTTP? | An HTTP transport used for remote MCP communication. |
| What is Elicitation? | A way for an MCP server to request missing information or confirmation from the user through the client. |
| What is Sampling in the 2026-07-28 specification? | A capability for requesting model generation from the client; it is deprecated for new implementations. |
| Are Roots a security boundary? | No. They provide location context; the server must enforce permissions independently. |
| What is MRTR? | A pattern where a server requests additional input, the client gathers it, and the original request is retried with that input. |

---

## 22. Self-test

1. Which component converts a chunk of text into a vector?  
   Answer: Embedding model

2. What does top-k=5 mean?  
   Answer: Return five top-ranked candidates

3. Which method is best suited to exact error codes and rare identifiers?  
   Answer: Keyword search such as BM25

4. What combines semantic and keyword retrieval?  
   Answer: Hybrid search

5. What is the purpose of chunk overlap?  
   Answer: Preserve context across chunk boundaries

6. Which vector index uses a navigable graph?  
   Answer: HNSW

7. What should you do if the embedding model changes?  
   Answer: Re-embed documents into a compatible index; do not mix unrelated vector spaces

8. Which cache stores final generated answers?  
   Answer: LLM response cache

9. What should happen if the LLM is unavailable but retrieval works?  
   Answer: Show verified source excerpts or use a secondary LLM; clearly disclose degraded mode

10. In MCP, which component does the user interact with?  
    Answer: Host / AI application

11. Which MCP primitive performs an action?  
    Answer: Tool

12. Which MCP primitive exposes read-only context?  
    Answer: Resource

13. Which transport is used for local process communication?  
    Answer: stdio

14. What should be required before a destructive DBA action?  
    Answer: Authorization, explicit confirmation, validation and audit logging

15. Is MCP the same as an AI agent?  
    Answer: No. An agent reasons/plans; MCP standardizes external connections.

---

## 23. Suggested learning and practice plan

| Stage | What to learn | Practical exercise |
|---|---|---|
| 1. Foundations | LLM, RAG, embeddings, vectors and similarity | Embed 10 DBA FAQ sentences and compare similar/different questions |
| 2. Search | TF-IDF, BM25, vector search and hybrid search | Search a small collection of SQL troubleshooting documents |
| 3. Document preparation | PDF loading, OCR, chunking, overlap and metadata | Load 5 policy PDFs and inspect the generated chunks |
| 4. Vector store | Store vectors, use filters and try an index | Retrieve top-k chunks and compare exact vs approximate search |
| 5. RAG answer generation | Prompting, grounding, citations and evaluation | Build a chatbot that answers only from the source documents |
| 6. Production readiness | Caching, monitoring, permissions and fallbacks | Disable one dependency at a time and test graceful degradation |
| 7. MCP basics | JSON-RPC, host/client/server, tools/resources/prompts | Build a Python MCP server with one read-only tool |
| 8. MCP operations | stdio, Streamable HTTP, elicitation and security | Connect the server to an MCP host and audit each tool call |

> Key idea: Recommended first project: a read-only SQL Server DBA knowledge assistant. Start with 5–10 runbooks, then add a small set of read-only database tools after the RAG flow is reliable.

---

## 24. Important source notes and updates

This guide reorganizes and expands the uploaded RAG2.0.docx. Repeated passages have been consolidated. Illustrative examples and study explanations have been retained, while additional topics have been added to clarify resilience, security and MCP architecture.

The MCP section was checked against the official Model Context Protocol documentation for specification version 2026-07-28. In particular, Sampling is deprecated for new implementations in that version.

Official MCP references:

- https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/architecture/index.mdx
- https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/docs/2026-07-28/learn/architecture.mdx
- https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/transports/streamable-http.mdx
- https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/patterns/mrtr.mdx
- https://blog.modelcontextprotocol.io/posts/2026-07-28/

Note: RAG architecture choices vary by framework, vector database and workload. Treat example chunk sizes, top-k values, similarity scores and index-size guidance as starting points to test, not fixed rules.

---

End of document.
