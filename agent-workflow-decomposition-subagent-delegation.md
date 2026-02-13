---
id: agent-workflow-decomposition-subagent-delegation
title: "Agent Workflow Decomposition: When to Delegate to Subagents"
type: reference
scope: architecture
domain:
  - agent-development
  - workflow-design
  - orchestration
stability: medium
intent: guide
tags:
  - subagents
  - delegation
  - workflow-decomposition
  - agent-orchestration
  - handoff-contracts
  - context-budget
  - reusability
  - decision-framework
  - quality-gates
source: conversation
confidence: medium
created: 2026-02-12
---

# Agent Workflow Decomposition: When to Delegate to Subagents

A subagent is a single round-trip delegation — one agent calls another like a tool call, gets a result back, and continues. No multi-turn follow-up, no persistent collaboration. It's function-call semantics applied to agents. The question this document addresses isn't *how* to delegate — that's a platform capability — but *when* to break a workflow into stages that delegate to subagents, and when to keep everything in one agent.

```
Monolithic Agent ──→ Staged Handoffs ──→ Full Delegation Chain
 simplest, highest context    clear boundaries     most resilient, most tokens
```

The spectrum is a cost curve. Moving right buys you resilience, reusability, and context isolation. It costs you tokens, orchestration complexity, and handoff overhead. Most workflows belong somewhere in the middle — the interesting decision is figuring out where.

## Decision Heuristics

Start here. Go deeper into the sections below when you need more context.

| Question | Answer | Approach |
|----------|--------|----------|
| Does the entire workflow fit comfortably in one context window? | Yes | **Monolithic** — keep it together |
| Are there distinct phases with clear input/output contracts? | Yes | **Staged handoffs** |
| Do individual stages have reuse value outside this workflow? | Yes | **Staged handoffs** or **delegation chain** |
| Does the user need to enter the workflow at different starting points? | Yes | **Staged handoffs** |
| Is there a quality gate or review step? | Yes | **Separate review agent** |
| Does a stage require a different persona or instruction set? | Yes | **Separate agent** |
| Is the workflow conversational (multi-turn with the user)? | Yes | **Keep that part monolithic** |
| Does failure in one stage need isolated recovery? | Yes | **Staged handoffs** |
| Is there a stage that touches external systems with failure risk? | Yes | **Separate agent** — isolate the blast radius |
| Is the total workflow token budget a concern? | Yes | **Staged handoffs** — each stage gets a fresh window |

The table is ordered by diagnostic value — the first few questions eliminate the most options. When multiple rows point to different approaches, that's a signal the workflow has natural seams worth splitting on.

## Monolithic Agent

One agent, one context window, one continuous interaction. The agent handles every phase of the workflow — gathering information, processing it, validating it, and acting on it.

**Use a monolithic agent when:**

- The workflow is short enough that context accumulation isn't a concern
- There are no distinct phases — it's one continuous reasoning task from start to finish
- Every step depends heavily on the full conversational history and there's no clean way to summarize what came before
- The workflow is inherently interactive and the user expects a single coherent conversation, not a hand-off between agents
- No individual stage has reuse value outside this specific workflow
- You don't need different instruction sets, personas, or tool configurations at different points in the workflow

**Real example — simple bug filing:** A user describes a bug, the agent asks a few clarifying questions, formats it, and files it. Three minutes, 2K tokens of conversation, one API call at the end. Decomposing this into capture/review/publish agents would triple the token cost and add orchestration complexity for zero benefit.

**Tradeoffs:**

- Zero handoff overhead — no serialization, no context re-establishment, no orchestration logic
- Full conversational continuity — every earlier exchange is available for reference
- Simplest to build, debug, and reason about
- Context window is a hard ceiling — as conversation grows, the agent's working memory for instructions, reference material, and reasoning gets squeezed
- No reusability — the logic is embedded in one agent's prompt and can't be invoked independently
- Late-stage failures are expensive — if the agent crashes or the API call fails after 20 minutes of conversation, the user may need to start over
- Persona bleed — an agent instructed to be collaborative and helpful during drafting may struggle to switch to being critical and thorough during review. Instructions compete for attention in a long prompt

**Rule of thumb:** If the workflow is a single cognitive mode from start to finish and fits comfortably in context, keep it monolithic. The moment you're cramming instructions for fundamentally different tasks into one prompt and hoping the agent context-switches cleanly, you've outgrown this approach.

## Staged Handoffs

The workflow is decomposed into discrete stages, each handled by a separate agent invocation. One agent's output becomes the next agent's input. The orchestrator — which may be the user, a parent agent, or a simple pipeline — manages the transitions.

