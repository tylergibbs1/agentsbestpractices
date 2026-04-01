# Agent Design Patterns

## Contents
- Single Agent
- Prompt Chaining
- Routing
- Parallelization
- Orchestrator-Workers (Coordinator)
- Evaluator-Optimizer
- Hierarchical Task Decomposition
- Sequential Multi-Agent
- Loop and Ralph Loop
- Swarm
- Human-in-the-Loop
- ReAct

## Single Agent

One agent with tools in an autonomous loop. The agent interprets requests, plans steps, selects tools, and synthesizes results.

**When to use:** Multi-step tasks requiring external data or actions. Start here before going multi-agent.

**Limits:** Performance degrades with too many tools or too much complexity. If you observe increased latency, wrong tool selection, or task failure — consider decomposing.

**Key insight:** Popular agents use surprisingly few tools. Claude Code uses ~12. Manus uses <20. Curate your action space.

## Prompt Chaining

Decompose a task into fixed sequential steps. Each LLM call processes the output of the previous one. Add programmatic gates between steps.

```
Step 1 → [gate] → Step 2 → [gate] → Step 3
```

**When to use:** Task cleanly decomposes into fixed subtasks. Trades latency for accuracy by making each call simpler.

**Examples:** Generate copy → translate. Write outline → validate → write document.

## Routing

Classify input, direct to specialized handler. Separation of concerns — each handler has its own optimized prompt.

```
Input → Classifier → Handler A / Handler B / Handler C
```

**When to use:** Distinct categories requiring different treatment. Classification must be accurate.

**Examples:** Support queries → general / refund / technical. Easy questions → Haiku, hard → Opus.

## Parallelization

Run sub-tasks simultaneously, aggregate results. Two variants:

- **Sectioning** — break task into independent parallel subtasks
- **Voting** — run same task multiple times for diverse outputs

**When to use:** Independent sub-tasks where speed matters, or multiple perspectives improve confidence.

**Examples:** Content moderation (guardrails + response in parallel). Code review (multiple reviewers checking different aspects).

## Orchestrator-Workers (Coordinator)

Central agent dynamically decomposes tasks and delegates to specialized workers. Unlike parallelization, subtasks aren't predefined — the orchestrator determines them based on input.

```
User → Coordinator (AI model) → Worker A, Worker B, Worker C → Synthesize
```

**When to use:** Complex tasks where subtasks can't be predicted upfront. Key distinction from parallelization: flexibility at the cost of an extra model call for orchestration.

**Trade-off:** Higher latency and cost from multiple model calls. Better quality through specialization.

**Examples:** Coding agents making changes across multiple files. Research gathering from multiple sources.

## Evaluator-Optimizer

One agent generates, another evaluates and provides feedback in a loop.

```
Generator → Output → Evaluator → Feedback → Generator → ... → Final
```

**When to use:** Clear evaluation criteria exist. LLM responses demonstrably improve with feedback. Analogous to human iterative editing.

**Examples:** Literary translation with quality critic. Search tasks requiring multiple rounds of refinement.

## Hierarchical Task Decomposition

Multi-level hierarchy. Root agent decomposes task → delegates to mid-level agents → they further decompose → worker agents execute.

**When to use:** Ambiguous, open-ended problems requiring multi-step reasoning across levels. Research, planning, synthesis.

**Trade-off:** Highest quality for complex tasks, but significant latency and cost from nested model calls. Most complex to debug.

## Sequential Multi-Agent

Fixed linear pipeline. Agent A → Agent B → Agent C. No AI model needed for orchestration — predefined logic controls flow.

**When to use:** Highly structured, repeatable processes. Data extraction → cleaning → loading.

**Trade-off:** Low cost (no orchestration model calls), but rigid — can't skip or adapt steps.

## Loop and Ralph Loop

### Basic Loop
Repeat agents until exit condition met. No AI model for orchestration.

**Risk:** Infinite loops if exit conditions are poorly defined.

### Ralph Loop
For tasks exceeding a single context window:

1. Initializer agent creates plan file + tracking file
2. Sub-agents tackle individual plan items
3. Stop hooks verify work after each iteration
4. Loop until plan satisfied

Context persists in files and git history. Each sub-agent gets a fresh context window.

**Key insight:** It's often infeasible for a long-running task to fit one context window. The Ralph loop solves this by making files the persistent medium and agents disposable.

## Swarm

All-to-all communication. Dispatcher routes to a group of specialized agents that can communicate with each other, share findings, critique proposals, and hand off tasks.

```
User → Dispatcher → Agent A ↔ Agent B ↔ Agent C → Result
```

**When to use:** Highly complex problems benefiting from debate and diverse perspectives.

**Trade-off:** Most complex and costly pattern. No central orchestrator means risk of unproductive loops. Requires explicit exit conditions (iteration limit, consensus, time bound).

## Human-in-the-Loop

Agent pauses at predefined checkpoints, calls external system, waits for human review before continuing.

**When to use:** High-stakes decisions, safety-critical operations, subjective approvals, compliance requirements.

**Examples:** Financial transaction approval. Sensitive data anonymization validation. Creative content sign-off.

## ReAct (Reason and Act)

Iterative loop: Think → Act → Observe → Think → ...

- **Think:** Model reasons about task, decides next step
- **Act:** Selects tool and executes, or formulates final answer
- **Observe:** Receives tool output, saves to memory, builds on prior observations

**When to use:** Complex, dynamic tasks requiring continuous planning and adaptation.

**Trade-off:** Transparent reasoning (debuggable), but higher latency from multi-step loop. Errors in one observation can propagate.

## Pattern Comparison Quick Reference

| Pattern | Orchestration | Latency | Cost | Flexibility |
|---|---|---|---|---|
| Single agent | Model-driven | Medium | Low | High |
| Prompt chaining | Predefined | Medium | Low | Low |
| Routing | Classifier | Low | Low | Medium |
| Parallelization | Predefined | Low | Medium | Low |
| Coordinator | Model-driven | High | High | High |
| Evaluator-optimizer | Loop | High | Medium | Medium |
| Hierarchical | Multi-level model | Very high | Very high | Very high |
| Sequential | Predefined | Medium | Low | Very low |
| Loop / Ralph | Predefined | Variable | Variable | Medium |
| Swarm | Peer-to-peer | Very high | Very high | Very high |
| ReAct | Model-driven | High | Medium | High |
