# Agent Best Practices

Skills for designing effective AI agents and the tools they use.

## Try it now

```
npx skills add tylergibbs1/agentsbestpractices
```

## What's included

### designing-agents

Architecture patterns and context engineering strategies for agentic AI systems. Covers:

- **Pattern selection** — single agent, routing, parallelization, coordinator, evaluator-optimizer, hierarchical, swarm, ReAct, Ralph loop
- **Context engineering** — progressive disclosure, offloading, caching, isolation, multi-layer action space
- **Context evolution** — trajectory reflection, memory learning, skill discovery, sleep-time pattern
- **Anti-patterns** — premature multi-agent, context stuffing, summarize-first, unverified loops

### designing-agent-tools

Expert guidance for building tools agents actually find ergonomic. Covers:

- **Tool boundaries** — consolidate workflow-level tools instead of 1:1 API wrapping
- **Schema design** — parameter naming, enums, descriptions with the 4-part anatomy
- **Response design** — concise vs detailed formats, pagination, actionable errors
- **Namespacing** — service and resource-level prefixes for multi-tool agents
- **MCP servers** — quick start, tool annotations, DXT packaging
- **Evaluation** — task generation, train/test splits, agent loops, metrics, iterative improvement

## Structure

```
skills/
├── designing-agents/
│   ├── SKILL.md                        # Architecture patterns & context strategies
│   ├── references/
│   │   ├── design-patterns.md          # Pattern catalog with trade-offs
│   │   └── context-engineering.md      # Context management strategies
│   └── evals/
│       └── evals.json                  # 7 evaluation tasks
└── designing-agent-tools/
    ├── SKILL.md                        # Tool design guidance
    ├── references/
    │   ├── tool-descriptions.md        # Description writing guide
    │   └── tool-evaluation.md          # Evaluation setup guide
    └── evals/
        └── evals.json                  # 7 evaluation tasks
```

## License

MIT
