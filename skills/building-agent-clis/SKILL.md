---
name: building-agent-clis
description: Designs and builds CLIs optimized for AI agent consumption — machine-readable output, schema introspection, input hardening, context window discipline, and safety rails. Triggers on "CLI for agents," "agent CLI," "agent-first CLI," "make my CLI agent-friendly," "MCP server from CLI," "machine-readable output," "agent DX," "CLI tool for AI," "rewrite CLI for agents," "agents keep breaking my CLI," "input validation for agents." For tool schema design, see designing-agent-tools. For prompting, see prompting-agents.
metadata:
  version: 1.0.0
---

# Building Agent CLIs

Expert guidance for designing CLIs that AI agents can use safely and effectively.

## Core Insight

Human DX optimizes for discoverability and forgiveness. Agent DX optimizes for predictability and defense-in-depth. These are different enough that retrofitting a human-first CLI for agents is often a losing bet.

**The agent is not a trusted operator.** You wouldn't build a web API that trusts user input without validation. Don't build a CLI that trusts agent input either.

## Before Building

Gather this context (ask if not provided):

1. **Current CLI state** — Existing CLI to retrofit, or building from scratch?
2. **Underlying system** — REST API, database, filesystem, or something else?
3. **Agent consumers** — Which agents will call this? (Claude Code, Gemini, custom agents)
4. **Risk profile** — What's the blast radius of a wrong command? (read-only vs destructive)

---

## Design Principles

### Raw Payloads Over Bespoke Flags

Agents generate JSON trivially. They struggle with complex flag combinations.

```bash
# Human-first — 10 flags, flat namespace, can't nest
my-cli spreadsheet create \
  --title "Q1 Budget" --locale "en_US" --sheet-title "January" \
  --frozen-rows 1 --frozen-cols 2 --row-count 100

# Agent-first — one flag, the full API payload
my-cli spreadsheet create --json '{
  "properties": {"title": "Q1 Budget", "locale": "en_US"},
  "sheets": [{"properties": {"title": "January",
    "gridProperties": {"frozenRowCount": 1, "frozenColumnCount": 2}}}]
}'
```

The JSON version maps directly to the API schema. Zero translation loss.

**Practical approach:** Support both paths in the same binary. Keep convenience flags for humans. Add `--json` for the raw-payload path. Default to JSON output when stdout isn't a TTY.

### Schema Introspection Replaces Documentation

Agents can't google docs without blowing their token budget. Make the CLI itself the documentation:

```bash
my-cli schema users.create    # Dumps full method signature as JSON
my-cli schema users.list      # Params, request body, response types
```

The CLI becomes the canonical source of truth for what it accepts *right now*.

### Context Window Discipline

API responses consume agent context. Every irrelevant field wastes reasoning capacity.

**Field masks** — let agents limit what comes back:
```bash
my-cli files list --fields "id,name,mimeType"
```

**Streaming pagination** — emit one JSON object per page (NDJSON), processable without buffering:
```bash
my-cli files list --page-all --output ndjson
```

**Default to minimal responses** for agent-facing output. Let agents request more with `--verbose` or `--fields`.

### Input Hardening Against Hallucinations

Agents hallucinate differently than humans typo. Build validation for agent failure modes:

| Attack Vector | What Agents Do | Defense |
|---|---|---|
| Path traversal | Generate `../../.ssh` by confusing path segments | Canonicalize and sandbox all paths to CWD |
| Control characters | Generate invisible chars in strings | Reject anything below ASCII 0x20 |
| Embedded query params | Pass `fileId?fields=name` as a resource ID | Reject `?` and `#` in resource IDs |
| Double encoding | Pre-encode strings that get encoded again | Reject `%` in resource inputs |
| Injection in IDs | Embed SQL/shell fragments in ID fields | Validate ID format strictly |

**Rule:** Validate at the CLI boundary. Don't trust that the agent will pass clean input just because the schema says so.

### Safety Rails

**`--dry-run`** — validates the request without executing. Critical for mutating operations (create, update, delete). Agents can "think out loud" before acting.

