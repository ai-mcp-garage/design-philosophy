---
id: agent-architecture-spectrum-programmatic-vs-agentic-vs-autonomous
title: "Agent Architecture Spectrum: Programmatic Frameworks, Agentic Assistants, and Autonomous Agents"
type: reference
scope: architecture
domain:
  - agent-development
  - system-design
  - orchestration
stability: medium
intent: guide
tags:
  - agent-frameworks
  - architecture
  - orchestration
  - autonomous-agents
  - agentic-assistants
  - programmatic-pipelines
  - skills
  - mcp
  - convergence
  - decision-framework
source: conversation
confidence: medium
created: 2026-02-14
---

# Agent Architecture Spectrum: Programmatic Frameworks, Agentic Assistants, and Autonomous Agents

When building systems that use language models, the first architectural decision isn't which model to use or how to prompt it. It's who controls the execution flow. A developer's code can call the model as a function within a predetermined pipeline. An interactive agent can reason about what to do and let a human steer. An autonomous agent can execute against a specification with no human in the loop. These are different architectures with different failure modes, different infrastructure requirements, and different strengths.

They're also converging. The frameworks are adding agent loops. The agents are extracting into SDKs. The infrastructure kits are serving multiple points on the spectrum simultaneously. The underlying capability layer — skills, tools, MCP, retrieval — is increasingly shared. What remains distinct is the orchestration layer: who decides what happens next, who decides when it's done, and what happens when something goes wrong.

This document provides a framework for choosing where on the spectrum a given use case belongs, understanding what changes when you move along it, and recognizing when it's time to shift.

## The Spectrum

```
Programmatic Frameworks ←——→ Agentic Assistants ←——→ Autonomous Agents
  developer is outer loop      LLM is outer loop,      LLM is outer loop,
  model is a function call     human is in the loop     no human in the loop
```

This isn't three boxes with hard boundaries. It's a continuous spectrum with specific tradeoff axes. A system can sit at different points on the spectrum for different parts of its workflow — a programmatic pipeline that delegates open-ended subtasks to an agentic assistant, or an autonomous agent that escalates ambiguous decisions to a human-in-the-loop step.

The position on the spectrum is determined by the answers to three questions:

1. **Who holds orchestration authority?** Who decides what action to take next?
2. **Who holds the termination condition?** Who decides when the task is done?
3. **What happens when something fails?** Who recovers?

## The Key Architectural Axes

### Orchestration Authority

**Programmatic frameworks.** The developer's code is the outer loop. The model is called within it — as a function, a pipeline step, a tool in a DAG. The developer decides when to call the model, what to ask it, and what to do with the response. The execution flow is predetermined. The model fills in blanks within a structure the developer designed.

```
developer code → call model → process response → call model again → ...
```

The model has no agency over the workflow. It doesn't decide to use a tool, search for information, or change approach. It does what it's asked, returns a result, and the code continues. This is the pattern of PydanticAI agents in simple mode, LangChain sequential chains, and any application that calls an LLM API with structured prompts and processes the response programmatically.

**Agentic assistants.** The LLM is the outer loop. It reasons about the task, decides which tools to use, calls them, processes the results, and decides what to do next. A human is present — watching, steering, approving. The agent proposes actions; the human accepts, rejects, or redirects.

```
LLM reasons → proposes action → human approves → LLM executes → LLM reasons → ...
```

The model has agency over the workflow but the human has authority over execution. This is the pattern of Claude Code, Cursor, Codex, and terminal-based coding agents. It's also the pattern of interactive assistants built on infrastructure kits that provide an agent loop, tool execution, and a UI for human interaction.

**Autonomous agents.** The LLM is the outer loop and there's no human watching. The agent receives a task specification, reasons about it, executes tools, evaluates its own progress, and terminates when it determines the task is complete (or when it hits a budget/timeout). Recovery from failure is automated — retry policies, fallback strategies, escalation paths that don't involve a human.

```
LLM receives spec → reasons → executes → self-evaluates → reasons → ... → terminates
```

The model has agency over both the workflow and the termination condition. This is the pattern of headless agent execution (Claude Agent SDK in `--print` mode), CI/CD-integrated agents, background processing agents, and any system where an agent runs unattended.

### The Termination Condition

This is the deepest architectural question and the one most often overlooked. Who decides when the task is done?

