# Feedback Loop Skills — Design Spec

## Goal

Two skills that create a closed feedback loop for Next.js + Supabase apps:

1. **`in-app-feedback`** (existing, refined) — Scaffolds the full feedback system into a project, including writing the CLAUDE.md startup routine.
2. **`check-feedback`** (new) — Queries unresolved feedback, summarizes it, proposes fixes, and marks items resolved after shipping.

## Skill 1: `in-app-feedback` — Changes

The existing skill covers components 1-5 (button, schema, rate limiter, API routes, admin dashboard). Add two new sections:

### Section 6: CLAUDE.md Startup Routine

After scaffolding all code, write a startup routine block into the project's CLAUDE.md (create the file if it doesn't exist). The block instructs Claude to check for unresolved feedback at session start.

Detection logic for which query method to write:
- If the project has Supabase MCP configured (check `.claude/settings.local.json` or MCP tool availability), write the MCP SQL variant: `SELECT * FROM public.feedback WHERE resolved_at IS NULL ORDER BY created_at DESC`
- Otherwise, write the curl variant using `$SUPABASE_SERVICE_ROLE_KEY` and the project ref extracted from `NEXT_PUBLIC_SUPABASE_URL` in `.env.local`

The startup block should also instruct Claude to summarize findings and suggest which items to address.

### Section 7: Verification Checklist

Five-step verification before declaring setup complete:
1. TypeScript check passes (`npx tsc --noEmit` or project's typecheck command)
2. Dev server boots without errors
3. Smoke test: sign in, see button, submit test feedback, confirm row in DB
4. Startup query returns the test row
5. Mark resolved, re-query, confirm it's gone

## Skill 2: `check-feedback` — New

### Frontmatter

```yaml
name: check-feedback
description: Query and triage unresolved user feedback from Supabase. Use when starting a session in a project with in-app feedback, when the user says "check feedback", "unresolved feedback", "feedback triage", or when CLAUDE.md startup routine triggers it.
```

### Behavior

**Step 1 — Detect query method:**
- Check if Supabase MCP is available (look for `mcp__supabase` tools or `.claude/settings.local.json` with supabase MCP config)
- If yes: use `mcp__supabase__execute_sql` with the query
- If no: read `NEXT_PUBLIC_SUPABASE_URL` from `.env.local` to get project ref, read `SUPABASE_SERVICE_ROLE_KEY`, run curl

**Step 2 — Query unresolved feedback:**
```sql
SELECT * FROM public.feedback WHERE resolved_at IS NULL ORDER BY created_at DESC
```

**Step 3 — Handle empty results:**
- If no rows: report "No open feedback items" and stop

**Step 4 — Summarize and prioritize:**
- Group by type: bugs first, then feature requests, then general
- For each item: show type, message (truncated if long), submission date
- Recommend which to address this session (bugs take priority)

**Step 5 — After fixes are shipped:**
- Mark resolved items via:
  - MCP: `UPDATE public.feedback SET resolved_at = now() WHERE id = '<uuid>'`
  - Or PATCH `/api/admin/feedback` with `{ id, resolved: true }`
- Re-run the query to confirm items dropped off
- Do a fresh-eyes review of changed code before committing

## File Structure

```
claude-skills/
├── skills/
│   ├── in-app-feedback/
│   │   └── SKILL.md          # Refined with sections 6-7
│   └── check-feedback/
│       └── SKILL.md          # New
└── docs/
    └── specs/
        └── 2026-04-14-feedback-loop-design.md  # This file
```

## Out of Scope

- Framework support beyond Next.js + Supabase
- Notification system (email/Slack when feedback submitted)
- Feedback analytics or dashboards beyond the existing admin page
- Plugin packaging (marketplace.json, hooks, etc.) — these are standalone skills in CJ's claude-skills repo, not a full plugin
