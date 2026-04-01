# Programmatic Tool Calling (PTC)

## Contents
- How PTC Works
- When to Use PTC vs Standard Tool Calls
- Designing Tools for PTC
- Web Search with PTC
- Trade-offs

## How PTC Works

Standard tool calling: the model emits a tool call, the handler runs it, the result returns to the model's context. Each round-trip costs latency and tokens.

PTC: the model writes code that calls tools as async functions. The code runs in a container. When the code calls a tool (e.g., `await web_search(query)`), the container pauses. The call crosses the sandbox boundary as a typed tool-use event, fulfilled by the same handler (MCP server, custom code, etc.) as a standard tool call. But the result returns to the running code, not the model's context window.

The code processes results following the control flow the model specified — calling more tools, filtering data, accumulating results. Only the final output reaches the model.

### Execution Flow

```
1. Model receives user query
2. Model writes code with tool calls as async functions
3. Code runs in container:
   a. Code calls tool_1(args)  →  container pauses
   b. Handler runs tool_1      →  result returns to code (NOT to model)
   c. Code processes result
   d. Code calls tool_2(args)  →  container pauses
   e. Handler runs tool_2      →  result returns to code
   f. Code filters/aggregates
   g. Code returns final output
4. Final output reaches model's context
5. Model responds to user
```

### Security Model

Tool handlers still sit in the middle of every call. They can:
- **Inspect** — see every tool invocation and its arguments
- **Reject** — refuse calls that violate policy
- **Log** — track latency, token usage, call patterns
- **Queue** — hold for human approval before executing

PTC preserves the full control surface of tools. The code runs in a sandbox; the tools run on your side of the boundary.

## When to Use PTC vs Standard Tool Calls

| Scenario | Use PTC | Use Standard |
|---|---|---|
| Multi-step search (filter many results) | Yes — intermediate results stay in code | No |
| Fetch → parse → filter → aggregate | Yes — pipeline runs without context bloat | No |
| Single tool call with small result | No — overhead isn't worth it | Yes |
| User needs to see intermediate results | No — results don't reach model context | Yes |
| Actions requiring human approval at each step | No — PTC batches execution | Yes |
| Workflows with large intermediate data | Yes — biggest token savings | No |

**Rule of thumb:** If intermediate results are large but the final output is small, PTC saves tokens. If every intermediate result matters to the user, use standard tool calls.

## Designing Tools for PTC

Tools used with PTC should be designed the same way as standard tools — same schemas, descriptions, and handlers. PTC is a calling convention, not a different tool format.

However, some design choices help PTC work better:

### Return Structured Data

PTC code processes tool results programmatically. Structured responses (JSON) are easier to filter and aggregate in code than free-text markdown.

```json
// Good for PTC — code can filter by relevance_score
{
  "results": [
    {"title": "...", "snippet": "...", "url": "...", "relevance_score": 0.95},
    {"title": "...", "snippet": "...", "url": "...", "relevance_score": 0.32}
  ]
}
```

### Keep Tool Granularity Small

PTC handles composition in code, so individual tools can stay atomic. Unlike standard tool calling where you might merge tools to reduce round-trips, PTC benefits from small, composable tools because the composition cost is near zero.

### Document Expected Output Shape

Since PTC code parses tool results, the output schema needs to be predictable. Document the response shape in the tool description so the model writes correct parsing code.

## Web Search with PTC

Web search is the canonical PTC use case. The built-in web search tool uses PTC by default:

```json
{
  "model": "claude-opus-4-6",
  "max_tokens": 4096,
  "tools": [
    {
      "type": "web_search_20260209",
      "name": "web_search"
    }
  ],
  "messages": [
    {"role": "user", "content": "What are the latest developments in..."}
  ]
}
```

**Why search benefits most from PTC:**
- Search returns many results, most of which are irrelevant
- The model needs to cross-reference across results
- Filtering and ranking are naturally expressed in code
- Only the synthesized answer needs to reach context

**Results on search benchmarks:**
- ~11% accuracy improvement (BrowseComp, DeepsearchQA)
- ~24% fewer input tokens
- #1 on LMArena Search Arena (Opus 4.6 with PTC)

## Trade-offs

### Benefits
- **Token savings** — intermediate results don't consume context
- **Performance** — model reasons over curated output, not raw data
- **Speed** — fewer round-trips through the model
- **Composability** — code can express complex orchestration naturally

### Costs
- **Opacity** — intermediate results aren't visible in the conversation
- **Debugging** — harder to inspect what happened inside the container
- **Container overhead** — code execution adds latency for simple tasks
- **Model capability** — requires models that reliably write correct tool-calling code

### When NOT to Use PTC

- Single tool calls with small results
- Workflows where the user needs visibility into each step
- Actions that each require human approval
- Tasks where the model needs to reason between each tool call (not just process data)