**In a programmatic framework**, the code decides. The pipeline has a fixed number of steps. When the last step completes, the task is done. There's no ambiguity — termination is structural.

**In an agentic assistant**, the human decides. The agent works until the human says "that's good" or "let's stop." The agent may suggest it's done, but the human has final authority. This is powerful for open-ended tasks but creates a bottleneck — the task can't complete without human attention.

**In an autonomous agent**, a specification decides. This might be a test suite that passes, a rubric that evaluates above a threshold, a checklist that's fully satisfied, or simply a timeout. The agent evaluates its own output against the specification and terminates when the criteria are met.

The specification is the hardest part of autonomous agent design. If the spec is too loose ("make this code better"), the agent may terminate prematurely or loop indefinitely. If the spec is too tight ("pass all 847 test cases"), the agent may spend resources on irrelevant edge cases. Writing a good termination spec is the autonomous equivalent of writing a good prompt — it encodes the task definition and the quality bar, and it's where most autonomous agent failures originate.

**The implications:**

| Aspect | Programmatic | Assistant | Autonomous |
|--------|-------------|-----------|------------|
| Termination defined by | Code structure | Human judgment | Specification/rubric |
| Termination is | Deterministic | Interactive | Evaluative |
| Runaway risk | None | Low (human can stop) | Real (needs budget/timeout) |
| Ambiguous tasks | Handled by code (badly) | Handled by human (well) | Handled by rubric (varies) |
| Completion guarantee | Strong (if code works) | Weak (depends on human) | Medium (depends on spec quality) |

### Session Model

**Programmatic frameworks** use stateless or explicitly managed state. Each model call is independent. State between calls is managed in application code — variables, databases, queues. If the process crashes, state is lost unless you've built persistence explicitly.

**Agentic assistants** use interactive sessions. The conversation context is the state. The agent remembers what happened earlier in the session because it's all in the context window. Sessions are inherently ephemeral — they exist while the conversation is active and are lost when it ends, unless the runtime provides session persistence (auto-compaction, session save/restore).

**Autonomous agents** need durable execution. The agent may run for minutes to hours. The process may crash and restart. The state must survive: what steps have been completed, what the intermediate results were, where to resume. This requires checkpointing, serializable state, and a task queue that can replay from the last checkpoint.

The session model determines your infrastructure requirements:

| Aspect | Programmatic | Assistant | Autonomous |
|--------|-------------|-----------|------------|
| State management | Application code | Context window | Durable checkpoints |
| Crash recovery | Restart from code | Restart conversation | Resume from checkpoint |
| Concurrency | Native (threads, queues) | One session at a time | Task queue with workers |
| Resource management | Code-controlled | Context window limits | Budget/timeout enforcement |

### Failure Recovery

**Programmatic frameworks**: the developer catches exceptions. Each model call has explicit error handling. A failed call can be retried, the input can be modified, or the pipeline can be routed to a fallback. The error handling is code the developer wrote, tested, and deployed.

**Agentic assistants**: the human intervenes. The agent encounters an error, reports it, and the human decides what to do — retry, change approach, provide additional information, or give up. The human is the universal fallback. This is robust for unpredictable failures but doesn't scale.

**Autonomous agents**: automated policies handle failures. Retry with backoff, fallback to a simpler approach, escalate to a different agent, or fail with a structured error report. The failure handling must be pre-designed because there's no human to improvise. This requires anticipating failure modes — a fundamentally different engineering challenge than handling them interactively.

## The Infrastructure Convergence

The three points on the spectrum are converging from both directions. Understanding the convergence helps you choose based on current needs without locking yourself into a dead end.

### What's converging: the capability layer

Skills, MCP servers, tool definitions, and retrieval patterns work across all three architectural categories. A well-designed MCP server that provides search tools over a compliance corpus is equally useful to:

- A programmatic pipeline that calls `search_standards(query)` at a predetermined step
- An agentic assistant that decides when to search based on the conversation
- An autonomous agent that searches as part of its self-directed workflow

The tool doesn't care who called it. The skill doesn't care whether a human approved the invocation. The retrieval mechanism doesn't care whether the query came from a code path or an agent's reasoning. This is the layer that's converging fastest — and it's why the existing documents in this collection (*Agent Tooling*, *MCP vs Scripts vs Hooks*, *Knowledge Architecture*) apply across the entire spectrum.