**The bug-filing example, decomposed:**

- **Stage 1 — Capture agent.** Conversational. Works with the user to understand the bug, asks clarifying questions, gathers reproduction steps, severity assessment, affected systems. Output: a structured draft document.
- **Stage 2 — Review agent.** Quality gate. Loads a completeness checklist and evaluates the draft against it. Checks for missing fields, ambiguous language, insufficient reproduction steps, missing severity justification. Output: approval, or a list of issues that need resolution.
- **Stage 3 — Publish agent.** Mechanical. Takes the approved draft and files it in the bug tracker via MCP tools or API calls. Handles field mapping, attachment uploads, and confirmation. Output: the filed bug ID and URL.

Each stage has a different cognitive mode: collaborative drafting, critical evaluation, and mechanical execution. Each benefits from a clean context loaded with stage-specific instructions rather than a shared prompt trying to do all three.

**Use staged handoffs when:**

- The workflow has natural phase boundaries where the cognitive task changes
- Individual stages have independent value — someone might want to use the review agent on a draft they wrote manually, or the publish agent on a bug from a different source
- The total workflow would strain a single context window, but each stage fits comfortably
- Different stages need different instruction sets, personas, or tool configurations
- A quality gate exists — validation and review are architecturally cleaner as separate agents
- Failure isolation matters — you don't want a publishing API error to blow away 20 minutes of drafting conversation

**Tradeoffs:**

- Handoff overhead — every stage transition requires serializing the output, passing it to the next agent, and re-establishing enough context for the downstream agent to function. This is real token cost
- Orchestration complexity — something has to manage the pipeline: routing between stages, handling review rejections, retrying failed stages. That logic exists somewhere, and it's code you maintain
- Information loss at boundaries — whatever doesn't make it into the handoff artifact is gone. If the capture agent and user had a nuanced side-conversation about edge cases that isn't reflected in the draft, the review agent doesn't know about it
- More total tokens than monolithic — each stage loads its own system prompt, instructions, and context. The combined token cost is higher even though each individual stage is cheaper
- Debugging spans multiple agent invocations — tracing a failure through a three-stage pipeline is harder than looking at one conversation

**Rule of thumb:** If the workflow has distinct phases with different cognitive modes and there's a natural artifact that passes between them, those phases are natural agent boundaries.

## The Handoff Contract

The handoff contract is the single most important design decision in a decomposed workflow. It defines what information passes between stages — and equally important, what doesn't.

**What a good handoff looks like:**

A structured artifact with a defined schema. For the bug-filing example, the capture agent produces a document with specific fields: title, description, reproduction steps, severity, affected systems, environment details, any relevant logs or screenshots referenced. The review agent knows exactly what to expect and can evaluate completeness mechanically. The publish agent knows exactly what to map to the bug tracker's fields.

```markdown
# Handoff Artifact: Security Bug Draft
## Required Fields
- title: string
- description: string (markdown)
- reproduction_steps: list[string]
- severity: critical | high | medium | low
- affected_systems: list[string]
- environment: string
- evidence: list[{type, reference}]
## Optional Fields
- suggested_fix: string
- related_bugs: list[string]
- reporter_notes: string
```

**What a bad handoff looks like:**

Dumping the entire conversation history from the capture agent into the review agent's context. This recreates the monolithic context problem — the review agent is now wading through 15 turns of back-and-forth to find the actual draft, and its review instructions are competing with all that conversational noise for attention. You decomposed the workflow structurally but not cognitively.

**Design principles for handoff contracts:**

- **Structured over narrative.** A JSON object or a markdown document with defined sections beats "here's what we discussed." The downstream agent shouldn't have to interpret — it should be able to parse.
- **Complete for the downstream stage's needs.** The review agent shouldn't need to ask follow-up questions that only the user could answer. If it does, your handoff contract is missing fields.
- **Minimal beyond that.** Don't pass the capture agent's internal reasoning, tool-call history, or conversational scaffolding. The downstream agent doesn't need it and it dilutes the signal.
- **Versioned when the workflow evolves.** If you add a field to the handoff contract, downstream agents need to handle its presence or absence gracefully. This is the same contract versioning discipline as API design.

**The compression question:** How much context compression is acceptable? The answer depends on whether the downstream agent needs to *understand why* or just *act on what*. A review agent evaluating draft quality may benefit from a brief "context" field summarizing what the user reported and what ambiguities were resolved. A publish agent filing a bug doesn't need any of that — just the structured data. Match the handoff richness to the downstream agent's actual needs.

**Rule of thumb:** If you can't describe the handoff as a schema with named fields and types, the boundary isn't clean enough to decompose on.