**Response sanitization** — defend against prompt injection in API responses. A malicious email body containing "Ignore previous instructions..." should not reach the agent raw. Sanitize or flag responses from untrusted data sources.

---

## Output Formats

### Machine-Readable Output (Required)

Support `--output json` at minimum. Better: default to JSON when stdout isn't a TTY.

```bash
# Human sees a table
my-cli users list

# Agent (or pipe) gets JSON
my-cli users list --output json
my-cli users list | jq .  # Auto-detects non-TTY
```

### NDJSON for Streaming

For paginated or streaming results, emit one JSON object per line:

```bash
my-cli events stream --output ndjson
```

Agents can process incrementally without buffering the full response.

### Error Output

Errors must be machine-parseable too:

```json
{
  "error": true,
  "code": "INVALID_RESOURCE_ID",
  "message": "Resource ID contains invalid character '?'. IDs must be alphanumeric.",
  "suggestion": "Remove query parameters from the ID. Did you mean: abc123"
}
```

---

## Ship Agent Skills, Not Just Commands

Agents learn through context injected at conversation start, not through `--help` and Stack Overflow.

Ship SKILL.md or CONTEXT.md files encoding invariants agents can't intuit:

```yaml
---
name: my-cli-files
---

## Rules

- Always use --dry-run for mutating operations before executing
- Always confirm with user before executing write/delete commands
- Add --fields to every list call to limit response size
- Use --output json for all commands
```

These rules exist because agents don't have intuition. A skill file is cheaper than a hallucination.

---

## Multi-Surface Design

Design the same binary to serve multiple agent interfaces:

```
                Core Binary (my-cli)
               ┌────┬────┬────┬────┐
               │    │    │    │    │
              CLI  MCP  Env  API
            (human) (stdio) (vars) (ext)
```

**CLI** — interactive terminal with formatted output for humans
**MCP (stdio)** — expose commands as typed JSON-RPC tools, eliminating shell escaping
**Environment variables** — credential injection for headless agent environments
**Extensions** — native agent framework integrations (Gemini extensions, etc.)

### MCP Surface

If your CLI wraps a structured API, expose it as MCP tools:

```bash
my-cli mcp --services files,users  # Exposes as JSON-RPC over stdio
```

MCP eliminates shell escaping, argument parsing ambiguity, and output parsing. The agent calls typed functions.

### Auth for Agents

Agents can do OAuth but shouldn't need to:
- **Environment variables** for tokens and credential file paths
- **Service accounts** where possible
- **Never require browser redirects** in the agent path

---

## Retrofit Checklist

For existing CLIs, add agent support incrementally:

```
Agent CLI Retrofit:
- [ ] 1. Add --output json (table stakes)
- [ ] 2. Validate all inputs (reject control chars, path traversals, embedded params)
- [ ] 3. Add schema/--describe command (runtime introspection)
- [ ] 4. Support --fields or field masks (limit response size)
- [ ] 5. Add --dry-run (validate before mutating)
- [ ] 6. Ship CONTEXT.md or SKILL.md (encode agent invariants)
- [ ] 7. Expose MCP surface (if wrapping an API)
- [ ] 8. Add env var auth path (no browser required)
```

---

## Anti-Patterns

- **Human-only output** — pretty tables with no JSON mode. Agents can't parse ANSI.
- **Interactive prompts** — `Are you sure? [y/N]` blocks agents. Use `--yes` or `--force` flags.
- **Trusting agent input** — agents hallucinate paths, IDs, and parameters. Validate everything.
- **Giant responses** — dumping full API responses without field filtering. Wastes context.
- **Auth that requires a browser** — agents run headless. Provide env var / service account paths.
- **No dry-run for mutations** — agents can't undo `DELETE`. Let them preview first.
- **Docs-only introspection** — "read the docs" doesn't work when docs cost tokens. Build runtime schema queries.

---

## Related Skills

- **designing-agent-tools**: For tool schema and description design
- **prompting-agents**: For system prompts and behavioral steering
- **designing-agents**: For architecture patterns and context engineering