### What's not converging: the orchestration layer

The orchestration — who decides what happens next, when to use which tool, when to stop — remains fundamentally different across the spectrum. And this difference has deep implications:

**Testability.** A programmatic pipeline can be unit-tested: given this input, the pipeline produces this output. An agentic assistant's behavior depends on the conversation flow, which is non-deterministic. An autonomous agent's behavior depends on the task spec and the model's reasoning, both of which are variable. Moving right on the spectrum makes testing harder and shifts from unit tests to behavioral evaluations.

**Observability.** A programmatic pipeline logs structured events at each step. An agentic assistant logs a conversation transcript. An autonomous agent logs a potentially long, branching reasoning trace. The tools for understanding what happened and why it went wrong are different for each.

**Predictability.** A programmatic pipeline does the same thing every time (assuming deterministic code). An agentic assistant does approximately the same thing but with variation based on how the human interacts. An autonomous agent may take entirely different paths to the same goal on different runs. Predictability decreases as orchestration authority shifts to the model.

### The "agentic infrastructure kit" pattern

A middle ground has emerged: projects that provide a thin agent runtime (agent loop, message types, tool execution, session management) with pluggable capabilities (skills, extensions, MCP connections, LLM provider abstraction). The runtime is the orchestration layer. The capabilities are the shared layer. Specific agents (a coding assistant, a Slack bot, a batch processor) are thin layers on top of the runtime.

This pattern sits between "use a hosted agent service" and "build from the LLM API up." It gives you control over the full stack without requiring you to build the full stack. The tradeoff is maintenance — you own the runtime, which means you maintain the runtime.

This pattern serves all three points on the spectrum:

- Wire the runtime to a terminal UI with human-in-the-loop → agentic assistant
- Wire it to a task queue with automated termination → autonomous agent
- Wire it to a pipeline orchestrator that calls agent steps programmatically → programmatic framework with agent capabilities

The runtime doesn't determine your architecture. The wiring does. This is the convergence in practice: the same core infrastructure, different orchestration layers on top.

## Decision Heuristics

Start here. Go deeper into the sections below when you need more context.

| Question | Answer | Architecture |
|----------|--------|-------------|
| Is the workflow path known in advance? | Yes | **Programmatic** — code the pipeline |
| Is the workflow path known in advance? | No | **Agentic** — let the model reason |
| Is there a human available during execution? | Yes, and needed | **Assistant** — human steers and approves |
| Is there a human available during execution? | No | **Autonomous** — or **programmatic** if path is known |
| What's the acceptable failure mode? | Crash and fix code | **Programmatic** |
| What's the acceptable failure mode? | Ask the human | **Assistant** |
| What's the acceptable failure mode? | Retry/fallback automatically | **Autonomous** |
| What's the latency tolerance? | Milliseconds to low seconds | **Programmatic** — model as function call |
| What's the latency tolerance? | Interactive (seconds to minutes) | **Assistant** |
| What's the latency tolerance? | Background (minutes to hours) | **Autonomous** |
| How many concurrent executions? | Hundreds+ | **Programmatic** or **autonomous with task queue** |
| How many concurrent executions? | One at a time | **Assistant** is fine |
| Is output format more important than reasoning? | Yes | **Programmatic** with structured outputs |
| Is reasoning more important than output format? | Yes | **Agentic** (assistant or autonomous) |
| Can you write a termination spec? | Yes, precisely | **Autonomous** or **programmatic** |
| Can you write a termination spec? | No, it requires judgment | **Assistant** — human is the termination condition |
| What's the testability requirement? | Unit tests, CI/CD | **Programmatic** |
| What's the testability requirement? | Behavioral eval | **Agentic** |

When multiple rows point to different architectures, that's a signal your system sits between two points on the spectrum — or that different parts of the system belong at different points. A programmatic pipeline that delegates one open-ended step to an agentic assistant is a valid architecture.

## The Skills and Decentralized Stack Pattern

The agent skills model — modular, self-contained capabilities that agents can discover and use — is the composability primitive that works across the entire spectrum. Skills don't care about orchestration architecture. A skill that provides date resolution works in a programmatic pipeline, an interactive assistant, and an autonomous agent.

**Skills as portable capabilities:**

