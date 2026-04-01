---
name: prompting-agents
description: Writes and optimizes prompts for AI agents and agentic systems — system prompts, tool descriptions, thinking configuration, output formatting, and multi-window state management. Model-agnostic with model-specific tips for Claude, GPT, Gemini. Triggers on "write a system prompt," "agent prompt," "prompt engineering," "my agent keeps doing X wrong," "how do I prompt," "system prompt," "tool prompt," "thinking tokens," "effort setting," "agent is too verbose," "won't use tools," "overthinks," "reduce hallucinations," "prompt for agent," "agentic prompt." For tool schema design, see designing-agent-tools. For agent architecture, see designing-agents.
metadata:
  version: 2.0.0
---

# Prompting Agents

Expert guidance for writing prompts that steer AI agents effectively. Model-agnostic principles with model-specific tips where behavior differs.

## Before Writing

Gather this context (ask if not provided):

1. **Agent type** — Single agent or multi-agent? API, CLI-based, or desktop?
2. **Task shape** — What does the agent do? What tools does it have?
3. **Current problems** — What's going wrong? (too verbose, wrong tools, hallucinations, overthinking)
4. **Model** — Which model? Behavior varies significantly across providers.

---

## Core Principles (Model-Agnostic)

### Clarity Over Cleverness

All LLMs respond to clear, explicit instructions. Show your prompt to a colleague with no context — if they'd be confused, the model will be too.

- Be specific about desired output format and constraints
- Use sequential numbered steps when order matters
- Explain *why* behind instructions — models generalize from motivation
- Tell the model what to do, not what not to do

### Context Ordering Matters

Put long documents and data at the top of the prompt, instructions and queries at the bottom. This improves performance across all major models — up to 30% quality improvement on complex, multi-document inputs.

### Examples Are the Most Reliable Lever

3-5 well-crafted input/output examples (few-shot prompting) dramatically improve accuracy and consistency across all models. Make them:
- **Relevant** — mirror actual use cases
- **Diverse** — cover edge cases
- **Structured** — wrap in clear delimiters so models distinguish examples from instructions

### Assign a Role

Setting a role in the system prompt focuses behavior and tone across all models. Even one sentence makes a difference.

---

## System Prompt Structure

Use clear delimiters to separate concerns. XML tags work well across models:

```xml
<role>
You are a [specific role]. Your goal is [specific goal].
</role>

<instructions>
[Core behavioral instructions]
</instructions>

<tools_guidance>
[When and how to use specific tools]
</tools_guidance>

<output_format>
[How to structure responses]
</output_format>

<constraints>
[Safety rails, confirmation requirements, scope limits]
</constraints>
```

Other delimiter options: markdown headers, triple-dash separators, JSON structure. XML works best for complex prompts with nested content.

---

## Steering Tool Use

### Make Agents Act, Not Suggest

Most models default to suggesting when given ambiguous instructions:

```
❌ "Can you suggest some changes to improve this function?"
✅ "Change this function to improve its performance."
```

For default action-taking:

```xml
<default_to_action>
By default, implement changes rather than only suggesting them. If intent is
unclear, infer the most useful action and proceed, using tools to discover
missing details instead of guessing.
</default_to_action>
```

### Parallel Tool Calling

When the model supports parallel tool execution:

```xml
<use_parallel_tool_calls>
If you intend to call multiple tools and there are no dependencies between
the calls, make all independent calls in parallel. Never use placeholders
or guess missing parameters in dependent calls.
</use_parallel_tool_calls>
```

### Calibrate Tool Aggressiveness

More capable models may over-trigger on tool use when given aggressive prompting. Less capable models may need stronger encouragement. Calibrate per model:

- **Over-triggering**: Replace "MUST use this tool" → "Use this tool when..."
- **Under-triggering**: Add "If in doubt whether to use [tool], use it"
- **Wrong tool selection**: Add disambiguation: "For X, use tool_a. For Y, use tool_b instead."

---

## Thinking and Reasoning

### Chain-of-Thought (All Models)

When thinking isn't built in, use structured tags:

```
Think through the problem step by step in <thinking> tags, then provide
your final answer in <answer> tags.
```

Include thinking in few-shot examples to demonstrate the reasoning pattern — models generalize the style.

### Self-Checking

Append for coding and math tasks:

```
Before you finish, verify your answer against [test criteria].
```

### Reduce Overthinking

When a model explores too many options or revisits decisions:

```
When deciding how to approach a problem, choose an approach and commit to it.
Avoid revisiting decisions unless you encounter new information that directly
contradicts your reasoning.
```

