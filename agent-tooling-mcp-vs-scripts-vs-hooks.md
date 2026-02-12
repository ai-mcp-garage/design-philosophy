---
id: agent-tooling-mcp-vs-scripts-vs-hooks
title: "Agent Tooling: Choosing the Right Mechanism"
type: reference
scope: tooling
domain:
  - agent-development
  - architecture
stability: medium
intent: guide
tags:
  - skills
  - agent-scripts
  - code-execution
  - mcp
  - hooks
  - native-tools
  - agent-development
  - architecture
  - decision-framework
  - security-boundaries
  - consent-enforcement
source: conversation
confidence: medium
created: 2026-02-08
---

# Agent Tooling: Choosing the Right Mechanism

Four mechanisms exist for giving agents access to functionality: native/first-party tools, hooks, MCP servers, and agent-executed scripts. Each sits at a different point on the capability-portability spectrum. Picking the wrong one creates friction. Picking the right one makes the tooling disappear into the workflow.

```
Native Tools (platform) ──→ Hooks (automatic) ──→ MCP (structured) ──→ Scripts (flexible)
 most capable, least portable    fire-and-forget     deliberate calls      agent code execution
```

## Know Your Runtime

Before deciding what to build, assess what your runtime already provides — and what it doesn't.

- **Full IDE client** (Cursor, Windsurf, Claude Code, Codex): rich first-party toolset — file I/O, search, UI elements, agent orchestration, linting. Use what the platform gives you. It will almost always be faster, more reliable, and better integrated than anything you build externally.
- **Framework-based agent** (PydanticAI, LangGraph, CrewAI, etc.): you get the framework's abstractions (tool registration, routing, maybe memory) but core operations like file I/O, search, and UI are yours to wire up.
- **Raw API agent** (direct model API calls): you get nothing. Every tool is something you build or connect yourself.

This spectrum matters for two reasons.

**First, native tools handle things MCP and scripts can't.** UI rendering, editor state, sub-agent orchestration, structured user interaction — these are platform-level capabilities. No amount of MCP wizardry gives you a "spawn a sub-agent" tool or a structured question dialog. If your runtime provides these, use them.

**Second, native tools handle things MCP handles *poorly*.** File I/O through MCP requires escaping JSON strings, chunking large content, and dealing with encoding overhead. A native Read/Write tool bypasses all of that. When you're building from scratch and your runtime *doesn't* give you these primitives, you may need to build first-party tools for operations where MCP's serialization layer fights you. MCP is excellent for portability and structured interfaces, but it's not the right abstraction for every core primitive.

The rest of this note covers the three mechanisms you *build*: hooks, MCP servers, and agent-executed scripts. Native tools are the environment you work within — assess them first, then decide what custom tooling you need on top.

## Decision Heuristics

Start here. Go deeper into the sections below when you need more context.

| Question | Answer | Mechanism |
|----------|--------|-----------|
| Does your runtime already handle this well? | Yes | **Native tool** — use it |
| Does it need UI, editor state, or agent orchestration? | Yes | **Native tool** (build one if your runtime lacks it) |
| Does the agent need to *decide* to use it? | No | **Hook** |
| Does it need typed schemas and shared state? | Yes | **MCP** |
| Is it shared across multiple skills or agents? | Yes | **MCP** |
| Do you need permission gating / elicitation? | Yes | **MCP** |
| Is it simple, stateless, and self-contained? | Yes | **Script** |
| Does the agent need to improvise with it? | Yes | **Script** |
| Do you want progressive disclosure? | Yes | **Script** |
| Must it work without external config? | Yes | **Script** |
| Will it be shared across workspaces or users? | Yes | **Script** (or MCP with script fallback) |

The table is ordered intentionally — check native tools first, then hooks, then MCP, then scripts. When multiple rows point to different mechanisms, that's a signal you may want hybrid layering.

## When to Use Hooks

Hooks are event-driven triggers bound to lifecycle moments — session start, pre/post tool use, etc. The agent has zero autonomy here: it doesn't decide to call a hook, the hook just fires. That's the point.

Not all runtimes support hooks. If yours doesn't, this section doesn't apply — the functionality hooks would handle needs to be pushed into your system prompt, startup scripts, or tool preambles.

**Use hooks when:**

- Context should be injected automatically without the agent needing to "decide" to fetch it
- Side effects should happen every time, unconditionally (environment setup, telemetry, loading baseline context)
- The operation is fire-and-forget — no return value needed in the agent's tool-call flow

**Real example:** `inject-date.sh` runs at session start and injects the current date, day of week, week number, and quarter into the agent's context. The agent never asks "what day is it?" — it just knows.

**Good candidates:** date/time injection, environment variable setup, session telemetry, auto-loading context files, pre-flight checks.

