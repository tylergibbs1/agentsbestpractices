# Agent Best Practices

Skills for designing, building, and optimizing tools that AI agents use effectively.

## Try it now

```
npx skills add tylergibbs1/agentsbestpractices
```

## What's included

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
└── designing-agent-tools/
    ├── SKILL.md                        # Main instructions (186 lines)
    ├── references/
    │   ├── tool-descriptions.md        # Description writing guide
    │   └── tool-evaluation.md          # Evaluation setup guide
    └── evals/
        └── evals.json                  # 7 evaluation tasks
```

## License

MIT
