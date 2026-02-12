---
id: agent-knowledge-architecture-retrieval-patterns
title: "Agent Knowledge Architecture: Retrieval, Discovery, and Context Design"
type: reference
scope: architecture
domain:
  - agent-development
  - knowledge-management
  - retrieval
stability: medium
intent: guide
tags:
  - knowledge-base
  - retrieval
  - rag
  - context-window
  - progressive-disclosure
  - mcp
  - agent-skills
  - index-design
  - orchestration
  - decision-framework
source: conversation
confidence: medium
created: 2026-02-11
---

# Agent Knowledge Architecture: Retrieval, Discovery, and Context Design

When agents need access to a large body of reference material — compliance standards, architectural guidance, design patterns, institutional knowledge — how you structure, store, and surface that knowledge determines whether the agent produces grounded, cited, contextual answers or confident-sounding hallucinations. The retrieval mechanism matters, but the harder decisions are about *what* the agent knows about, *when* it goes looking, and *how much* context it pulls in for a given task.

```
Full Context Injection ──→ Index + On-Demand ──→ Tool-Based Retrieval ──→ Custom RAG Pipeline
 simplest, least scalable     best general fit     infrastructure required    highest build cost
```

## Understand Your Corpus

Before choosing an architecture, characterize the knowledge you're exposing. Different corpus types have different retrieval needs.

- **Structured reference material** (compliance standards, security policies, regulatory requirements): numbered documents, formal language, clear boundaries, predictable queries. Each document is authoritative and self-contained. Citations need to point to specific documents and sections.
- **Organic institutional knowledge** (architectural decisions, design patterns, lessons learned, postmortems): varied structure, informal language, fuzzy boundaries. Documents reference each other implicitly. The useful unit of retrieval is sometimes a thread across several documents.
- **Executable reference** (code snippets, configuration templates, runbooks): the agent doesn't just read these — it applies them. Retrieval needs to return complete, usable artifacts, not fragments.
- **Enriched documents** (any of the above with operational metadata layered on): applicability constraints, environment scoping, known exceptions, cross-references, agent-specific guidance notes. The metadata often matters more than the source text.

These aren't mutually exclusive. A mature knowledge base will contain all four types, and the architecture needs to handle that heterogeneity.

## The Two Retrieval Problems

Every knowledge-retrieval design has to solve two distinct problems. Conflating them leads to architectures that solve the easy one well and the hard one not at all.

**The lookup problem:** The agent knows it needs information and searches for it. "What does Standard SEC-087 say about encryption in transit?" This is the problem most retrieval systems are built for. Keyword search, semantic search, and hybrid search all address it.

**The discovery problem:** The agent doesn't know relevant knowledge exists. A user asks the agent to design a retry mechanism. Somewhere in the corpus there's a document about service resilience patterns that includes a hard-won constraint about jittered exponential backoff from a production incident. The agent needs to not just find that document — it needs to realize it should look for it in the first place.

Lookup is a search quality problem. Discovery is an awareness problem. Search quality improves with better retrieval mechanisms. Awareness improves with better orchestration and context design. Most teams over-invest in search quality and under-invest in awareness.

## Decision Heuristics

Start here. Go deeper into the sections below when you need more context.

| Question | Answer | Approach |
|----------|--------|----------|
| Does the entire corpus fit in context (<100K tokens)? | Yes | **Full context injection** — just load it |
| Is the corpus stable, well-bounded, and <500 docs? | Yes | **Index + on-demand files** — curated markdown with a semantic index |
| Do multiple agents or teams need the same corpus? | Yes | **MCP search server** — centralized retrieval |
| Is the corpus maintained by another team? | Yes | **MCP wrapping their API** — but own your enrichment layer |
| Does retrieval quality need tuning beyond keyword search? | Yes | **Custom RAG** — hybrid search, re-ranking, chunk optimization |
| Does the agent need to discover non-obvious connections? | Yes | **Index in context** — passive awareness over active search |
| Is the corpus growing continuously? | Yes | **File-based with automated indexing** — low-friction additions |
| Do citations need to reference specific sections? | Yes | **Full documents on demand** — don't chunk, retrieve whole docs |
| Is operational metadata critical (applicability, exceptions)? | Yes | **Enriched files you control** — not raw source documents |

