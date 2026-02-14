---
id: output-determinism-structured-vs-freeform-vs-rubric
title: "Output Determinism: Structured Outputs, Free Generation, and Rubric Evaluation"
type: reference
scope: architecture
domain:
  - agent-development
  - output-quality
  - reliability
stability: medium
intent: guide
tags:
  - structured-outputs
  - constrained-decoding
  - rubrics
  - quality-gates
  - determinism
  - output-validation
  - llm-as-judge
  - decision-framework
  - schema-validation
source: conversation
confidence: medium
created: 2026-02-14
---

# Output Determinism: Structured Outputs, Free Generation, and Rubric Evaluation

When an agent produces output — a JSON payload for a downstream system, a bug report for a human, a decision that feeds another agent — what determines whether that output is reliable? The answer isn't one mechanism. It's three distinct layers, each solving a different problem, each with different costs, and each composable with the others.

The common framing of "structured outputs vs. free generation" collapses these layers into a single binary choice. That framing loses the important distinctions. A perfectly schema-valid JSON bug report can be useless. A free-text analysis with no format constraints can be exactly right. The question isn't "structured or unstructured" — it's "which layers of output quality does this use case actually need?"

## The Three Layers of Output Quality

```
Format Compliance ──→ Content Accuracy ──→ Semantic Quality
 "Is this valid?"      "Is this correct?"    "Is this good enough?"
```

Each layer solves a different class of failure, requires different mechanisms, and has different cost profiles. They build on each other but are independently addressable.

**Format compliance.** Does the output match the expected structure? Valid JSON, correct field names, expected types, no missing required fields. This is the problem structured outputs solve. The failure mode is mechanical: a downstream parser crashes, an API rejects the payload, a pipeline step gets null where it expected a string.

**Content accuracy.** Are the values correct? The JSON is valid and the fields are present, but is the severity rating right? Is the account ID the one the user asked about? Did the agent cite the correct standard? This is a factual correctness problem. Format validation can't catch it — a well-formed lie passes every schema check. Verification requires ground truth: checking against a database, running a test, comparing to known values.

**Semantic quality.** Is the output good enough for its purpose? The bug report has all required fields with accurate values, but is the description clear? Are the reproduction steps sufficient for another engineer to follow? Is the severity justification convincing? This is a judgment problem. It's what a senior reviewer evaluates when they read a draft — not whether the format is right or the facts are wrong, but whether the artifact achieves its purpose.

Most discussions about output reliability conflate these layers. "Use structured outputs for reliability" addresses only the first layer. "The model is good enough now" usually refers to the first layer being less of a concern, while saying nothing about the second or third. "Use rubrics" addresses the third layer but doesn't help when the output can't be parsed.

**The key insight:** Schema validation is necessary but insufficient for quality. A perfectly valid JSON bug report can still be useless. Conversely, an unstructured free-text analysis can be exactly right for its purpose if no downstream system needs to parse it. Match the mechanism to the actual failure mode you're preventing.

## Constrained Decoding and Structured Outputs

Constrained decoding (also called guided generation or grammar-constrained generation) restricts the model's token space at each generation step to enforce a target schema. The output is guaranteed to be valid against the schema. Every time. No parsing errors, no missing fields, no unexpected types.

Several mechanisms exist: API-level response schemas (OpenAI's `response_format`, Anthropic's tool-use schemas), client-side grammar constraints (llama.cpp grammars, Outlines, LMQL), and framework-level validation (Pydantic models in PydanticAI, BAML type definitions). They all share the same fundamental approach — constrain what the model can generate so the output matches a predefined structure.

**What constrained decoding guarantees:**

- Format compliance on every invocation. No retries needed for malformed output.
- Type safety for downstream consumers. The integer field contains an integer. The enum field contains one of the allowed values.
- Parseable output without heuristic extraction. No regex, no "find the JSON in the markdown," no hoping the model remembered to close its braces.

**What constrained decoding costs:**

