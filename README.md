# Agent Best Practices

Skills for designing effective AI agents, tools, prompts, and CLIs.

## Try it now

```
npx skills add tylergibbs1/agentsbestpractices
```

## What's included

### designing-agents

Architecture patterns and context engineering strategies for agentic AI systems.

- **Pattern selection** — single agent, routing, parallelization, coordinator, evaluator-optimizer, hierarchical, swarm, ReAct, Ralph loop
- **Context engineering** — progressive disclosure, offloading, caching, isolation, multi-layer action space
- **Context evolution** — trajectory reflection, memory learning, skill discovery, sleep-time pattern

### designing-agent-tools

Building tools agents find ergonomic — schemas, descriptions, responses, evaluation.

- **Tool boundaries** — consolidate workflow-level tools instead of 1:1 API wrapping
- **Schema design** — parameter naming, enums, 4-part description anatomy
- **Response design** — concise vs detailed formats, pagination, actionable errors
- **Evaluation** — task generation, train/test splits, agent loops, iterative improvement

### prompting-agents

Model-agnostic prompt engineering for agentic systems.

- **System prompt structure** — XML tags, role setting, context ordering
- **Tool use steering** — action vs suggestion, parallel calling, aggressiveness calibration
- **Thinking control** — chain-of-thought, self-checking, overthinking reduction
- **Long-running agents** — multi-window state, context compaction, progress tracking
- **Copy-paste patterns** — ready-to-use prompt blocks for common behaviors

### building-agent-clis

Designing CLIs that AI agents can use safely and effectively.

- **Agent-first design** — raw JSON payloads, schema introspection, field masks
- **Input hardening** — validation for hallucination-specific failure modes
- **Safety rails** — dry-run, response sanitization, confirmation patterns
- **Multi-surface** — same binary serves CLI, MCP, env vars, extensions
- **Retrofit checklist** — incremental steps to make existing CLIs agent-friendly

## Structure

```
skills/
├── designing-agents/
│   ├── SKILL.md
│   ├── references/
│   │   ├── design-patterns.md
│   │   └── context-engineering.md
│   └── evals/
│       └── evals.json
├── designing-agent-tools/
│   ├── SKILL.md
│   ├── references/
│   │   ├── tool-descriptions.md
│   │   └── tool-evaluation.md
│   └── evals/
│       └── evals.json
├── prompting-agents/
│   ├── SKILL.md
│   ├── references/
│   │   ├── thinking-config.md
│   │   └── prompt-patterns.md
│   └── evals/
│       └── evals.json
└── building-agent-clis/
    ├── SKILL.md
    ├── references/
    │   └── input-hardening.md
    └── evals/
        └── evals.json
```

## License

MIT