## Context Budget

Context windows are finite. Every token spent on conversation history, instructions, reference material, or tool definitions is a token unavailable for reasoning. Decomposition is one of the most effective tools for managing this budget.

**The monolithic budget problem:**

A monolithic bug-filing agent might need to carry:
- System prompt with persona and behavioral instructions (~500–1K tokens)
- Security bug template and field requirements (~500 tokens)
- Completeness checklist for review (~500 tokens)
- Bug tracker API documentation or MCP tool schemas (~500–1K tokens)
- The entire conversation history with the user (variable, potentially 5K+ tokens)
- The draft itself (~1K tokens)

That's 8K+ tokens before the model starts reasoning about the current turn. In a long conversation, the history alone can push past 10K. The instructions for later stages — review criteria, publishing procedures — sit in the prompt from turn one, consuming budget and attention even though they won't be needed for 15 turns.

**How decomposition helps:**

Each stage loads only what it needs:
- The capture agent carries the persona, template, and conversation history. No review checklist, no API docs.
- The review agent carries the checklist, the draft, and a brief context summary. No conversation history, no API docs.
- The publish agent carries the API docs, the finalized draft, and the tool schemas. No checklist, no conversation history.

Each stage's context is purpose-built. The instructions don't compete with each other. The model's full attention budget goes to the current task.

**The crossover point:**

Decomposition has a fixed overhead — each stage's system prompt, handoff parsing, and context establishment. For short workflows, this overhead exceeds the savings. For long workflows, the savings dwarf the overhead. The crossover is workflow-dependent, but a rough heuristic: if the monolithic agent's context would exceed 60–70% of the working window by the time it reaches the final stage, decomposition probably saves net tokens by letting each stage work with a clean, focused context.

**Rule of thumb:** If you're loading instructions at the start of the conversation that won't be used until the end, those instructions are candidates for a separate stage.

## Reusability and Entry Points

Decomposed stages are independently invocable. This is the highest-leverage benefit of decomposition and the one most often overlooked when weighing the costs.

**Multiple entry points from the bug-filing example:**

- **Full workflow (capture → review → publish):** the user starts from scratch, the capture agent collects information, the review agent validates, the publish agent files.
- **Review + publish:** the user already has a draft — maybe they wrote it manually, maybe another tool generated it, maybe they're revising a previously rejected draft. They skip capture entirely and start at review.
- **Publish only:** the user has a reviewed, finalized draft from some other process. They just need it filed. They skip capture and review.
- **Review only:** the user has a draft and wants quality feedback but isn't ready to publish. They run the review agent in isolation.

A monolithic agent has one entry point: the beginning. If a user already has a draft, they either feed it into the monolithic agent and hope it recognizes that capture is unnecessary, or they fight the agent's conversational flow to skip ahead. Neither is clean.

**Reusability across workflows:**

The review agent isn't bug-specific if designed correctly. A review agent that evaluates structured documents against a completeness checklist can serve:
- Security bug drafts
- Incident reports
- Change requests
- Design proposals

The review logic is the same: load a checklist, evaluate the draft against it, report gaps. The checklist changes. The agent doesn't. Similarly, a publish agent that maps structured documents to a bug tracker's API can serve any source that produces drafts in the expected schema — not just your capture agent.

**When reusability isn't worth it:**

If the stages are so tightly coupled to one workflow that they'd never be invoked independently, the reusability argument doesn't hold. A "review agent" whose checklist is hardcoded to one specific bug template and can't be parameterized isn't reusable — it's just a decomposed monolith with extra overhead. Reusability requires that stages be designed with generic interfaces, not just split out mechanically.

**Rule of thumb:** If you can describe a stage's function without referencing the parent workflow — "reviews a structured document against a configurable checklist" rather than "reviews the output of the bug capture agent" — that stage has reuse value worth the decomposition cost.

## Error Handling and Resilience

Decomposed workflows fail in smaller, more recoverable units. This matters most when workflows are long, involve external systems, or produce artifacts the user invested time creating.

**Isolated failure:**

If the publish agent fails — the bug tracker API is down, authentication expired, a field mapping is wrong — the draft is still intact. The user doesn't lose the capture conversation. Retry is scoped: fix the issue, re-invoke the publish agent with the same draft. No need to re-gather information or re-validate.

In a monolithic agent, a late-stage failure can be catastrophic to the session. The API error happens at turn 25 of a conversation. The agent tries to recover, but its context is now polluted with error messages, retry attempts, and confusion. The user's only reliable option may be starting over.

**Retry semantics:**