Constraining the token space at each generation step can force the model away from its natural reasoning path. When the correct answer requires tokens that the constraint eliminates, the model produces a schema-valid but incorrect response. This isn't theoretical — it's been measured.

- When a model needs to output a decimal value like `0.46` but the schema expects an integer, constrained decoding produces `1` rather than the correct answer. The output is schema-valid. It's also wrong.
- Chain-of-thought reasoning degrades when the model must generate within structural constraints from the start. The model can't "think out loud" in a way that the schema doesn't accommodate.
- For complex schemas with nested objects, unions, and conditional fields, the constraint filtering becomes aggressive enough that the model effectively loses access to significant portions of its reasoning capability.

The degradation is proportional to how much the constraint restricts the natural generation. Simple schemas (a few flat fields with common types) impose minimal cost. Complex schemas with many constraints can measurably reduce output accuracy.

**When constrained decoding is essential:**

- Machine-consumed interfaces where a format violation causes a crash, not a degraded experience
- Pipeline processing at scale, where even a 1-2% format failure rate creates operational pain across thousands of invocations
- Multi-agent handoffs where the downstream agent has no tolerance for ambiguity — the handoff contract (see *Agent Workflow Decomposition*) needs to be mechanically parseable
- APIs and integrations where the consuming system has a rigid schema and no fallback parser

**When constrained decoding is counterproductive:**

- Complex reasoning tasks where the model needs full token-space access to think through tradeoffs, weigh evidence, or construct nuanced arguments
- Open-ended generation where the output structure should emerge from the content, not be imposed on it
- Human-consumed outputs where a format violation is cosmetic, not catastrophic
- Exploratory tasks where you don't know the output structure in advance

**Rule of thumb:** If the output's first consumer is a parser, constrain it. If the output's first consumer is a human, don't.

## Free Generation with Post-Hoc Handling

The model generates freely — no token-space constraints, no schema enforcement during generation. Format compliance is handled after the fact, either through parsing (extract structured data from the free-text output) or through a separate extraction step (ask the model to reason freely, then ask it again to format the conclusion).

**The split approach — reason freely, then constrain extraction:**

This is emerging as the best general-purpose pattern for tasks that need both reasoning quality and structured output. Two steps:

1. The model reasons freely about the problem. Full token-space access, natural chain-of-thought, no structural constraints. The output is whatever the model naturally produces — paragraphs, lists, hedged conclusions, explored alternatives.
2. A second call (or a tool-call step within the same turn) extracts the structured conclusion from the reasoning. This step uses constrained decoding or schema validation, but the input is the model's own unconstrained reasoning, not the raw task. The extraction is a much simpler cognitive task than the original reasoning, so the constraint cost is minimal.

This preserves reasoning quality while delivering structured output. The tradeoff is latency and cost — two model calls instead of one. For interactive use cases where the user is waiting, this may be too slow. For pipeline processing where accuracy matters more than speed, it's usually worth it.

**Relying on instruction-following alone:**

Current frontier models (Claude 4.x, GPT-4.1 and later, Gemini 2.x) follow format instructions with high reliability for simple schemas. "Respond with a JSON object containing these fields" works most of the time. The failure rate has dropped substantially — what used to be a 10-20% failure rate on format instructions is now 1-3% for simple schemas on frontier models, and lower for well-crafted prompts.

But "most of the time" and "low failure rate" are different guarantees than "every time." At interactive scale (one user, one request), a 2% failure rate is a minor annoyance — retry and move on. At pipeline scale (10,000 requests per day), a 2% failure rate is 200 failures per day, each needing error handling, retry logic, or manual intervention.

**Tradeoffs:**

- Full reasoning capability preserved — no token-space constraints during the thinking phase
- Works with any model, no API-specific schema features needed
- Flexible — the output structure can adapt to the content rather than being imposed
- Format violations still happen, especially with complex schemas, under-specified instructions, or less capable models
- Post-hoc parsing is brittle if the model's output format drifts between invocations
- The split approach doubles latency and token cost