**Tradeoffs:**

- Cannot be called on demand — they only fire at their bound lifecycle event
- Don't compose — you can't chain hooks or pass output from one to another
- Failure is silent — if a hook fails, the agent doesn't know it's missing context. There's no error to react to, just an absence. This makes hooks the most dangerous mechanism to depend on for critical information.
- Not tools — they inject context or cause side effects, they don't return structured results
- Runtime-dependent — not all agent frameworks support lifecycle hooks

**Rule of thumb:** If the agent should never have to *think* about whether to use something, it's a hook.

## When to Use MCP Servers

MCP servers are persistent processes that expose typed tools via a structured protocol. The agent sees every available tool upfront with full schemas — parameter names, types, descriptions — and calls them deliberately through conventional tool-call syntax.

**Use MCP when:**

- The same tool is needed across multiple skills and you don't want to duplicate the logic
- You need elicitation or permission gating — forcing the agent to ask before performing certain actions
- Operations need to maintain state between calls (sessions, queues, caches, connection pools)
- The setup is convoluted or requires hosted dependencies that would be impractical to bundle inside a skill
- Type safety matters — the agent needs to know every parameter's data type and shouldn't have to guess invocation syntax
- The agent needs full upfront discoverability of all available operations
- A persistent process avoids cold-start overhead on every call
- Multiple agents or environments need shared access to the same tools — MCP servers are persistent processes that multiple clients can connect to simultaneously. If two agents need to coordinate through shared state, MCP is the only mechanism here that naturally supports that. Scripts are stateless and scoped to the invoking agent. Hooks are scoped to the session that triggered them.

**Real example:** `date-resolver/server.py` exposes `resolve_date`, `get_current_datetime`, `get_day_of_week`, and `show_calendar` as typed MCP tools. The agent calls them with structured arguments and gets structured results. No ambiguity about invocation.

**Tradeoffs:**

- Requires a running server process — something to start, monitor, and restart if it crashes
- A server crash takes out *all* tools it exposes, not just one. The agent gets explicit errors, but can't recover without a fallback.
- Harder to debug than scripts — requires server logs, not just "run the command and see what happens"
- Setup friction for sharing — recipients need to install dependencies, configure the server, and wire it up at the right scope
- Adds infrastructure overhead for simple operations that don't need persistence or state
- Serialization overhead — MCP's JSON protocol layer makes it a poor fit for high-throughput or binary-heavy operations like raw file I/O

**Rule of thumb:** If multiple consumers need the same structured, stateful tools — MCP. If you're building from scratch without a client, MCP is your structured foundation for everything that doesn't fight the protocol.

## When to Use Agent-Executed Scripts

Agent-executed scripts are code the agent runs via shell. The core mechanism is simple: the agent has access to code execution and uses it. What varies is how the scripts are organized and delivered:

- **Skill-bundled scripts**: files shipped in a skill's `/scripts` directory, with invocation instructions in the SKILL.md. The most structured form — portable, version-controlled, and self-documenting.
- **Standalone project scripts**: scripts sitting in a workspace or project directory that the agent is pointed at or discovers. Less structured, but no skill packaging overhead.
- **Ad hoc execution**: the agent writes and runs code on the fly. Maximum flexibility, zero durability — the code exists only for that invocation unless the agent saves it.

All three share the same fundamental tradeoffs. They're stateless, self-contained, and give the agent full freedom to improvise — piping output, chaining commands, modifying arguments, even forking or rewriting the script if needed.

**Use scripts when:**

- The operation is simple, repeatable, and essentially a one-shot command or straightforward logic
- You can containerize the functionality into a few scripts and modules without needing shared state
- The agent needs to write code that interacts with the logic — batch operations, piping output into other commands, manipulating results programmatically in ways that would be inefficient through MCP tool calls
- You have many disparate features that benefit from progressive disclosure — the SKILL.md reveals them contextually instead of dumping every capability into the tool list upfront
- Self-containment matters — just files, no daemon, no server, no external configuration
- Portability matters — scripts travel with the skill or project, zero setup for recipients
- The agent may need to modify or fork the script for edge cases
- Offline operation matters — no server process to depend on

**Real example:** `date-resolver/scripts/dates.py` does the same date resolution as the MCP server, but with zero dependencies beyond Python stdlib. It runs anywhere Python 3.10+ exists, ships with the skill, and needs no setup.

**Tradeoffs:**

- No state between calls — every invocation starts fresh
- The agent may misinterpret how to invoke a script if instructions aren't explicit about execution vs. reading as reference
- Less type safety — the agent constructs shell commands rather than calling typed tool schemas
- Output parsing depends on the agent interpreting stdout correctly
- Failure is isolated — a broken script only kills that one operation. The agent gets an error and can adapt. This is the safest failure mode of any mechanism.