Staged handoffs give you clean retry boundaries. Each stage is idempotent with respect to its input: given the same handoff artifact, the stage produces the same output (or fails cleanly). This means:
- Retrying a failed stage doesn't require replaying the entire workflow
- A failed stage can be retried with modified configuration (different credentials, different endpoint) without affecting prior stages
- The handoff artifact serves as a checkpoint — the workflow can resume from any stage

**Failure routing:**

When the review agent rejects a draft, what happens? Two patterns:

- **Return to orchestrator.** The review agent returns its findings. The orchestrator (or the user) decides whether to send the draft back to the capture agent for revision, edit it manually, or override the rejection. The orchestrator owns the routing decision.
- **Direct kickback.** The review agent returns its findings to the capture agent, which revises the draft and resubmits. This creates a loop between two stages, which is architecturally fine but requires the capture agent to handle both "initial capture" and "revision" modes — or a separate revision agent that takes a draft plus review feedback and produces an updated draft.

The first pattern is simpler and more flexible. The second is more autonomous but harder to debug when the loop doesn't converge.

**Rule of thumb:** If a stage touches an external system that can fail, it should be a separate agent. The user's investment in prior stages shouldn't be at risk from downstream infrastructure problems.

## The Review/Quality Gate Pattern

A review agent that validates output before it moves forward is common enough and valuable enough to warrant its own discussion.

**Why review is better as a separate agent:**

- **Different cognitive mode.** The capture agent's job is to be collaborative — help the user articulate the bug, fill in gaps, suggest improvements. The review agent's job is to be critical — find what's missing, flag what's ambiguous, reject what's incomplete. These are opposing instruction sets. Putting both in one agent creates internal tension. The agent either goes easy on review because it's been trained to be helpful, or it becomes adversarial during capture because it's been told to be critical. Splitting them eliminates the tension.
- **Different context needs.** The review agent needs the quality checklist, the completeness criteria, and the draft. It does not need 15 turns of conversation about how the user discovered the bug. Loading only what the review agent needs improves its focus and reduces the chance of checklist items being overlooked.
- **Reusability.** A review agent parameterized by checklist can serve any workflow that produces structured documents. Bug drafts, incident reports, design docs — same validation logic, different criteria.
- **Auditability.** When the review agent is a separate invocation, its output is a discrete artifact: "reviewed draft X, found issues Y and Z" or "reviewed draft X, approved." This is auditable in a way that "the agent reviewed its own work somewhere around turn 18" isn't.

**Designing the review agent's interface:**

- **Input:** the draft artifact (matching the handoff schema) plus the review criteria (a checklist, a rubric, or a set of rules)
- **Output:** a structured review result — approved or rejected, with specific findings. Each finding references the relevant field or section and explains the issue clearly enough that either the user or the capture agent can address it
- **No side effects.** The review agent does not modify the draft. It evaluates and reports. Modification is the capture agent's job (or the user's). Keeping review read-only makes the pipeline easier to reason about

**When review doesn't need its own agent:**

If the "review" is a simple format check — are all required fields present, is the severity one of the allowed values — that's validation logic, not a cognitive task. Encode it in the handoff contract's schema or run it as a script. You don't need an LLM to check whether a field is null.

**Rule of thumb:** If the review requires judgment — evaluating clarity, assessing completeness, checking for logical consistency — it's a cognitive task that benefits from a dedicated agent. If it's mechanical validation, it's a schema or a script.

## Cost and Complexity Tradeoffs

Decomposition is not free. Every stage boundary costs tokens and introduces a failure point.

**Token overhead per stage:**

- System prompt: 500–2K tokens per agent, loaded fresh each invocation
- Handoff artifact parsing: the downstream agent reads and processes the artifact, which may be 500–2K tokens
- Context re-establishment: any background information the downstream agent needs that wasn't in the handoff
- Response generation: the downstream agent produces its output, which may need to be more explicit than in a monolithic flow because it can't reference "what we discussed earlier"

For a three-stage pipeline, this might add 3–6K tokens of overhead compared to a monolithic agent that handles the same workflow. For a simple workflow that's 5K tokens total, that's a 60–120% increase. For a complex workflow that's 30K tokens total, it's a 10–20% increase and likely a net savings because each stage's context is more focused.

**Orchestration complexity:**

Something has to manage the pipeline. In the simplest case, the user is the orchestrator — they invoke each stage manually and pass artifacts between them. This is low-tech but gives the user full control and visibility.

In an automated case, a parent agent or script manages routing: invoke capture, take the output, invoke review, check the result, handle rejection loops, invoke publish. This orchestration logic needs to handle:
- Happy path routing (capture → review → publish)
- Rejection loops (review → back to capture or user)
- Stage failures (retry, abort, skip)
- User intervention points (where the user can inspect, modify, or override)

