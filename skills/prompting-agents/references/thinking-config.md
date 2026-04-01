# Thinking Configuration Guide

How to configure and steer Claude's thinking behavior across models.

## Contents
- Adaptive Thinking (Claude 4.6)
- Extended Thinking (Legacy)
- Migration Path
- Prompting Thinking Behavior

## Adaptive Thinking (Claude 4.6)

Claude 4.6 models use `thinking: {type: "adaptive"}` — Claude dynamically decides when and how much to think based on two factors:

1. **Effort parameter** — controls thinking depth (`low`, `medium`, `high`, `max`)
2. **Query complexity** — harder queries trigger more thinking automatically

On easy queries that don't require thinking, the model responds directly. In internal evaluations, adaptive thinking reliably outperforms manual extended thinking.

### Configuration

```python
client.messages.create(
    model="claude-opus-4-6",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},
    messages=[{"role": "user", "content": "..."}],
)
```

### Effort Guidelines

| Effort | Latency | Quality | Best for |
|---|---|---|---|
| `low` | Fastest | Good | High-volume, classification, simple lookups |
| `medium` | Moderate | Better | Most applications, chat, content generation |
| `high` | Slower | Best | Agentic coding, multi-step reasoning, research |
| `max` | Slowest | Highest | Large-scale migrations, deep research, hardest problems |

### Sonnet 4.6 Specifics

- Defaults to `high` effort (may cause higher latency than Sonnet 4.5)
- Set effort explicitly: `medium` for most apps, `low` for latency-sensitive
- Set large max output tokens (64k recommended) at medium/high effort

### Disabling Thinking

If you don't need thinking, explicitly disable it:

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    thinking={"type": "disabled"},
    output_config={"effort": "low"},
    messages=[{"role": "user", "content": "..."}],
)
```

## Extended Thinking (Legacy)

Extended thinking with `budget_tokens` is deprecated on Claude 4.6 but still functional. If keeping temporarily during migration:

```python
# Coding use cases — start with medium effort
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16384,
    thinking={"type": "enabled", "budget_tokens": 16384},
    output_config={"effort": "medium"},
    messages=[{"role": "user", "content": "..."}],
)

# Chat/non-coding — start with low effort
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    thinking={"type": "enabled", "budget_tokens": 16384},
    output_config={"effort": "low"},
    messages=[{"role": "user", "content": "..."}],
)
```

16k budget provides headroom without runaway token usage.

## Migration Path

```
Extended thinking (budget_tokens)  →  Adaptive thinking (effort)
─────────────────────────────────────────────────────────────────
budget_tokens: 4000               →  effort: "low"
budget_tokens: 8000-16000         →  effort: "medium"
budget_tokens: 32000              →  effort: "high"
budget_tokens: 64000+             →  effort: "max"
```

## Prompting Thinking Behavior

### Encourage Deeper Thinking

```
After receiving tool results, carefully reflect on their quality and determine
optimal next steps before proceeding. Use your thinking to plan and iterate.
```

### Reduce Unnecessary Thinking

```
Extended thinking adds latency and should only be used when it will meaningfully
improve answer quality — typically for problems requiring multi-step reasoning.
When in doubt, respond directly.
```

### Prevent Decision Thrashing

```
When deciding how to approach a problem, choose an approach and commit to it.
Avoid revisiting decisions unless you encounter new information that directly
contradicts your reasoning.
```

### Self-Checking

Append to prompts for coding and math:

```
Before you finish, verify your answer against [test criteria].
```

### Manual Chain-of-Thought (Thinking Disabled)

When thinking is off, use structured tags:

```
Think through the problem step by step in <thinking> tags, then provide
your final answer in <answer> tags.
```

### Few-Shot with Thinking

Include `<thinking>` tags inside examples to demonstrate the reasoning pattern:

```xml
<example>
<user_query>What's the time complexity of binary search?</user_query>
<thinking>
Binary search halves the search space each iteration.
Starting with n elements: n → n/2 → n/4 → ... → 1
Number of steps = log₂(n)
</thinking>
<answer>O(log n)</answer>
</example>
```

Claude generalizes the reasoning style to its own thinking blocks.
