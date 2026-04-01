# Tool Description Writing Guide

## Contents
- Description Anatomy
- Parameter Naming Rules
- Before and After Examples
- Common Mistakes

## Description Anatomy

Four parts, in order:

### 1. What It Does (1-2 sentences)
Lead with the action.
```
✅ "Search Slack messages across channels by keyword, sender, or date range."
❌ "This tool can be used to search for messages in Slack."
```

### 2. When to Use It (1-2 sentences)
Disambiguate from similar tools.
```
✅ "Use for finding messages. For channel management, use slack_channels_manage."
❌ "Use when you need to search Slack."
```

### 3. How It Works (if non-obvious)
Query formats, implicit behaviors, surprising defaults.
```
✅ "Searches text by default. Use `from` for sender. Sorted by relevance — use `sort: 'recent'` to change."
```

### 4. Constraints and Limits
```
✅ "Returns up to 20 results. Use `cursor` for pagination. Max query: 500 chars."
```

## Parameter Naming Rules

| Bad | Good | Why |
|-----|------|-----|
| `query` | `search_query` | Disambiguates |
| `user` | `user_email` / `user_id` | Format is clear |
| `type` | `message_type` | Won't collide |
| `id` | `channel_id` | Resource is clear |
| `data` | `event_payload` | Shape is clear |
| `n` | `max_results` | Self-documenting |

Always use enums for fixed value sets.

Include examples for non-obvious inputs:
```
"Query supports Slack syntax:
- Simple: 'quarterly report'
- From user: 'from:@jane budget'
- In channel: 'in:#engineering deploy'"
```

### Response Format Enum

Support `concise` (content only, ~72 tokens) vs `detailed` (includes IDs/metadata, ~206 tokens) to let agents control verbosity.

## Before and After Examples

### Calendar Tool

**Before:**
```
name: create_event
description: Creates a calendar event
parameters:
  - title (string): Event title
  - time (string): When the event is
  - attendees (string): Who to invite
```

**After:**
```
name: schedule_meeting
description: >
  Schedule a meeting by finding mutual availability and creating a calendar event.
  Checks all attendees' calendars for conflicts. Use instead of manually checking
  availability then creating an event.
parameters:
  - title (string, required): Example: "Q1 Planning Review"
  - preferred_window (object, required):
      earliest (string): ISO 8601. Example: "2025-03-17T09:00:00Z"
      latest (string): ISO 8601. Example: "2025-03-21T17:00:00Z"
  - duration_minutes (integer, required): Default: 30
  - attendee_emails (string[], required): Example: ["jane@co.com", "bob@co.com"]
  - room_required (boolean): Default: false
```

### Search Tool

**Before:**
```
name: search
description: Searches the knowledge base
parameters:
  - q (string): Search query
```

**After:**
```
name: search_knowledge_base
description: >
  Search internal docs, runbooks, and policies by keyword or question.
  Returns relevant passages with source links. For code, use search_codebase.
  For people, use search_directory.
parameters:
  - search_query (string, required):
      Examples: "refund policy", "PCI compliance", "on-call rotation"
  - category (enum: [policies, runbooks, engineering, product, all]): Default: "all"
  - max_results (integer): Default: 5, max: 20
  - response_format (enum: [concise, detailed]): Default: "concise"
```

## Common Mistakes

- **Describing implementation, not behavior:** "Makes a GET to /api/v2/search" — agents don't care about HTTP methods
- **Assuming agent context:** "Uses standard query syntax" — standard to whom?
- **Omitting scope boundaries:** State what the tool CAN'T do to prevent impossible attempts
- **Vague errors:** "Error: channel not found" → include how to fix: "check spelling, use channel_id, or call slack_channels_list"