When multiple rows point to different approaches, that's a signal you need a tiered architecture.

## Full Context Injection

The simplest approach: load everything into the agent's context window. No retrieval mechanism needed — the agent has all the information and can reason over it directly.

**Use full context injection when:**

- The entire corpus fits comfortably in the context window with room for the agent's working memory and output
- The material is dense and interconnected enough that selective retrieval would miss important cross-references
- Query patterns are unpredictable — you can't anticipate which documents will be relevant
- Citation accuracy matters and you want the agent to quote directly from source material

**Practical limits:** At current context window sizes, this works for roughly 50–100 documents of moderate length (2–5 pages each), depending on the model. For the "200 compliance standards at 200–300 lines each" case, this is almost certainly off the table — that's 1–3M tokens before the agent even starts working.

**Tradeoffs:**

- Zero retrieval latency — the agent has everything immediately
- No missed documents — every piece of knowledge is available for every query
- Expensive per invocation — you're paying for the full corpus in tokens on every call, even when 95% of it is irrelevant
- Signal dilution — the more irrelevant context the agent has, the more its attention is spread across material that doesn't matter. This actively degrades response quality for focused queries
- No scalability path — when the corpus outgrows the context window, you need a fundamentally different approach

**Rule of thumb:** If the corpus is small enough that you wouldn't think twice about loading it, load it. The moment you're asking "will this fit?" you've outgrown this approach.

## Index + On-Demand Retrieval (Files)

The agent has a compact semantic index in context — always visible, always available — and retrieves full documents on demand when it identifies something relevant. The index is the discovery layer. The files are the depth layer.

This is the approach with the best general-purpose tradeoff profile for corpora in the 50–500 document range.

**The index is not a table of contents.** A table of contents lists titles. A semantic index gives the agent enough context per entry to make a relevance judgment *without* reading the underlying document. Each entry includes the document ID, a human-written summary of what's in it, the key conclusions or principles, and cross-references to related documents. Think of it as a senior colleague's annotated reading list — not just "what exists" but "what's in it and why you'd care."

```markdown
## Service Resilience
- service-resilience-patterns.md — Retry, circuit breaker, and bulkhead
  patterns as implemented in our platform. Key constraint: all retries
  must use jittered exponential backoff per incident INC-2024-087.
  Related: latency-budget-principles.md, cache-strategy-principles.md
```

**Use index + on-demand when:**

- The corpus is too large for full context injection but well-bounded enough to index comprehensively
- Documents are self-contained enough that retrieving one or a few at a time is useful
- The agent needs passive awareness of what knowledge exists — the index in context solves the discovery problem
- Human curation of the index is feasible and valuable — your domain experts can write better summaries than any automated system
- The corpus changes at a pace where manual index maintenance is practical (weekly to monthly additions)
- Whole-document retrieval preserves reasoning chains that chunking would destroy

**Tradeoffs:**

- The index itself consumes context window space — a 500-document index with meaningful summaries might be 30–50K tokens
- Index quality is the bottleneck — a poorly written index entry means the agent won't find the document. Human curation is the strength and the dependency
- Two-step retrieval adds latency — the agent reads the index, decides what's relevant, then fetches the full doc(s). For multi-document queries, this is multiple sequential tool calls
- Relies on the agent's ability to scan the index and make good relevance judgments. Models are generally good at this, but not perfect — especially for non-obvious connections where the index entry uses different terminology than the query
- Client-dependent — the agent needs filesystem access or a tool that retrieves file contents. Not all runtimes provide this natively

**Index design principles:**

- Each entry should be 2–4 sentences, not just a title
- Include the *conclusion*, not just the *topic* — "recommends write-behind caching for high-write services" is more useful than "discusses caching strategies"
- Include cross-references explicitly — the agent won't infer that the locking doc relates to the caching doc unless you say so
- Tag entries with applicability metadata when relevant — environments, teams, domains
- If an entry's summary is good enough that the agent rarely needs to read the full doc, that's a feature, not a problem

