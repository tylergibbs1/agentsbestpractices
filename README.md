# Agent Best Practices

Everything you need to build effective AI agents — architecture, tools, prompts, and CLIs. Distilled from Anthropic, Google, and practitioner research into installable agent skills.

## Install

```
npx skills add tylergibbs1/agentsbestpractices
```

## Skills

### [designing-agents](skills/designing-agents/SKILL.md)

**Pick the right architecture for your agent.** 12 design patterns with trade-offs, context engineering strategies, and a decision framework for when to go multi-agent.

| You'll learn | From |
|---|---|
| Single agent vs multi-agent decision framework | Anthropic's "Building effective agents" |
| Context engineering (progressive disclosure, offloading, caching) | Lance Martin's agent design patterns |
| Ralph loop for long-running tasks | Claude Code patterns |
| Coordinator, hierarchical, swarm patterns | Google's agent design guide |

### [designing-agent-tools](skills/designing-agent-tools/SKILL.md)

**Build tools agents actually use correctly.** Schema design, description writing, response formatting, and evaluation-driven improvement.

| You'll learn | From |
|---|---|
| Fewer, deeper tools beat many shallow tools | Anthropic's tool writing guide |
| 4-part description anatomy (what/when/how/constraints) | SWE-bench tool optimization |
| Response format control (concise vs detailed) | Slack MCP optimization |
| Evaluation loops with agent transcript analysis | Tool evaluation cookbook |

### [prompting-agents](skills/prompting-agents/SKILL.md)

**Write prompts that steer agents effectively.** Model-agnostic principles with ready-to-use prompt blocks. Works across Claude, GPT, Gemini.

| You'll learn | From |
|---|---|
| System prompt structure with XML tags | Anthropic prompting best practices |
| Action vs suggestion steering | Claude 4.6 migration guide |
| Multi-window state management (tests.json, git checkpoints) | Long-horizon agent patterns |
| Copy-paste blocks for common fixes (verbosity, hallucinations, overengineering) | Production agent debugging |

### [building-agent-clis](skills/building-agent-clis/SKILL.md)

**Design CLIs that agents can use safely.** Input hardening, schema introspection, context window discipline, and a retrofit checklist for existing CLIs.

| You'll learn | From |
|---|---|
| Raw JSON payloads over bespoke flags | Google Workspace CLI (gws) |
| Input hardening against hallucinations (path traversal, double encoding) | Agent-first CLI design |
| Schema introspection replaces documentation | Runtime discovery patterns |
| Multi-surface design (CLI + MCP + env vars) | Agent DX principles |

## How skills work

Each skill follows the [Agent Skills spec](https://agentskills.io/specification.md):

```
skills/<skill-name>/
├── SKILL.md           # Main instructions (loaded when triggered)
├── references/        # Detailed guides (loaded on demand)
└── evals/
    └── evals.json     # Evaluation tasks
```

Only the skill metadata is loaded into context at startup. The full SKILL.md is read when the skill triggers. Reference files are loaded only when needed — progressive disclosure keeps your context window clean.

## Sources

These skills distill guidance from:

- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — Anthropic
- [How to write tools](https://www.anthropic.com/engineering/how-to-write-tools) — Anthropic
- [Prompting best practices](https://docs.anthropic.com/en/docs/build-with-claude/prompting-best-practices) — Anthropic
- [Agent design patterns](https://lancemartin.notion.site/agent-design-patterns) — Lance Martin
- [Design pattern for your agentic AI system](https://cloud.google.com/discover/agent-design-pattern) — Google Cloud
- [You Need to Rewrite Your CLI for AI Agents](https://jpoehnelt.dev/posts/rewrite-cli-for-ai-agents/) — Justin Poehnelt
- [Skill authoring best practices](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/authoring-best-practices) — Anthropic

## Contributing

1. Fork the repo
2. Create a branch (`feature/skill-name`)
3. Follow the skill structure above — SKILL.md under 500 lines, references one level deep
4. Include evals with realistic tasks (not trivial ones)
5. Submit a PR

## License

MIT
