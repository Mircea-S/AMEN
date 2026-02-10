# AMEN — Agent Memoization for Exploration Nodes

A simple technique to stop Claude Code from rediscovering your codebase every conversation. One JSON file, a few lines in `CLAUDE.md`, and your Explore agents become a persistent knowledge layer.

## The Problem

Claude Code's Explore agents are powerful: they search, read, and synthesize information about your codebase. But they start from zero every conversation. 
Ask about your auth system on Monday, then again on Thursday, and you'll burn the same tokens tracing the same files twice.

There's no built-in memory for agent results. So I built one. Say AMEN.

## How It Works

```
Conversation 1                          Conversation 2
┌─────────────────────┐                 ┌─────────────────────┐
│ "How does auth work?"│                │ "Add OAuth support"  │
│                     │                 │                     │
│ Launch Explore agent│                 │ Check cache → HIT   │
│ Agent reads 12 files│                 │ Skip agent entirely │
│ Returns findings    │                 │ Use cached summary  │
│                     │                 │ + file list directly│
│ Distill → cache     │                 │                     │
└────────┬────────────┘                 └─────────────────────┘
         │                                        ▲
         ▼                                        │
    .agent-cache.json ────────────────────────────┘
```

1. Before launching an Explore agent, Claude checks the cache file for a matching entry
2. If the entry exists and is fresh, the agent is skipped entirely
3. If the entry is missing or stale, the agent runs normally, then its findings are distilled and saved
4. Reusable implementation patterns are saved separately with no expiry

## Token Savings

Here's what AMEN actually saves.

### Cost of one Explore agent (no cache)

An Explore subagent typically runs 5–8 turns, each accumulating context: file contents, search results, reasoning. Because context grows with every turn, total token consumption across all turns sums to roughly:

- **Input tokens**: ~50,000–100,000 (context accumulates across turns)
- **Output tokens**: ~5,000–10,000 (summaries, reasoning, tool calls)

### Cost of a cache hit

One Read call to `.agent-cache.json`, pulling only the relevant key:

- **Total tokens**: ~200–500

That's a **~99% reduction** per cache hit.

Not too mention the speed aspect.

## Setup

### 1. Create the cache file

Create `.agent-cache.json` in your project root:

```json
{
  "_meta": { "maxAgeDays": 7 },
  "topics": {},
  "patterns": {}
}
```

### 2. Gitignore it

```
# .gitignore
.agent-cache.json
```

The cache is local and machine-specific. It should never be committed.

### 3. Add instructions to CLAUDE.md

Paste this section into your project's `CLAUDE.md` (the file Claude Code reads at the start of every conversation). Place it near the top, after any project-level conventions:

```markdown
## AMEN — Agent Memoization for Exploration Nodes

`.agent-cache.json` stores distilled results from Explore agents across conversations.

**Protocol — follow on every task that would launch an Explore agent:**
1. **Read before launching.** Check `.agent-cache.json` for a matching topic key. Read only the key you need — never preload the entire file into context.
2. **Check freshness.** Compare the entry's `ts` to today. If older than `_meta.maxAgeDays` (default 7), treat as missing.
3. **Skip the agent if cache hits.** Use the cached `summary` and `files` list directly.
4. **CRITICAL — Write back IMMEDIATELY.** When an Explore agent returns results, your VERY NEXT action MUST be writing to `.agent-cache.json`. Do NOT use the results, do NOT continue the task, do NOT write to the plan file — write the cache entry FIRST. This is a blocking prerequisite before any other action. Distill findings into:
   - `ts` — ISO 8601 timestamp
   - `summary` — 2-4 sentences: what exists, where, key function names. No code blocks.
   - `files` — array of key file paths discovered
5. **Save reusable patterns.** If the agent uncovered a recurring implementation pattern (e.g., how to add a new DB event hook, how settings sections are structured), add it to the top-level `patterns` object as a one-liner keyed by slug.
6. **Prune on write.** When writing to the cache, remove any topic entries older than 30 days.

**Rules:**
- Topic keys are kebab-case slugs (e.g., `mission-system`, `push-notifications`, `settings-views`)
- Summaries must be plain text, no markdown/code fences — keep under 300 chars
- The `patterns` object is long-lived (no TTL) — only update when a pattern changes
- Never cache user-specific data or secrets
```

That's it. No tooling, no dependencies, no build step.

### 4. Add instructions to the projects' MEMORY.md file

```markdown
## AMEN Cache — Mandatory Write-Back
- After ANY Explore agent returns results, your **very next action** MUST be writing to `.agent-cache.json`. Do not use the results, continue the task, or write to a plan file first. Cache write is a blocking prerequisite.
- This applies in all modes: plan mode, normal mode, background agents.
- Format: `{ ts, summary (plain text, <300 chars), files: [...] }` under `topics.<kebab-key>`.
```

