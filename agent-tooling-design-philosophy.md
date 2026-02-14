# Why Agents Should Use Purpose-Built Libraries, Not CLI Applications

## The Question

When building agentic workflows that need to interact with external systems (cloud providers, CRMs, databases, etc.), should the agent shell out to an existing CLI application, or should we build a dedicated library that abstracts those interactions into high-level tool calls?

The answer, in nearly every case, is the library.

This document lays out the reasoning behind that position. The principles apply broadly to any situation where you're choosing between handing an agent a general-purpose CLI tool and building a purpose-fit harness around the functionality the agent actually needs.

---

## The Core Principle

If you already know what an agent is supposed to do, you should encode that knowledge into the tooling rather than asking the agent to figure it out at runtime.

An agent generating raw CLI commands or raw queries on the fly is doing work that we, the developers, already did when we designed the workflow. Offloading that work back to the model at inference time is wasteful at best and dangerous at worst.

The corollary: tools should exist to remove known failure modes, not just to add capabilities. If a model *can* do something through bash but frequently fails at it due to escaping, quoting, or syntax complexity, that's a signal to build a tool. The tool's value isn't that it does something the model can't — it's that it does something the model gets wrong.

---

## Security

### Agents Should Not Be Encouraged to Execute Terminal Commands

Letting agents issue arbitrary terminal commands is a pattern we should be moving away from, not leaning into. The more terminal commands an agent produces, the more likely it is that operators and users will habituate to approving them without reading them. That normalization of "click through and accept" is exactly how mistakes and abuse happen.

A purpose-built library replaces those terminal commands with well-defined tool calls that have clear, bounded behavior. There is nothing to review because the action is already scoped.

### Principle of Least Privilege

A CLI application gives an agent access to the full surface area of that tool. If the agent only needs to run three types of queries, it shouldn't have the ability to run every command the CLI supports. Using a library that only exposes the operations we actually need enforces least privilege at the tooling layer rather than relying on the model to self-restrict.

In cases where we already know the routines the agent should follow, giving it a wider set of capabilities than necessary is a safety concern. In the best case it's unused overhead. In the worst case it's an attack vector.

### Secret Management

Some CLI applications store credentials on disk, others use the system keychain. The storage mechanism varies, and some implementations are more responsible than others. But the real issue isn't where a specific CLI puts its token. The issue is that by depending on a CLI application for authentication, you're ceding control of your secrets management to whatever decisions that tool's maintainers made.

Different use cases should have different tokens. A token used by an agent running an automated workflow should not be the same token a developer uses for interactive CLI sessions. Purpose-driven, scoped credentials are a basic security practice. If that means you have multiple tokens for the same service across different tools, that's fine. That's the point.

A purpose-built library lets you own those decisions. You control how credentials are obtained, how they're scoped, how long they live, and where (or whether) they're stored. You're not subject to another team's opinions about secret lifecycle management, and you're not hoping that the next CLI update doesn't change the storage behavior you were depending on.

### Lateral Movement Risk

If a system is compromised, an installed and authenticated CLI application is a gift to an attacker. They don't need to steal credentials or figure out the API. The CLI is already configured and ready to use. Every agent running on that system, malicious or otherwise, is one terminal command away from full access.

By keeping authentication scoped to our library and out of the system-level toolchain, we limit the blast radius.

---

## Efficiency

### Token and Performance Cost

Having an agent construct raw CLI commands or raw queries (SQL, GraphQL, platform-specific query languages, whatever the system requires) is heavily inefficient. The model has to reason about syntax, field names, escaping, flags, and output parsing, all of which burns tokens and wall-clock time.

Replace that with a high-level tool call like `get_account_details(account_id)` and the model doesn't need to reason about any of that. The query is already written. The parsing is already handled. The token cost drops, latency drops, and reliability goes up.

### Error Reduction

Complex queries and multi-flag CLI commands are exactly the kind of thing language models get wrong in subtle ways. A misplaced quote, a wrong field name, a missing flag. These errors are hard to catch, hard to debug, and they compound in multi-step workflows.

Simple, well-defined tool calls are far more likely to succeed on the first attempt. They also require less capable (and less expensive) models to execute reliably. If a task can be accomplished with a straightforward function call instead of a generated query, there is no reason to use the harder path.

### Case Study: File Operations

The clearest illustration of this principle is file operations. Models can technically read, write, and edit files through bash — `sed`, `awk`, `echo` with redirects, heredocs, `cat <<EOF`. Every one of these works. And every one of these has a failure surface that models hit constantly:

- **`sed`** has escaping rules that models get wrong. Special characters, regex metacharacters, multi-line patterns, the difference between basic and extended regex — each is a class of error that compounds when the content being edited contains quotes, slashes, or brackets.
- **`awk`** requires understanding state machines for multi-line edits. Models generate syntactically plausible awk that silently does the wrong thing.
- **Heredocs** have quoting edge cases. `'EOF'` vs `EOF` controls variable expansion. Nested quotes inside heredocs interact with the shell's own quoting. Models get this wrong in ways that produce corrupted output.
- **`echo` with redirects**: `>` vs `>>` is a one-character difference between "replace the file" and "append to the file." Getting this wrong destroys content.
- **All of these** require the model to reason about shell escaping *on top of* the actual content it's trying to write or edit. That's two layers of reasoning — the edit and the shell mechanics of expressing the edit — where one layer would suffice.

Purpose-built file tools collapse this entire error surface. `Edit(file, old_string, new_string)` can't misquote, can't clobber, can't mis-escape. `Write(file, content)` can't accidentally append when it should replace. `Read(file)` can't fail because of a quoting issue in a `cat` command.

The tools don't add capability the model doesn't have. The model *can* do all of this through bash. The tools remove the ways the model *fails* at doing it through bash. That's the core principle in action: we know that `sed` escaping is error-prone for models, that heredoc quoting is a trap, that redirect operators are a clobber risk. We encode that knowledge into tools that bypass those failure modes entirely.

This is why some of the most effective agent harnesses use just a handful of tools — Read, Write, Edit, Bash — rather than exposing raw shell access for everything. Four tools that remove known failure classes outperform unrestricted bash access, because the tools encode the developers' knowledge of where models fail. Bash remains available for everything else — commands where the model is actually reliable at composing the syntax. The tool boundary is drawn at the failure boundary, not the capability boundary.

---

## Portability and Operations

### Environment Independence

A CLI application has to be installed and configured in every environment where the agent runs. That means managing installation, versioning, authentication, and configuration across dev machines, CI/CD pipelines, staging environments, and production servers.

A Python library is a dependency you declare in your project. It goes wherever your code goes. No separate install step, no system-level configuration, no "did you remember to authenticate the CLI on this box" debugging sessions.

### Headless and Platform-Constrained Environments

We may want agents to run headless on servers, inside containers, or within platforms that don't support arbitrary system-level dependencies. Requiring a CLI application installed at the OS level limits where and how we can deploy.

A library embedded in the application code runs anywhere the runtime runs.

### Reusability

A well-built library isn't just useful for the agent. The same abstractions can be reused in scripts, dashboards, other automations, and integrations. Because the library encapsulates the logic rather than delegating it to an external tool, it becomes a shared asset across the organization.

---

## Flexibility and Control

### Human-in-the-Loop

One of the most important capabilities in agentic workflows is the ability to insert human review at critical points. When you own the code that executes the operations, you can enforce elicitation, confirmation steps, and approval gates wherever you need them.

Try doing that with a CLI application. You're either wrapping it in brittle shell logic to intercept commands, or you're hacking the CLI's own config to insert pauses. Neither approach is maintainable.

With a library, human-in-the-loop is a first-class design consideration. You decide where the checkpoints go, what information gets surfaced for review, and how approvals flow. The control lives in your code, not in a tool you don't own.

### Workflow Customization

Internal processes often have requirements that don't map cleanly to a CLI's interface: logging, audit trails, conditional branching, rollback logic, notifications. When the integration lives in your own code, these are just features you add. When the integration is a CLI call, every additional requirement becomes a wrapper, a parser, or a workaround.

---

## When CLI Tools Do Make Sense

This isn't an absolute rule. There are cases where a CLI application is the right choice:

- **Exploratory work**: When a developer is manually investigating or prototyping, a CLI is the right tool.
- **One-off operations**: If you need to do something once and never again, writing a library for it is overkill.
- **Rapidly changing APIs**: If the underlying API is unstable and the CLI vendor is maintaining compatibility, it may be pragmatic to lean on their work.
- **Genuinely general-purpose agents**: If the agent's job is to be a flexible assistant that handles unpredictable requests, broader tool access may be appropriate, with corresponding guardrails.

But for defined, repeatable agentic workflows where we know what the agent needs to do, purpose-built tooling is the better path.

---

## Summary

| Concern | CLI Application | Purpose-Built Library |
|---|---|---|
| Installation & config | Required per environment | Bundled with the code |
| Agent permissions | Full CLI surface area | Only what's exposed |
| Secret management | Controlled by CLI maintainers | Controlled by your team |
| Token cost | High (query/command generation) | Low (simple tool calls) |
| Error rate | Higher (complex syntax) | Lower (pre-built operations) |
| Portability | OS-level dependency | Runtime dependency |
| Human-in-the-loop | Difficult to retrofit | First-class design option |
| Lateral movement risk | Authenticated CLI available system-wide | Scoped to the running process |
| Design intent | Expose raw capabilities | Remove known failure modes |

Build the harness. Encode what you know. Don't make the agent rediscover it every time it runs.