**Rule of thumb:** If your domain experts could curate a useful annotated reading list of your corpus, this approach will work and will likely outperform more complex alternatives.

## MCP Search Server

A persistent MCP server exposes search tools over the corpus. The agent calls `search_standards(query)` or `get_document(id)` and gets back relevant content with metadata. The retrieval logic — keyword matching, semantic search, filtering, re-ranking — is encapsulated in the server.

**Use MCP search when:**

- Multiple agents, skills, or teams need access to the same knowledge base
- The canonical source of the knowledge is managed by another team (a GRC system, a documentation platform, an internal API) and you're wrapping their data
- The corpus is large enough that file-based approaches create client compatibility issues
- You need centralized access control, audit logging, or usage tracking
- Retrieval needs to be consistent across heterogeneous agent runtimes — MCP is the portable interface

**Tradeoffs:**

- Infrastructure dependency — the server must be running for any agent to access knowledge. A server crash means total knowledge loss for all consumers
- Query formulation is the silent failure mode — if the agent doesn't know the right terms to search for, it won't find relevant documents. Unlike the index-in-context approach, there's no passive awareness. The agent only finds what it actively searches for
- Network latency per retrieval, compounded when the agent needs to search multiple times to cross-reference standards
- You inherit the search quality of whatever's behind the server. If the underlying API only offers keyword search, your agents get keyword search. You can build re-ranking and enrichment into your MCP layer, but that's your engineering investment, not theirs
- Organizational coupling — if another team owns the API, your agent capabilities are gated by their roadmap and priorities. Feature requests for better search compete with their other work

**Mitigating the query formulation problem:**

The discovery problem is worst with pure tool-based retrieval because the agent has no ambient awareness of the corpus. Two mitigations help:

- **Pair with a lightweight index.** Even if full retrieval goes through MCP, inject a category-level index into the agent's context — not per-document summaries, just a list of knowledge domains with brief descriptions. This gives the agent enough awareness to formulate better queries.
- **Expose a `list_categories` or `list_documents` tool.** Let the agent browse before searching. The equivalent of scanning a table of contents before using the search bar.

**Rule of thumb:** If the knowledge base is a shared organizational resource with multiple consumers, MCP is the right interface. But don't assume MCP alone solves the discovery problem — pair it with awareness mechanisms.

## Custom RAG Pipeline

You build a full retrieval-augmented generation pipeline: chunk the documents, generate embeddings, store in a vector database (SQLite with sqlite-vec, ChromaDB, Pinecone, etc.), and expose hybrid search (vector + BM25 + metadata filtering) through an MCP tool or skill script.

**Use custom RAG when:**

- The corpus is large enough (1,000+ documents) that index-based approaches don't scale
- Retrieval quality needs to be measurably better than keyword search — you have eval data showing that keyword search misses relevant documents
- The corpus is homogeneous enough that a single chunking strategy works — all documents have similar structure and information density
- You have the engineering capacity to build, tune, and maintain an embedding pipeline
- You need to handle queries where relevant information is distributed across many documents and no single document answers the question

**Tradeoffs:**

- Highest build and maintenance cost of any approach — embedding pipeline, chunking strategy, vector store, re-ranking, index refresh on updates
- **Chunking destroys reasoning chains.** A design considerations document that argues through tradeoffs to reach a conclusion becomes misleading fragments when chunked. The chunk captures a conclusion without the reasoning, or one tradeoff without the counterarguments. This is the fundamental tension of RAG for knowledge corpora where *the argument structure is the value*
- Observability is poor — when the agent gives a wrong answer, debugging "why did retrieval return these chunks?" requires specialized tooling. The database is opaque in a way that a folder of markdown files isn't
- You're maintaining a derived data store. The documents are the source of truth. The vector store is a projection. Keeping them in sync is a process you have to own
- Cross-references break at chunk boundaries. A standard that says "except as noted in SEC-042" becomes meaningless if the chunk doesn't include what SEC-042 says. Naively chunked corpora with heavy cross-referencing perform worse than expected
- Citation provenance can be lossy — chunks need to carry their source document ID, section number, and enough context for the agent to produce accurate citations. This metadata must be engineered into the chunking strategy, not bolted on later

