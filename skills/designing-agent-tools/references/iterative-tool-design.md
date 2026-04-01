# Iterative Tool Design

Case studies from building Claude Code, illustrating how tools evolve through observation and experimentation.

## Contents
- The AskUserQuestion Tool (3 iterations)
- TodoWrite → Tasks (evolving with model capabilities)
- Search Interface Evolution (RAG → Grep → progressive disclosure)
- The Claude Code Guide Agent (expanding action space without tools)

## The AskUserQuestion Tool

**Goal:** Improve elicitation — help the agent ask users better questions with lower friction.

### Attempt 1: Parameter on ExitPlanTool

Added a `questions` array parameter to the existing plan tool. The agent would output a plan and questions simultaneously.

**Problem:** Confused the model. A plan and questions about the plan are contradictory — what if answers conflict with the plan? Would the agent need to call the tool twice?

**Lesson:** Don't overload a tool with two competing purposes.

### Attempt 2: Custom Output Format

Modified output instructions to have the agent emit questions in a special markdown format (bullet points with bracketed alternatives). Parsed this format to render UI.

**Problem:** The agent was inconsistent — appended extra sentences, omitted options, used different formats. Free-form output parsing is fragile.

**Lesson:** Structured tool output is more reliable than parsing free-form text, even with good instructions.

### Attempt 3: Dedicated Tool (final)

Created `AskUserQuestion` as a standalone tool callable at any point. When triggered, it showed a modal and blocked the agent loop until the user answered.

**Why it worked:**
- Structured output: the tool schema enforced the question format
- Clear trigger: the agent understood when to call it
- Composable: could be referenced in skills, used in the Agent SDK
- The model "liked" calling it — outputs were consistently good

**Lesson:** The best-designed tool doesn't work if the model doesn't understand how to call it. Test whether the model naturally reaches for the tool.

## TodoWrite → Tasks

### Phase 1: TodoWrite

Early models needed a todo list to stay on track. `TodoWrite` let the agent create and check off items, displayed to the user.

**Problem:** The agent still forgot what it had to do. Added system reminders every 5 turns to re-inject the todo list.

### Phase 2: System Reminders

Injecting the todo list periodically kept the agent on track.

**Problem:** As models improved, the reminders became counterproductive. The agent interpreted reminders as "you must stick to this list" instead of "here's your current plan." It stopped adapting.

### Phase 3: Tasks (current)

Replaced TodoWrite with a Task tool designed for agent-to-agent communication:
- Tasks have dependencies
- Sub-agents can share updates
- The agent can alter and delete tasks
- Designed for coordination, not just tracking

**Lesson:** Tools that helped weaker models may constrain stronger ones. As model capabilities increase, revisit whether a tool is still helping or limiting. Stick to a small set of supported models with similar capability profiles so you're not maintaining tools for different ability levels.

## Search Interface Evolution

### Phase 1: RAG Vector Database

Claude Code initially used a vector database to find relevant context, injecting it into the prompt.

**Problems:**
- Required indexing and setup
- Fragile across different environments
- Context was given to the agent, not found by the agent

### Phase 2: Grep Tool

Replaced RAG with a Grep tool. Let the agent search the codebase itself.

**Key insight:** As models get smarter, they become increasingly good at building their own context if given the right search tools. An agent that finds its own context understands it better than one that receives pre-selected context.

### Phase 3: Progressive Disclosure via Skills

Formalized progressive disclosure through Agent Skills. The agent reads SKILL.md files, which reference other files, which reference more files. The agent recursively navigates to find exactly the context it needs.

**Common pattern:** Skills add search capabilities — instructions for querying APIs, databases, or documentation systems. The agent learns to use these on demand rather than having everything pre-loaded.

**Evolution:** Over one year, went from "can't really build its own context" to "nested search across several layers of files to find exactly what's needed."

**Lesson:** Give agents search tools and let them build context, rather than injecting context for them. Progressive disclosure scales better than pre-loading.

## The Claude Code Guide Agent

**Problem:** Claude didn't know enough about its own features (MCP setup, slash commands, etc.). Users would ask "how do I add an MCP?" and get wrong answers.

### Option A: System Prompt (rejected)

Could put all documentation in the system prompt. But users rarely asked these questions, so the docs would:
- Add context rot (constant token cost for rare queries)
- Interfere with the agent's primary job (writing code)

### Option B: Link to Docs (partially worked)

Gave the agent a link to documentation it could load and search. The agent loaded it but pulled too many results into context to find the answer.

### Option C: Dedicated Sub-Agent (final)

Created a "Guide" sub-agent that the main agent is prompted to call when users ask about Claude Code itself. The sub-agent has:
- Specialized instructions for searching docs effectively
- Guidance on what to return (just the answer, not everything)

**Why it worked:** Expanded the action space without adding a tool to the main agent. The main agent's tool list stayed small — it just delegates to a sub-agent for this specific domain.

**Lesson:** Sub-agents with specialized instructions are often better than new tools. They keep the main agent's action space clean while adding capabilities through progressive disclosure.

## Summary of Principles

1. **Don't overload tools** — one purpose per tool
2. **Structured output beats parsed free-form** — use tool schemas to enforce format
3. **Test if the model "likes" the tool** — a well-designed tool the model doesn't reach for is useless
4. **Revisit tools with each model upgrade** — helpful tools become constraining tools
5. **Let agents build their own context** — search tools over injected context
6. **Expand action space without adding tools** — progressive disclosure, sub-agents, code execution
7. **Keep the tool count small** — each tool is one more option the model must evaluate