A well-designed skill (see *Agent Tooling: MCP vs Scripts vs Hooks*) bundles:
- A capability (tool definitions, scripts, or MCP server endpoints)
- Documentation (SKILL.md with invocation instructions and progressive disclosure)
- Fallback behavior (check for MCP availability, fall back to bundled script)

This packaging travels across orchestration architectures. The skill doesn't need to know whether a programmatic pipeline or an interactive agent is calling it. This portability is the core value proposition of the skills model over monolithic frameworks, where capabilities are tightly coupled to the framework's orchestration abstractions.

**MCP as the shared tool protocol:**

MCP provides typed tool interfaces that work regardless of who calls them. A programmatic framework calls MCP tools through its tool registration layer. An agentic assistant calls them through its tool-use capability. An autonomous agent calls them through its execution loop. The tool schemas, permission gates, and state management are the same in all cases.

This is the same pattern that won in web services: a thin protocol layer (HTTP/REST, then GraphQL, now MCP) that decouples capabilities from the systems that use them. The protocol survives across orchestration architecture changes.

**The tradeoff: composability vs. integration testing:**

The decentralized stack (thin orchestration core + pluggable skills + MCP tools + retrieval) is highly composable. You add capabilities by adding skills. You share capabilities by sharing MCP servers. You swap orchestration by rewiring the core.

The cost is integration testing complexity. When every capability is a separate module, testing the combination requires testing across boundaries. A monolithic framework where everything is integrated provides consistent behavior across capabilities but at the cost of being locked to that framework's abstractions and update cadence.

| Aspect | Decentralized (skills + MCP) | Monolithic framework |
|--------|------------------------------|---------------------|
| Adding capabilities | Add a skill or MCP server | Write a framework-specific integration |
| Sharing capabilities | Share the skill or point at the MCP server | Build framework-specific adapter |
| Swapping orchestration | Rewire the core | Rewrite within the framework or migrate |
| Integration testing | Cross-boundary, multi-component | Framework provides integrated testing |
| Debugging | Spans multiple independent components | Single system, integrated logging |
| Upgrade path | Independent per component | Framework-wide, potentially breaking |