**When RAG outperforms simpler approaches:**

- High-volume, diverse queries where no human could anticipate which documents are relevant
- Corpora with thousands of documents where index curation is impractical
- Queries that need to synthesize across many documents — "summarize all standards that mention encryption" is a query where scanning a 200-document index is slow and RAG's batch retrieval is fast

**Rule of thumb:** If you're reaching for RAG, make sure you've exhausted simpler approaches first. Many teams build RAG pipelines for corpora that would be better served by a well-curated index and file retrieval.

## Enriched Documents: The Layer That Matters Most

Regardless of which retrieval mechanism you choose, the single highest-leverage investment is enriching your documents with operational metadata that isn't in the source material. This applies to every approach above — full context, indexed files, MCP, and RAG all benefit from richer documents.

**What enrichment looks like:**

```markdown
---
id: SEC-087
title: Encryption in Transit
applies_to: [cloud, on-prem, hybrid]
environments: [production, staging]
not_applicable: [dev, sandbox]
owner: Network Security Team
last_reviewed: 2025-01-15
related: [SEC-042, SEC-091, NET-003]
common_exceptions:
  - Internal east-west on isolated segments (RA-2024-031)
agent_notes: >
  This standard is often cited for internal APIs but the
  enforcement threshold differs by environment classification.
  Always check data classification of what's being transmitted
  before flagging — internal telemetry on isolated segments
  has a standing exception.
---
```

The `agent_notes` field is where institutional knowledge lives — the things a knowledgeable human would mention that the document itself doesn't say. Exception context, common misapplications, relationships to other documents, gotchas.

**Why enrichment is architecturally independent:**

- If you're doing full context injection, enriched front-matter makes the agent's reasoning more accurate
- If you're using an index, the enrichment feeds better index summaries
- If you're behind an MCP server, the enrichment travels with the document in search results
- If you're doing RAG, the enrichment metadata becomes filterable fields that dramatically improve retrieval precision
- If you later migrate from files to RAG to MCP, the enrichment carries forward untouched

**Why you — not the source team — own enrichment:**

The team that maintains the canonical documents (GRC, legal, platform engineering) writes authoritative source text. They do not write agent-optimized operational context. The knowledge about which environments a standard applies to, which exceptions exist, and how standards interact in practice — that's your team's domain knowledge. It's the value-add that no upstream API or document management system provides, and it's the layer that makes the difference between an agent that parrots the standard and one that applies it correctly.

**Rule of thumb:** Start enrichment before you choose a retrieval mechanism. The enrichment work is valuable regardless of architecture, and the process of doing it often clarifies which retrieval approach you actually need.

## Orchestrating Discovery

Giving the agent access to knowledge isn't enough. The agent also needs to *use* it at the right moments, and for the discovery problem specifically, it often won't unless the orchestration creates explicit checkpoints.

### The Awareness Spectrum

Four levels of orchestration exist, each progressively more explicit about when the agent should consult the knowledge base:

**Implicit availability.** The knowledge base exists. Tools are available. The agent uses them if it thinks to. This works for casual augmentation where missing a relevant document is acceptable. It does not work for anything where a missed document has consequences.

**Prompted awareness.** The system prompt describes what's in the knowledge base with enough specificity that the agent can connect incoming queries to available knowledge domains. "You have access to guidance on caching patterns, service communication, observability, and common anti-patterns in our stack." This gives the agent semantic hooks — when someone says "retry mechanism," the agent connects that to "service communication" and goes looking. The index-in-context pattern provides prompted awareness automatically.

**Explicit workflow steps.** The orchestration includes a mandatory retrieval phase. The agent doesn't decide whether to check the knowledge base — the workflow requires it. "Before proposing any architectural design, check the knowledge base index for existing patterns, prior decisions, or anti-patterns related to the problem space. Cite any relevant guidance. If nothing relevant exists, note that explicitly." This is what you want for compliance, architecture reviews, security assessments — anywhere that "I didn't know we had guidance on that" is an unacceptable outcome.

