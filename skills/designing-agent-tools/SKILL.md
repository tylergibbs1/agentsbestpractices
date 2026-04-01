---
name: designing-agent-tools
description: Designs, builds, and optimizes tools for AI agents — covering schema design, descriptions, response formatting, MCP servers, evaluation, and iterative improvement. Triggers on "build a tool," "write a tool," "MCP tool," "agent tool," "tool for Claude," "tool design," "improve my tools," "tool evaluation," "tool description," "my agent keeps calling the wrong tool," "agent can't use this tool," building MCP servers, Desktop extensions (DXT), or API tool definitions.
metadata:
  version: 2.0.0
---

# Designing Agent Tools

Expert guidance for building tools that AI agents use effectively.

## Before Building

Gather this context (ask if not provided):

1. **Agent context** — What agent uses these tools? What tasks? What other tools exist?
2. **Underlying resources** — What APIs/databases/services do the tools wrap? Auth required?
3. **User workflows** — What real-world prompts will users give? What multi-step workflows?

---

## Core Principles

### Build for Agent Affordances

Agents have limited context, process tokens sequentially, and reason in natural language:

- Don't build `list_all_X`. Build `search_X` with filters.
- Return human-readable names, not opaque UUIDs.
- Every token in a response competes with reasoning space.

### Fewer, Deeper Tools

Each tool added increases wrong-tool risk. Consolidate:

| Instead of | Build |
|---|---|
| `list_users` + `list_events` + `create_event` | `schedule_event` (finds availability and schedules) |
| `read_logs` | `search_logs` (returns relevant lines with context) |
| `get_customer_by_id` + `list_transactions` + `list_notes` | `get_customer_context` (compiles all relevant info) |

**Test:** If an agent chains 3+ tools for a common task, merge them.

### Make Invalid Calls Unrepresentable

Use enums, not strings. Name `user_id`, not `user`. Mark required params required.

---

## Schema Design

**Parameters:**
- Name unambiguously: `user_id` not `user`, `channel_name` not `channel`
- Use enums for constrained values
- Set sensible defaults for optional params

**Descriptions** — write as if briefing a new team member:
- Make implicit context explicit (query formats, terminology, resource relationships)
- Include 1-2 examples of valid inputs
- State what the tool does NOT do if boundary is non-obvious

For detailed description guidance: See [references/tool-descriptions.md](references/tool-descriptions.md)

---

## Response Design

**High-signal only:**
- Natural language names alongside any technical IDs
- Contextual relevance over raw completeness

**Control size:**
- Pagination with sensible defaults (10-25 items)
- `response_format` enum: `"concise"` (content only) vs `"detailed"` (includes IDs, metadata)
- 25,000 token ceiling on responses
- Never silently truncate — append: "Showing 10 of 47. Use `offset` or narrow `query` for more."

**Errors must be actionable:**
```
❌ "Error: Invalid parameter"
✅ "Error: `start_date` must be ISO 8601 (e.g., '2025-03-15T00:00:00Z'). Received: 'March 15'"
```

---

## Namespacing

When agents access many tools, namespace by service and resource:

- **Service:** `slack_search`, `github_search`, `asana_search`
- **Resource:** `asana_projects_search`, `asana_tasks_search`

Test prefix vs suffix naming — effect on agent performance varies by model.

---

## MCP Server Quick Start

1. Scaffold with the MCP SDK for your language
2. Connect locally:
   - **Claude Code:** `claude mcp add <name> <command> [args...]`
   - **Claude Desktop:** Settings > Developer
3. Add tool annotations: `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`
4. For distribution via Claude Desktop, package as a DXT

---

## Tool Design Workflow

Copy this checklist and track progress:

```
Tool Design Progress:
- [ ] Step 1: Map the workflow (identify what tasks the tools enable)
- [ ] Step 2: Choose tool boundaries (consolidate where possible)
- [ ] Step 3: Define schemas (names, types, enums, descriptions)
- [ ] Step 4: Design responses (format, pagination, error messages)
- [ ] Step 5: Implement and connect locally
- [ ] Step 6: Test manually — find rough edges
- [ ] Step 7: Write evaluation tasks
- [ ] Step 8: Run evaluation and analyze results
- [ ] Step 9: Improve based on agent feedback → re-evaluate
```

**Step 1–2:** Think about the workflow the way a human would subdivide it. Tools should match task decomposition, not API structure.

**Step 3–4:** For each tool, provide a complete spec:
- Name, description, parameters (types, constraints, examples)
- Response schema with success and error examples

**Step 5–6:** Connect to Claude Code or Claude Desktop. Try the tools yourself on real prompts.

**Step 7–9:** See [references/tool-evaluation.md](references/tool-evaluation.md) for evaluation setup.

---

## Evaluation

### Strong vs Weak Tasks

| Strong | Weak |
|---|---|
| "Schedule a meeting with Jane next week to discuss Acme Corp. Attach last planning notes and reserve a room." | "Schedule a meeting with jane@acme.corp." |
| "Customer 9182 reported triple charges. Find logs and check if others are affected." | "Search payment logs for customer_id=9182." |

Strong tasks require judgment and multiple tool calls. Weak tasks spell out the answer.

### Key Metrics

| Metric | Signal |
|---|---|
| Task accuracy | Tools working correctly? |
| Tool call count | Efficient paths? |
| Tool errors | Schemas confusing? |
| Token consumption | Responses bloating context? |

### Iterative Improvement

1. Concatenate evaluation transcripts → feed to Claude
2. Identify patterns → refine tools
3. Re-evaluate with held-out test set
4. Repeat until performance plateaus

For detailed evaluation setup, metrics, and analysis: See [references/tool-evaluation.md](references/tool-evaluation.md)

---

## Anti-Patterns

- **1:1 API wrapping** — Agents need workflow-level tools, not CRUD endpoints
- **Overlapping tools** — If two tools handle the same prompt, agents oscillate. Merge or differentiate.
- **Returning everything** — `list_contacts` with 10k results destroys context. Use `search_contacts`.
- **Opaque IDs only** — `"a1b2c3d4-e5f6"` is meaningless. Include `"Jane Smith"` alongside.
- **Silent failures** — Empty results with no explanation. Always say what happened.
- **Generic param names** — `query`, `input`, `data` give no clue. Be specific.

---

## Output Format

When designing tools, deliver:

1. **Tool specification** — name, description, parameters, response schema with examples
2. **Implementation** — working code with validation and pagination
3. **Evaluation tasks** — 5-7 realistic prompts covering happy path, multi-step, edge cases, scope boundaries
4. **Integration instructions** — how to connect (MCP add, API config, env vars)
