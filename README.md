# Design Philosophy Docs

Various writeups about how to build agentic tooling and solutions that have worked well so far.

## Documents

- **Agent Tooling Design Philosophy** — Why agents should use purpose-built libraries instead of CLI applications. The core principle: encode what you know into the tooling.
- **Agent Tooling: MCP vs Scripts vs Hooks** — Decision framework for choosing between native tools, hooks, MCP servers, and agent-executed scripts. Covers security boundaries, hybrid layering, and portability.
- **Agent Workflow Decomposition: Subagent Delegation** — When to keep workflows monolithic vs. decomposing into staged handoffs. Covers handoff contracts, context budgets, reusability, and the review/quality gate pattern.
- **Agent Knowledge Architecture: Retrieval & Discovery** — How to structure agent access to large knowledge corpora. Covers full context injection, index + on-demand retrieval, MCP search, custom RAG, and enriched documents.
- **Output Determinism: Structured Outputs, Free Generation, and Rubric Evaluation** — The three layers of output quality (format compliance, content accuracy, semantic quality) and which mechanisms solve which layer. Decision framework for when to use constrained decoding, free generation, and rubric-based evaluation.
- **Agent Architecture Spectrum: Programmatic Frameworks, Agentic Assistants, and Autonomous Agents** — How to choose between programmatic pipelines, interactive agent assistants, and autonomous agents. Covers orchestration authority, termination conditions, the infrastructure convergence, and the skills/MCP composability model.