**Structured orchestration with routing.** The agent analyzes the task, identifies which knowledge domains are likely relevant, retrieves from those domains specifically, then proceeds with the task grounded in what it found. This is the most token-intensive approach but produces the most consistently grounded outputs. Suitable for complex, multi-domain tasks where the agent needs to synthesize across several knowledge areas.

### Matching Awareness to Stakes

| Stakes if knowledge is missed | Minimum awareness level |
|-------------------------------|------------------------|
| Inconvenient but harmless | Implicit availability |
| Suboptimal output quality | Prompted awareness |
| Incorrect advice given confidently | Explicit workflow steps |
| Compliance, legal, or safety exposure | Structured orchestration with routing |

Under-orchestrating high-stakes workflows is the most dangerous failure mode. The agent gives a confident, well-structured answer that misses a critical standard because nothing in the orchestration triggered retrieval. The answer looks right. It isn't.

### Anti-Patterns in Discovery Orchestration

- **"You have access to a knowledge base"** as the only instruction. This is the equivalent of telling a junior engineer "there's a Confluence." Technically true. Practically useless.
- **Assuming the agent will search proactively.** Without a trigger — either an index entry that catches its attention or an explicit instruction to search — the agent will default to its parametric knowledge and reason from first principles. This produces plausible but ungrounded output.
- **Over-relying on query formulation.** If the user asks about "data residency," the agent needs to know that might touch standards on data sovereignty, cloud hosting, encryption at rest, cross-border transfer, and backup/DR. The agent's ability to decompose a concept into the right set of search queries is limited. The index and the orchestration need to bridge that gap.

## Tiered Architecture for Production

For corpora that serve multiple workflows with varying stakes and scope, a single retrieval mechanism is usually insufficient. A tiered architecture matches each layer to a specific purpose.

**Tier 1 — Ambient awareness (index in context).** A compact semantic index of the full corpus, injected into the agent's context or available as a readily-loaded reference. Solves the discovery problem. The agent always knows what knowledge exists, even if it hasn't retrieved the details. Cost: 15–50K tokens depending on corpus size.

**Tier 2 — On-demand depth (retrieval tool).** An MCP tool, skill script, or file-read capability that fetches full document text by ID or search query. The agent uses Tier 1 to identify candidates, then retrieves full text for the ones it needs. Solves the lookup problem with full document fidelity.

**Tier 3 — Workflow-scoped pre-loading.** For workflows that predictably need specific subsets of the corpus (a cloud architecture review always needs the 15–20 standards on cloud, networking, and data classification), pre-load those documents into the agent's context. Not a separate agent per domain — a context package per workflow that supplements the universal index and retrieval tools. Eliminates retrieval latency for the most common access patterns.

**How the tiers interact:**

- Tier 1 provides discovery across the full corpus
- Tier 3 provides immediate depth for known-relevant domains
- Tier 2 fills the gap when the agent discovers something relevant via Tier 1 that isn't covered by Tier 3

This degrades gracefully. If Tier 2 is unavailable (MCP server down, no file access), the agent still has Tier 1 awareness and Tier 3 pre-loaded documents — it can note "Standard SEC-087 appears relevant but I couldn't retrieve the full text" rather than silently missing it. If Tier 3 isn't configured for a particular workflow, Tier 1 + Tier 2 still cover the full corpus, just with extra retrieval steps.

## Maintenance and Lifecycle

The retrieval mechanism you can actually maintain is better than the one you build and let rot.

### Update Cadence by Approach

| Approach | Update workflow | Practical cadence |
|----------|----------------|-------------------|
| Full context injection | Replace the loaded text | Immediate — changes are live next invocation |
| Index + files | Edit the file, update the index entry | Low friction — minutes per document |
| MCP wrapping external API | No update needed on your side | Automatic — but you don't control quality |
| MCP over your own corpus | Update source files, re-expose | Low friction if file-backed, moderate if database-backed |
| Custom RAG | Update source, re-chunk, re-embed, re-index | High friction — requires pipeline execution |