**Rule of thumb:** If it needs to be portable, self-contained, and the agent might need to get creative with it — script.

## Security Boundaries: Hard vs. Soft Enforcement

A critical distinction between MCP servers and agent-executed scripts is *where consent enforcement lives* — and whether the agent can subvert it.

### MCP Servers — Hard Enforcement

When an MCP server gates an operation behind a consent prompt, that check runs inside the server process. The LLM cannot modify it. The agent makes a request; the server decides whether to honor it, including whether to prompt the user first. Even if the agent is jailbroken, receives adversarial instructions, or simply makes a mistake, it *physically cannot bypass* the consent check. The security boundary exists outside the agent's control surface.

### Scripts — Soft Enforcement

If consent logic lives in a script the agent executes — an `input()` call, a CLI confirmation prompt, a "are you sure?" dialog — the agent is the one writing and invoking that code. Nothing stops it from:

- Writing code that never calls the prompt in the first place
- Modifying an existing script to remove or short-circuit the check
- Piping `yes` into a confirmation prompt
- Rewriting the logic entirely to skip the gate

You can put "always prompt for user consent before destructive operations" in your system prompt or skill instructions, and a well-behaved agent will follow it. But that's an *instruction*, not *enforcement*. Prompt injection, jailbreaks, model errors, or even just the agent optimizing for efficiency can bypass it. You're relying on compliance, not architecture.

### The Practical Line

For trusted automation where the consequences of skipping consent are low — renaming files, reformatting code, running tests — script-based prompts are fine. The risk/reward doesn't justify MCP infrastructure.

For operations where you *need* guaranteed consent — deleting data, financial transactions, publishing content, credential access, anything irreversible or high-stakes — MCP servers are architecturally superior. The enforcement is structural, not behavioral. This isn't paranoia; it's recognizing that the agent is an untrusted execution context for security-critical decisions.

## Hybrid Layering

Sometimes the right answer isn't one mechanism — it's multiple mechanisms covering different needs. Layering is justified when each mechanism addresses a distinct *timing* (automatic vs. on-demand) or *audience* (local workspace vs. portable skill).

**Case study: date-resolver**

This exists as all three buildable mechanisms simultaneously, and each serves a purpose:

- **Hook** (`inject-date.sh`): fires at session start, injects today's date into context. Free, automatic, always there. The agent never has to think about it.
- **MCP server** (`date-resolver/server.py`): on-demand date math with typed tool calls. Structured, reliable, discoverable. For when the agent needs to *compute* something.
- **Skill script** (`date-resolver/scripts/dates.py`): same logic, zero dependencies, portable. Works in any environment where the MCP server isn't configured.

**When layering makes sense:**

- Each layer covers a distinct timing or audience
- The hook handles the baseline (what the agent should always know)
- The MCP server handles structured on-demand operations (where type safety and state matter)
- The script provides a portable fallback (where infrastructure can't be assumed)

**When layering is redundant:** If you only ever use the skill in one workspace where the MCP server is always running, the script fallback adds maintenance burden for no gain. Be honest about your actual usage patterns.

## Coupling, Scope, and Portability

### Self-Containment

Agent-executed scripts are fully self-contained — no external dependency, no configuration contract. A skill with bundled scripts works the moment it's installed. A skill that depends on an MCP server creates an implicit contract: *this MCP server must be configured and running for me to function.* That contract may or may not be obvious to someone installing the skill.

### MCP Configuration Scopes

MCP servers can be configured at different scope levels, and this directly affects which skills can depend on them:

- **Global MCP** (user-level): available across all workspaces for that user. Sharing the skill with others still requires they configure the same server.
- **Project-level MCP**: only available in that specific workspace. A skill depending on it breaks if used elsewhere.
- **Sub-agent MCP**: most restricted scope — only certain agent invocations within certain sub-agent configurations see those tools.

A skill that depends on a project-level MCP server is effectively locked to that workspace. A skill that depends on a global MCP server is locked to users who've configured it. Only scripts have no lock-in.

### Portability Ranking

1. **Scripts**: zero friction. Files travel with the skill or project. No configuration needed.
2. **Hooks**: scripts are portable, but require lifecycle configuration at the appropriate scope.
3. **MCP**: high friction. Requires dependency installation, server configuration, and correct scope-level wiring.
4. **Native tools**: zero portability across platforms. They exist or they don't.

### Defensive Design

A skill can check for MCP tool availability and fall back to a bundled script when the server isn't present. This gives you structured MCP calls when the infrastructure exists, and self-contained operation when it doesn't. If you're building skills that need to work across environments, this is the pattern to follow.