**Rule of thumb:** If your volume is low enough that you can afford retries, and your schema is simple enough that frontier models follow it reliably, instruction-based formatting may be sufficient. If either condition doesn't hold, add a structural guarantee — either constrained decoding or a validation-and-retry layer.

## Rubric-Based Evaluation

Rubrics solve the semantic quality problem — the layer that schema validation fundamentally cannot address. A rubric defines criteria for what "good" looks like and evaluates the output against those criteria. The evaluation can be done by a separate model (LLM-as-judge), by the same model in a review step (self-evaluation), by a dedicated review agent (see *Agent Workflow Decomposition: The Review/Quality Gate Pattern*), or by a human.

**What rubrics evaluate that schemas can't:**

- Clarity: is the description understandable to someone who didn't write it?
- Completeness: are the reproduction steps sufficient to actually reproduce the bug?
- Accuracy of judgment: is the severity rating appropriate given the described impact?
- Coherence: does the analysis follow logically from the evidence?
- Actionability: can someone act on this output without asking follow-up questions?

These are the dimensions that determine whether an output is useful, not just parseable. A bug report that passes schema validation but has vague reproduction steps wastes the receiving engineer's time. A compliance assessment that correctly cites every standard but misapplies the applicability criteria is worse than one that cites fewer standards but applies them correctly.

**LLM-as-judge patterns:**

A second model (or the same model in a separate call) evaluates the output against explicit criteria. The judge receives the output, the rubric, and optionally the original task, and produces a structured evaluation: pass/fail per criterion, with explanations for failures.