### Staleness Risk

For knowledge bases where accuracy matters (compliance, security, legal), staleness is the primary failure mode of any retrieval architecture. A wrong answer from an agent citing an outdated standard is worse than no answer.

- **File-based approaches** have visible staleness — you can see file modification dates, diff against source, and automate freshness checks
- **RAG databases** have invisible staleness — the embeddings look fine but the underlying text may be outdated, and there's no easy way to diff the vector store against the source of truth
- **API-backed approaches** have zero staleness for source text but may have stale enrichment metadata if that layer is maintained separately

The approach that's easiest to keep current is often the right one, even if its retrieval quality is slightly worse in theory. For compliance use cases specifically, an easily maintained system that's always accurate beats a sophisticated system that occasionally serves outdated guidance.

### The Enrichment Maintenance Workflow

Enrichment metadata (applicability, exceptions, agent notes) needs a lightweight process for staying current:

- When a document is updated at the source, someone reviews and updates the enrichment
- When an exception is granted or revoked, the relevant document's enrichment is updated
- Quarterly review of agent notes for staleness and accuracy
- The enrichment is version-controlled alongside the documents, not in a separate system

This is a curation task, not an engineering task. It belongs with the domain experts who understand the operational context, not with the platform team that maintains the retrieval infrastructure.

## Scaling Inflection Points

Different approaches hit scaling limits at different corpus sizes. Recognize the inflection points so you can plan migrations rather than hitting walls.

- **Full context injection** runs out around 50–100 documents (model and context-window dependent). Beyond that, you need retrieval.
- **Index-in-context** runs out around 500–1,000 documents, where the index itself exceeds comfortable context window budgets. Beyond that, you need retrieval over the index — at which point you're effectively building search anyway.
- **File-based retrieval** runs out when your agent client can't efficiently search the file tree, or when the number of files makes sequential scanning impractical. This is client-dependent — an agent with `grep` and filesystem access can handle thousands of files. An agent limited to reading one file at a time struggles at hundreds.
- **Custom RAG** scales to tens of thousands of documents but the maintenance cost scales with it. Chunking strategies, embedding model updates, and re-indexing pipelines become ongoing engineering commitments.

For most organizations, the practical corpus of genuinely useful institutional knowledge — when cleaned up and deduplicated — fits comfortably under 500 documents. The problem is rarely scale. It's that the knowledge isn't written down, or it's scattered, or it's stale. The act of curating a clean corpus with good enrichment is itself enormously valuable, independent of which retrieval mechanism serves it.

## Portability Across Agent Runtimes

Different retrieval approaches impose different requirements on the agent client, and this matters when the same knowledge base needs to serve multiple environments.

| Approach | Client requirement | Portability |
|----------|-------------------|-------------|
| Full context injection | Context window capacity | High — any client that accepts a system prompt |
| Index + files | Filesystem access or file-read tool | Medium — depends on client capabilities |
| MCP search | MCP client support | Medium-high — growing ecosystem support |
| Custom RAG via MCP | MCP client support | Medium-high — same as MCP |
| Custom RAG via skill script | Shell/code execution | Medium — agent needs to run code |
| Embedded RAG in application | Application-level integration | Low — tightly coupled to one runtime |

If your knowledge base needs to work across Claude Code, a PydanticAI agent, and a web-based chat interface, MCP provides the most consistent interface. If it only needs to work in one environment, optimize for that environment's strengths.

## The Core Design Principle

The best retrieval mechanism for agents is the one that puts the most relevant context into the fewest tokens while preserving the reasoning and nuance that makes the knowledge useful. This is progressive disclosure applied to agent cognition: always-visible awareness at the top (the index), detail on demand in the middle (document retrieval), and pre-staged depth for known workflows at the bottom (context packages).

Over-engineering retrieval for a corpus that would be better served by a curated index is the most common mistake. Under-investing in orchestration — assuming the agent will search at the right moments without being guided — is the most dangerous one.