This is real code with real edge cases. The orchestration itself can have bugs. Test it like you'd test any pipeline.

**Debugging across stages:**

When something goes wrong in a monolithic agent, you look at one conversation. When something goes wrong in a staged pipeline, you need to trace through multiple agent invocations: what did the capture agent produce? What did the review agent flag? Was the handoff artifact well-formed? Did the publish agent receive what it expected?

Good observability practices mitigate this — log the handoff artifact at each boundary, tag invocations with a workflow ID — but it's inherently more work than reading one conversation transcript.

**When the overhead isn't worth it:**

- The workflow completes in under 5 minutes of interaction and under 5K tokens
- There's no reuse case for any individual stage
- The workflow has no external system calls that could fail
- There's no meaningful quality gate — the output is either trivially correct or not worth validating
- The team maintaining the workflow doesn't have the infrastructure to manage multi-stage pipelines

In these cases, monolithic is the right call. Don't decompose because it's architecturally fashionable. Decompose because the workflow's characteristics demand it.

**Rule of thumb:** Decomposition pays for itself when the workflow is long enough, complex enough, or risky enough that the benefits of isolation, reusability, and focused context exceed the fixed overhead of staging. For most workflows, the crossover is around 10K tokens or three distinct cognitive phases — whichever comes first.

## Anti-Patterns

**Decomposing for the sake of decomposition.** A three-stage pipeline for a workflow that takes two minutes and fits easily in context is over-engineering. The overhead costs more than the benefits. Decompose when the workflow's characteristics demand it, not because microservices taught you that smaller is always better.

**Passing raw conversation history as the handoff.** If the capture agent dumps its entire conversation with the user into the review agent's context, you haven't actually decomposed — you've just added a stage boundary with none of the context isolation benefits. The review agent is now parsing conversational noise to find the draft, and its review criteria are competing for attention with 15 turns of "can you tell me more about the reproduction steps?" The handoff should be a structured artifact, not a transcript.

**Making stages too granular.** Not every logical step in a workflow needs its own agent. "Parse the user's first message" is not a stage. "Validate the severity field" is not a stage. If a stage's entire job can be described in one sentence and takes under 500 tokens to execute, it's a function call, not an agent. Agents are for tasks that require reasoning, judgment, or multi-step interaction.

**No input/output contracts between stages.** If the capture agent's output format is "whatever it feels like producing" and the review agent has to guess what to expect, every handoff is a gamble. Undefined contracts between stages are the equivalent of untyped function signatures — they work until they don't, and when they don't, the failure is confusing and hard to trace. Define the schema. Validate against it.

**Assuming decomposition fixes a context problem that's actually a prompt problem.** If the monolithic agent produces bad results because the instructions are poorly written, decomposing into three agents with three sets of poorly written instructions produces three bad results instead of one. Decomposition improves context isolation and focus. It doesn't fix bad prompts.

**Circular delegation without convergence criteria.** A capture agent that sends drafts to a review agent that kicks them back to the capture agent that sends them back to review — without a maximum iteration count or escalation path — can loop indefinitely. Define convergence: after N rejections, escalate to the user. Unbounded loops are unbounded costs.

## Agent Teams — Explicit Non-Scope

This document covers single round-trip delegation and linear stage pipelines: agent A calls agent B, gets a result, continues. That's the model most workflows need and the model that existing platforms (Cursor subagents, Claude tool use, framework-based delegation) support well.

Persistent multi-agent teams — agents with shared memory, ongoing bidirectional communication, role negotiation, and collaborative reasoning — are a fundamentally different architecture with different tradeoffs. They're relevant for open-ended research tasks, adversarial red-team/blue-team setups, and complex planning problems where agents need to argue and converge. They're not relevant for the structured, phased workflows this document addresses.

If your workflow requires agents to *debate* rather than *hand off*, you need team architecture, not staged delegation. That's its own topic.

## The Core Design Principle

Decompose on cognitive boundaries, not arbitrary ones. When a workflow shifts from one mode of thinking to another — collaborative to critical, creative to mechanical, interactive to autonomous — that shift is a natural agent boundary. Each agent gets a focused prompt, a clean context, and a clear mandate. The handoff contract between them is a structured artifact that carries the work product forward without the baggage of how it got there.

When the workflow is one continuous cognitive task, decomposition adds overhead without adding value. Keep it together.

The right decomposition makes each stage independently testable, independently invocable, and independently improvable. The wrong decomposition creates a distributed monolith — all the complexity of multiple agents with none of the benefits of isolation.