This works well when:
- The evaluation criteria can be articulated as specific, measurable dimensions
- The judge model is at least as capable as the generating model
- The cost of evaluation is justified by the cost of a bad output reaching its consumer
- You need consistent evaluation across many outputs (human review doesn't scale, rubric-based LLM evaluation does)

This works poorly when:
- The rubric criteria are vague ("is this good?") — the judge needs specific dimensions to evaluate against
- The evaluation requires domain knowledge the judge model doesn't have
- The evaluation is more expensive than the original generation and the failure cost doesn't justify it
- Self-evaluation (same model judging its own output) introduces systematic blind spots — models are biased toward rating their own output favorably

**Connecting to the review agent pattern:**

The review/quality gate pattern from *Agent Workflow Decomposition* is rubric evaluation instantiated as an architectural component. The review agent is a rubric evaluator: it takes a draft, a set of criteria, and produces a structured judgment. The key design principles transfer directly:

- The review agent (rubric evaluator) should not modify the output — it evaluates and reports
- The rubric should be explicit and specific, not "check if this is complete"
- Evaluation should be a separate invocation with its own context, not buried in the generating agent's session
- The output of evaluation is itself a structured artifact: which criteria passed, which failed, and why

**When rubrics outperform schemas as quality mechanisms:**

- Complex artifacts where quality is multi-dimensional (bug reports, design reviews, compliance assessments)
- Subjective quality dimensions that can be operationalized but not mechanized (clarity, actionability, appropriate severity)
- Tasks where the failure mode is "confidently wrong but structurally valid" — the most dangerous failure mode, and the one that schema validation systematically misses
- Workflows where iterative improvement is possible — the rubric evaluation produces specific feedback that guides revision

**Rule of thumb:** If a senior reviewer would look at the output and have opinions about quality beyond "is it formatted correctly," that review process should be captured as a rubric, not a schema.

## Decision Heuristics

| Question | Answer | Approach |
|----------|--------|----------|
| Is the output consumed by a parser or API? | Yes | **Constrained decoding** — format failures are crashes |
| Is the output consumed by a human? | Yes | **Free generation** — format flexibility, optimize for readability |
| Is it consumed by both? | Yes | **Split approach** — reason freely, extract structured |
| What volume? | >1,000/day | **Constrained decoding or validation layer** — error rates compound |
| What volume? | Interactive (single user) | **Instruction-following may suffice** — retry is cheap |
| What's the reasoning complexity? | High | **Free generation** with optional structured extraction — don't constrain reasoning |
| What's the reasoning complexity? | Low (slot-filling, classification) | **Constrained decoding** — minimal accuracy cost |
| Is the failure mode "wrong but valid"? | Yes | **Rubric evaluation** — schemas can't catch this |
| Is the failure mode "can't parse"? | Yes | **Constrained decoding** — guaranteed parseable |
| Does quality have subjective dimensions? | Yes | **Rubric evaluation** — operationalize the review |
| Is this a multi-agent handoff? | Yes | **Constrained decoding on the handoff artifact** — see workflow decomposition |
| Is iterative improvement possible? | Yes | **Rubric evaluation** — provides actionable revision feedback |

When multiple rows point to different approaches, you need composition. The most common composition: constrained decoding for the artifact structure, rubric evaluation for the artifact quality.

## Composing the Layers

The layers aren't mutually exclusive. Most production systems need more than one, and the composition pattern depends on the use case.

**Pattern 1: Constrained decoding alone.**

Single-layer. The output must match a schema. Format compliance is both necessary and sufficient — if the fields are present and the types are correct, the output is acceptable.

Use for: data extraction from documents, classification tasks, slot-filling, API field mapping. Tasks where the cognitive challenge is low and the format requirement is strict.

**Pattern 2: Free generation → constrained extraction.**

Two-layer. The model reasons freely about the problem, then a second step extracts the structured conclusion. Format compliance is necessary but reasoning quality matters more.

Use for: analysis tasks that produce a structured result, complex classification where the model needs to reason through edge cases, any task where constraining generation from the start would degrade accuracy.

**Pattern 3: Constrained decoding + rubric evaluation.**

Two-layer. The output is structurally valid (constrained decoding ensures this) and also semantically adequate (rubric evaluation checks this). This is defense in depth — the schema catches format failures, the rubric catches quality failures.

Use for: high-stakes artifacts consumed by both machines and humans (compliance reports, audit findings), multi-agent handoffs where the downstream agent needs both parseable input and high-quality content, pipeline outputs where "valid but wrong" is the dangerous failure mode.

**Pattern 4: Free generation → constrained extraction → rubric evaluation.**

Three-layer. Full composition. The model thinks freely, the result is extracted into a structured format, and the structured result is evaluated against quality criteria. This is the most thorough and the most expensive.

Use for: complex artifacts where reasoning quality, format compliance, and semantic quality all matter — and where the cost of a bad output justifies three processing steps. Compliance assessments, security reviews, architectural recommendations.

**When fewer layers are better:**

Every layer adds latency, token cost, and system complexity. Don't add layers to feel thorough. Add them because the failure mode they prevent actually occurs and actually matters.

- If format failures don't happen in practice (simple schema, capable model, low volume), skip constrained decoding and rely on instruction-following
- If semantic quality isn't a meaningful dimension (the task is mechanical, the answer is objectively right or wrong), skip rubric evaluation
- If the output goes directly to a human who can evaluate quality themselves, skip rubric evaluation — the human is the rubric

**Rule of thumb:** Start with the minimum layers needed for your actual failure modes. Add layers when you have evidence that a failure mode exists, not when you fear it might.

## The Model Capability Trajectory

This section is an honest assessment of where things stand, not a prediction of where they're going. The trajectory matters because it changes the cost-benefit analysis of each layer.

**Format compliance is getting cheaper to guarantee.**

Frontier models follow format instructions more reliably than they did 18 months ago. The gap between "instruction-following" and "constrained decoding" for format compliance is narrowing. For simple schemas, the gap is already small enough that constrained decoding is insurance, not necessity. For complex schemas, the gap is still meaningful.

This means: constrained decoding is shifting from "required for reliability" to "defense in depth." The cost of adding it is low (API-level schema features are easy to use). The benefit is shrinking for simple cases. The benefit remains significant for complex schemas and high-volume pipelines.

**Content accuracy is improving but not solved.**

Models are less likely to hallucinate facts when relevant context is provided. Grounding through retrieval (see *Agent Knowledge Architecture*) and tool use reduces factual errors. But content accuracy is fundamentally a harder problem than format compliance — it requires the model to not just follow instructions about structure but to reason correctly about substance. Improvement here is gradual, not a step function.

This means: verification layers (ground truth checks, tool-assisted validation) remain important. Don't confuse improved format compliance with improved factual accuracy.

**Semantic quality is not improving at the same rate.**

A model can follow a format instruction perfectly and still produce a vague, unhelpful analysis. Semantic quality depends on reasoning depth, domain understanding, and judgment — capabilities that improve slowly relative to instruction-following. The quality gap between "technically correct" and "genuinely useful" is not closing as fast as the format compliance gap.

This means: rubric evaluation is becoming proportionally more important, not less. As format compliance becomes easier to guarantee, the remaining quality failures are increasingly semantic — exactly the type that rubrics catch and schemas miss. Invest in rubrics proportionally more as models improve at format compliance.

**The practical implication:**

If you're deciding where to invest engineering effort in output quality:

1. Format compliance infrastructure (constrained decoding, validation layers) is mature and becoming commoditized. Use it, but don't over-engineer it.
2. Content accuracy infrastructure (retrieval, grounding, tool-assisted verification) is where the current active development is.
3. Semantic quality infrastructure (rubric evaluation, review agents, quality feedback loops) is underinvested relative to its importance and is where the highest marginal returns are.

## Anti-Patterns

**Using constrained decoding for complex reasoning tasks.** If the task requires the model to weigh evidence, consider alternatives, and reach a nuanced conclusion, constraining the output format from the start sacrifices reasoning quality for format compliance. Use the split approach instead — let the model reason freely, then extract the structured result.

**Assuming schema-valid output is correct output.** A JSON object with all required fields and correct types tells you nothing about whether the values are right. If your quality assurance process is "validate the schema and ship it," you're catching the least dangerous failure mode (format violations, which are obvious) and missing the most dangerous one (confident errors wrapped in valid structure).

**Rubric evaluation without specific criteria.** "Evaluate whether this output is good" is not a rubric. "Evaluate whether: (1) the reproduction steps are specific enough to follow without additional context, (2) the severity rating is justified by the described impact, (3) the affected systems list is complete based on the description" is a rubric. Vague rubrics produce vague evaluations. Specific rubrics produce actionable feedback.

**Over-engineering format validation for human-consumed outputs.** If the output is a summary that a person reads, spending engineering effort on schema validation is solving a problem nobody has. The person reading the summary is a better format validator than any schema. Focus on semantic quality instead.

**Under-engineering format validation for machine-consumed interfaces.** The inverse mistake. If the output feeds a pipeline, a 1% format failure rate at 10,000 requests/day is 100 failures requiring error handling. "The model usually gets it right" is not an architecture.

**Evaluating the generating model with itself.** Self-evaluation has systematic blind spots — models are biased toward rating their own output favorably and tend to miss the same errors in review that they made in generation. Use a separate model, a separate prompt, or ideally a separate agent invocation with different instructions and context.

**Treating structured outputs as the solution to output determinism.** Structured outputs solve format determinism — the same schema produces the same structure every time. They do not solve semantic determinism — the same prompt does not produce the same reasoning or conclusions every time. If what you actually need is reproducible decision-making, structured outputs address the wrong layer. You need deterministic inputs (grounded retrieval, explicit criteria) and deterministic evaluation (rubric-based assessment), not just deterministic formatting.

## The Core Design Principle

Output quality has three layers — format compliance, content accuracy, and semantic quality — and each is solved by a different mechanism. Structured outputs guarantee format. Grounding and verification address accuracy. Rubrics evaluate quality. The right combination depends on who consumes the output, what the failure cost is, and which failure modes you can actually afford.

The most common mistake is solving the easiest layer (format compliance) and assuming the output is reliable. The most expensive mistake is adding layers you don't need. Match the mechanism to the actual failure mode, not the theoretical one.