**The practical reality:** Most production systems are hybrid. A thin orchestration core handles the agent loop. Skills and MCP servers provide capabilities. Some capabilities are framework-specific (native tools that leverage the runtime's UI or editor state). The architecture is neither fully decentralized nor fully monolithic. It's a pragmatic mix driven by what each component needs.

**Rule of thumb:** Use skills and MCP for capabilities that have value across multiple contexts (different agents, different workflows, different environments). Use framework-specific integrations for capabilities that are tightly coupled to a specific runtime's features. Don't abstract what doesn't need to be portable.

## Migrating Between Categories

Requirements change. An interactive assistant that works for one user may need to serve a team. A programmatic pipeline may encounter tasks too open-ended for code paths. Knowing what's portable across migrations and what isn't helps you make decisions that don't paint you into a corner.

### Programmatic → Assistant

**When to migrate:** The workflow has tasks that can't be predetermined. Edge cases multiply faster than you can code handlers. The model needs to reason about what to do, not just fill in blanks. Users want to interact with the system, not just consume its output.

**What's portable:** Tool definitions, MCP servers, retrieval patterns, validation logic. All of these plug into an agent runtime as easily as they plug into a pipeline.

**What's not portable:** The pipeline orchestration logic. In a programmatic framework, this is code — explicit control flow, error handling, routing. In an agentic assistant, this is the model's reasoning, guided by system prompts and tool availability. You're replacing code with cognition, and that requires different design patterns (system prompts instead of code paths, tool schemas instead of function signatures, rubric evaluation instead of assert statements).

### Assistant → Autonomous

**When to migrate:** The human is the bottleneck. Tasks queue up waiting for human approval. The system needs to run overnight, over weekends, or across time zones without human supervision. Scale requirements exceed what interactive sessions can handle.

**What's portable:** Everything from the assistant model, plus the skills and tools the agent already uses.

**What's not portable:** The human-as-fallback assumption. Every point in the assistant workflow where the agent asks the human — for clarification, for approval, for a judgment call — needs a programmatic replacement. Clarification becomes a heuristic or a retrieval step. Approval becomes a policy check. Judgment calls become rubric evaluation (see *Output Determinism*). This is the hardest part of the migration: identifying every point where the human was silently providing intelligence that the system now needs to provide itself.

Additionally, you need new infrastructure: durable execution (checkpointing, crash recovery), resource management (budgets, timeouts), termination specifications, and automated failure handling. None of these exist in the assistant model because the human managed them implicitly.

### Autonomous → Programmatic

**When to migrate:** The autonomous agent's behavior is too variable. You need reproducible, testable, auditable execution. Regulatory requirements demand deterministic processing. The task has been running long enough that the workflow path is now well-understood and can be coded.

**What's portable:** Tool integrations, retrieval patterns, validation logic, and the understanding of the workflow that you gained from observing the autonomous agent's behavior.

**What's not portable:** The agent's reasoning-based orchestration. You're replacing cognition with code, which means explicitly coding every decision point that the model was handling. This is feasible when you've observed enough executions to understand the decision space. It's premature when the workflow is still being explored.

### The migration principle

Capabilities (tools, skills, MCP, retrieval) migrate easily. Orchestration (who decides what to do next) does not. Design your capabilities to be orchestration-agnostic, and accept that changing orchestration architectures is a significant engineering effort, not a configuration change.

## Anti-Patterns

**Choosing architecture based on tooling fashion.** "Everyone is building agents" is not an architectural justification. If your workflow is a known, bounded pipeline — extract data from these documents, validate against this schema, write to this database — a programmatic framework gives you stronger guarantees at lower cost than an agentic system. The model is a tool, not an identity.

**Building autonomous agents when the termination condition requires human judgment.** If the only reliable way to know whether the task is done is for a human to look at it, the system is an assistant, not an autonomous agent. Removing the human without replacing their judgment with a specification doesn't create autonomy — it creates an uncontrolled process.

**Using programmatic frameworks for genuinely open-ended tasks.** If the workflow involves exploring a codebase, understanding a bug, reasoning about possible fixes, and choosing an approach — that's not a pipeline. Trying to code every possible path through an open-ended reasoning task produces a brittle, unmaintainable state machine that handles the common cases and fails unpredictably on everything else. Let the model reason.

**Removing the human without adding the infrastructure.** The most common failure pattern in the assistant-to-autonomous migration. The team takes an interactive assistant, removes the human-in-the-loop approval steps, and calls it autonomous. But they don't add durable execution, automated failure handling, termination specifications, or resource budgets. The result is a system that runs unsupervised with no mechanism for detecting when it's stuck, wrong, or out of resources. The human wasn't just approving — they were monitoring, evaluating, and course-correcting. All of those functions need automated replacements.

**Over-decentralizing capabilities.** Forty MCP servers, sixty skills, and a skills-discovery-agent that finds the right skill to use. If the agent needs a meta-agent to manage its capabilities, the system is too fragmented. Composability has diminishing returns. The overhead of tool discovery, schema parsing, and capability selection consumes context and reasoning capacity that should go to the actual task. Group related capabilities into coherent services. Not every function needs its own MCP server.

**Confusing the capability layer with the orchestration layer.** "We use MCP, therefore we have an agent architecture." MCP is a tool protocol. It works in programmatic pipelines, interactive assistants, and autonomous agents. Using MCP doesn't determine your orchestration architecture any more than using HTTP determines your application architecture. The architecture is defined by who controls the execution flow, not by what protocol the tools use.

**Premature autonomy.** Building an autonomous system before you understand the workflow. The assistant model is a discovery tool — by watching how a human interacts with the agent, you learn the decision points, the failure modes, the edge cases, and the quality criteria. That understanding is what you need to write the termination specification and failure policies for the autonomous version. Skip the discovery phase and you'll build an autonomous system that automates the wrong workflow.

## The Core Design Principle

The architectural spectrum is determined by orchestration authority: who decides what happens next, who decides when it's done, and who recovers when something fails. The capability layer — tools, skills, MCP, retrieval — is increasingly shared across all points on the spectrum. Build capabilities to be orchestration-agnostic. Choose your orchestration position based on the workflow's actual requirements: how known the path is, whether a human is available, what failure mode is acceptable, and whether you can write a termination specification.

The convergence is real. The same infrastructure increasingly serves all three architectural categories. But convergence in capabilities doesn't mean convergence in orchestration. Programmatic control, human-in-the-loop steering, and autonomous execution remain fundamentally different architectures with fundamentally different failure modes. Choosing between them is an architectural decision, not a framework selection.