For model-specific thinking configuration (Claude adaptive thinking, effort settings): See [references/thinking-config.md](references/thinking-config.md)

---

## Output Control

### Reduce Verbosity

```xml
<concise_output>
Write in clear, flowing prose using complete paragraphs. Reserve markdown
for inline code, code blocks, and simple headings. DO NOT use bullet lists
unless presenting truly discrete items or explicitly requested.
</concise_output>
```

### Control Format

1. **Tell the model what to do** — "Write in flowing prose" beats "Don't use markdown"
2. **Use format indicators** — "Write analysis in `<analysis>` tags"
3. **Match prompt style to desired output** — removing markdown from your prompt reduces markdown in output
4. **Provide examples** — most reliable formatting lever across all models

### Eliminate Preambles

```
Respond directly without preamble. Do not start with phrases like
"Here is...", "Based on...", "Sure, I can...", etc.
```

---

## Long-Running Agent Prompts

### Context Window Management

Tell agents about context compaction if your harness supports it:

```
Your context window will be automatically compacted as it approaches its limit,
allowing you to continue working indefinitely. Do not stop tasks early due to
token budget concerns. Save progress and state before the context refreshes.
```

### Multi-Window State Management

For tasks spanning multiple context windows:

1. **First window** — set up framework (write tests, create setup scripts)
2. **Subsequent windows** — iterate on a todo-list
3. **Structured formats** for state (JSON for tests/status, freeform for progress notes)
4. **Git for checkpoints** — agents can review git log for history
5. **On fresh start**, be prescriptive: "Review progress.txt, tests.json, and git logs"

### Encourage Full Context Usage

```
This is a long task. Plan your work clearly. Spend your entire output context
working on the task — just don't run out of context with uncommitted work.
```

---

## Safety and Autonomy

### Require Confirmation for Risky Actions

```
Consider reversibility and impact of your actions. Take local, reversible
actions freely (editing files, running tests). For hard-to-reverse or
shared-system actions, ask before proceeding:
- Destructive: deleting files/branches, dropping tables, rm -rf
- Hard to reverse: git push --force, git reset --hard
- Visible to others: pushing code, commenting on PRs, sending messages
```

### Reduce Overengineering

```xml
<minimal_changes>
Only make changes directly requested or clearly necessary.
- Don't add features, refactor, or "improve" beyond what was asked
- Don't add docstrings or type annotations to unchanged code
- Don't add error handling for impossible scenarios
- Don't create abstractions for one-time operations
</minimal_changes>
```

### Minimize Hallucinations

```xml
<investigate_before_answering>
Never speculate about code you have not opened. If the user references a
specific file, read it before answering. Investigate relevant files BEFORE
answering questions about the codebase.
</investigate_before_answering>
```

---

## Sub-Agent Orchestration

If the model proactively spawns sub-agents:

```
Use subagents when tasks can run in parallel, require isolated context, or
involve independent workstreams. For simple tasks, sequential operations,
single-file edits, or tasks needing shared context, work directly.
```

---

## Common Fixes

| Problem | Fix |
|---|---|
| Won't use tools | "Implement changes rather than suggesting them" |
| Uses wrong tool | Clarify tool descriptions, add "For X, use Y instead" |
| Too verbose | Add `<concise_output>` block |
| Overthinks | Add "commit to an approach" or lower thinking effort |
| Hallucinations | Add `<investigate_before_answering>` block |
| Overengineers | Add `<minimal_changes>` block |
| Stops early | Add context compaction awareness prompt |
| Too many sub-agents | Add "work directly for simple tasks" guidance |
| Hard-codes test values | "Implement general logic, not test-specific solutions" |
| Creates temp files | "Clean up temporary files at end of task" |

---

## Prompt Design Workflow

```
Prompt Design Progress:
- [ ] Step 1: Identify current failure mode (what's going wrong?)
- [ ] Step 2: Write minimal system prompt addressing the failure
- [ ] Step 3: Test on 3-5 representative prompts
- [ ] Step 4: Add examples if output format is inconsistent
- [ ] Step 5: Add structured delimiters if model misinterprets sections
- [ ] Step 6: Test across models you plan to use
- [ ] Step 7: Calibrate tool aggressiveness per model
- [ ] Step 8: Measure and iterate
```

For copy-paste prompt blocks: See [references/prompt-patterns.md](references/prompt-patterns.md)

---

## Related Skills

- **designing-agent-tools**: For tool schema and description design
- **designing-agents**: For architecture patterns and context engineering
- **building-agent-clis**: For designing CLIs that agents consume
