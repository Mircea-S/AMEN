### Add to your project memory file
## AMEN Cache with Mandatory Write-Back
- After ANY Explore agent returns results, your **very next action** MUST be writing to `.agent-cache.json`. Do not use the results, continue the task, or write to a plan file first. Cache write is a blocking prerequisite.
- This applies in all modes: plan mode, normal mode, background agents.
- Format: `{ ts, summary (plain text, <300 chars), files: [...] }` under `topics.<kebab-key>`.
- **Post-implementation reconciliation**: After implementing changes that fix an issue described in a cached finding, update that cache entry's summary to reflect the fix and refresh its `ts`. This prevents future sessions from treating resolved issues as current.
