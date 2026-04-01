# Tool Evaluation Guide

## Contents
- Evaluation Architecture
- Task Design
- Verifier Design
- Running Evaluations
- Analysis Playbook
- Iterative Improvement Loop

## Evaluation Architecture

```
For each task:
  1. Agent loop: LLM call → tool calls → feed results → repeat until done
  2. Run verifier on final output
  3. Collect metrics: accuracy, tool calls, errors, tokens, latency
```

Each task runs in an isolated agent loop with a system prompt, single task, and tool access.

## Task Design

### Structure

```json
{
  "skills": ["designing-agent-tools"],
  "query": "The user prompt that triggers the workflow",
  "files": ["optional - files needed for the task"],
  "expected_behavior": [
    "Specific checkable claims about the output",
    "Each should be independently verifiable"
  ]
}
```

### Strong vs Weak Tasks

Strong tasks require judgment and multiple tool calls. Weak tasks spell out the answer.

### Coverage

Ensure tasks cover: happy path, multi-step (3-10+ calls), ambiguity, edge cases, scope boundaries, error recovery, and token pressure (large result sets).

### Train/Test Split

- **Training (70%)** — identify and fix issues
- **Held-out test (30%)** — measure final performance only

Never optimize on test set results.

## Verifier Design

**Exact match** — for unambiguous factual outputs:
```python
assert response["customer_name"] == "Sarah Chen"
```

**LLM-as-judge** — for nuanced evaluation:
```python
judge_prompt = f"""Given task: {task}, response: {response}, criteria: {assertions}
Score each criterion PASS/FAIL with explanation."""
```

**Tool call verification** — check right tools called:
```python
assert "search_logs" in called_tools
assert called_tools.count("search_logs") <= 3
```

**Pitfalls:** Too strict (rejects valid phrasings), too lenient (passes keyword matches that miss the point), format-dependent (fails correct answers over JSON vs markdown).

## Running Evaluations

### Agent Loop

```python
def run_eval_task(task, tools, model="claude-sonnet-4-20250514"):
    messages = [{"role": "user", "content": task["query"]}]
    tool_calls_log = []
    total_tokens = 0

    while True:
        response = client.messages.create(
            model=model, system=system_prompt,
            messages=messages, tools=tools, max_tokens=4096
        )
        total_tokens += response.usage.input_tokens + response.usage.output_tokens
        tool_use_blocks = [b for b in response.content if b.type == "tool_use"]

        if not tool_use_blocks:
            break

        tool_results = []
        for block in tool_use_blocks:
            result = execute_tool(block.name, block.input)
            tool_calls_log.append({"tool": block.name, "input": block.input, "output": result})
            tool_results.append({"type": "tool_result", "tool_use_id": block.id, "content": result})

        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

    return {"response": extract_text(response), "tool_calls": tool_calls_log, "total_tokens": total_tokens}
```

### Eval System Prompt

Instruct agents to output `<reasoning>`, `<feedback>`, and `<response>` blocks. The feedback surfaces tool design issues. Enable extended thinking for deeper insight.

## Analysis Playbook

### Triage Failures by Mode

| Failure Mode | Fix |
|---|---|
| Wrong tool called | Clarify descriptions, add "use X instead" |
| Right tool, wrong params | Rename params, add examples, use enums |
| Correct calls, wrong conclusion | Restructure response format |
| Too many tool calls | Consolidate tools, improve pagination |
| Error cascade | Improve error messages with recovery guidance |

### Read Agent CoT

Look for: why they chose a tool, what they expected vs got, where they backtracked, what they wished existed.

### Read Between the Lines

- Tools never called → unclear purpose
- Params always defaulted → possibly unnecessary
- Success without certain tools → redundancy signal
- Same param error across tasks → schema problem
- Agents appending "2025" to queries → description should steer away

## Iterative Improvement Loop

```
1. Run evaluation → baseline
2. Analyze failures
3. Make ONE category of change (descriptions, schema, response format, or tool boundaries)
4. Re-evaluate → compare
5. Validate on held-out test set
6. Repeat until held-out accuracy plateaus across 3+ iterations
```

Isolate changes to attribute improvements. Stop when remaining failures are model limitations, not tool design.