## Cache File Reference

### Structure

```json
{
  "_meta": { "maxAgeDays": 7 },
  "topics": {
    "topic-slug": {
      "ts": "2026-02-08T14:30:00Z",
      "summary": "Plain text summary of what the agent found. 2-4 sentences, under 300 chars.",
      "files": ["src/auth.ts", "src/middleware/session.ts"]
    }
  },
  "patterns": {
    "pattern-slug": "One-liner describing a recurring implementation pattern."
  }
}
```

### Fields

| Field | Type | Description |
|---|---|---|
| `_meta.maxAgeDays` | number | Topic entries older than this are treated as stale. Default `7`. |
| `topics.<key>.ts` | string | ISO 8601 timestamp of when the entry was written. |
| `topics.<key>.summary` | string | Distilled agent findings. Plain text, no code fences, under 300 chars. |
| `topics.<key>.files` | string[] | Key file paths the agent identified as relevant. |
| `patterns.<key>` | string | Reusable implementation pattern as a one-liner. No TTL — updated only when the pattern changes. |

### Example (populated)

```json
{
  "_meta": { "maxAgeDays": 7 },
  "topics": {
    "auth-system": {
      "ts": "2026-02-08T14:30:00Z",
      "summary": "JWT auth in src/auth.ts using jose library. Login/register routes in src/routes/auth.ts. Session stored as httpOnly cookie named 'token'. Middleware in src/middleware/auth.ts extracts and verifies JWT on every request.",
      "files": ["src/auth.ts", "src/routes/auth.ts", "src/middleware/auth.ts"]
    },
    "database-schema": {
      "ts": "2026-02-07T09:15:00Z",
      "summary": "PostgreSQL with Drizzle ORM. Schema defined in src/db/schema.ts. Migrations in migrations/ dir managed by drizzle-kit. Key tables: users, posts, comments, tags. Relations defined inline with Drizzle's relations() helper.",
      "files": ["src/db/schema.ts", "src/db/index.ts", "drizzle.config.ts"]
    }
  },
  "patterns": {
    "new-api-route": "Create handler in src/routes/<name>.ts, add to router in src/routes/index.ts, add validation schema in src/validation/<name>.ts.",
    "db-migration": "Run 'npx drizzle-kit generate' after schema change, then 'npx drizzle-kit migrate' to apply."
  }
}
```

## Tuning

### Freshness window

The default `maxAgeDays: 7` works well for active projects. Adjust based on how fast your codebase changes:

| Pace | Suggested `maxAgeDays` |
|---|---|
| Rapid iteration (multiple changes/day) | 3 |
| Normal development | 7 |
| Stable/maintenance mode | 14–30 |

### What makes a good topic key

Topic keys should map to the kind of question you'd ask an Explore agent:

- `auth-system` — "How does authentication work?"
- `api-routes` — "Where are the API endpoints defined?"
- `database-schema` — "What's the database structure?"
- `deployment-pipeline` — "How is the app deployed?"

Bad keys: `bug-fix-123` (too specific), `code` (too broad), `refactoring-plan` (not a codebase fact).

### What makes a good pattern

Patterns capture the *recipe* for recurring implementation tasks:

- **Good**: `"Add route in src/routes/<name>.ts, register in src/routes/index.ts, add middleware in src/middleware/<name>.ts."`
- **Bad**: `"The routes are in the routes folder."` (too vague — that's a topic summary, not a pattern)

The difference: a **topic** describes what exists, a **pattern** describes how to add something new.

## How It Behaves in Practice

**First conversation about a topic** — no cache entry exists. Claude launches the Explore agent normally, then saves the results. You see no difference except the cache file gets populated.

**Subsequent conversations** — Claude finds the cache entry, checks the timestamp, and skips the agent. The response is faster and cheaper. You'll often see Claude reference the cached files directly without any exploration step.

**After the entry expires** — Claude re-runs the agent and overwrites the entry. This keeps the cache accurate as your codebase evolves.

**Stale entries (>7 days)** — automatically pruned whenever any write to the cache occurs.

## Limitations

- **Claude must follow instructions.** This technique relies on Claude Code reading and obeying the `CLAUDE.md` instructions. In practice it works reliably, but it's a convention, not an enforcement mechanism.
- **No cross-machine sync.** The cache is local. If you work on multiple machines, each builds its own cache independently.
- **Summaries can drift.** If you make major structural changes between cache writes, the summary may be outdated until it expires. Lower `maxAgeDays` if this bothers you, or manually delete the stale key.
- **Not a replacement for CLAUDE.md.** The cache handles dynamic, exploratory knowledge. Static facts about your project (conventions, tech stack, directory structure) still belong in `CLAUDE.md`.

## License

This technique is for the public domain. Use it however you like. Send me a message if you liked it or want to improve on it.
