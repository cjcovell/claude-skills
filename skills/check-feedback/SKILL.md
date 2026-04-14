---
name: check-feedback
description: Query and triage unresolved user feedback from Supabase. Use when starting a session in a project with in-app feedback, when the user says "check feedback", "unresolved feedback", "feedback triage", or when CLAUDE.md startup routine triggers it.
---

# Check Feedback

Query the project's Supabase `feedback` table for unresolved items, summarize them for the user, and help work through fixes. After shipping fixes, mark items resolved and confirm they drop off.

## When to Use

- At session start when CLAUDE.md contains a feedback startup routine
- When the user says "check feedback", "any open feedback?", "feedback triage"
- Mid-session when you want to re-check after resolving items

## Step 1 — Detect Query Method

Determine how to reach the feedback table:

**Option A — Supabase MCP (preferred):**
Check if `mcp__supabase` tools are available in the current session, or if `.claude/settings.local.json` has a Supabase MCP entry. If yes, use MCP for all queries.

**Option B — curl via REST API:**
If MCP is not available:
1. Read `NEXT_PUBLIC_SUPABASE_URL` from `.env.local` to get the project ref (subdomain of the URL)
2. Read `SUPABASE_SERVICE_ROLE_KEY` from `.env.local`
3. Use curl against the Supabase REST API

If neither method works (no MCP, no `.env.local`, no service role key), tell the user what's missing and stop.

## Step 2 — Query Unresolved Feedback

**Via MCP:**
```sql
SELECT * FROM public.feedback WHERE resolved_at IS NULL ORDER BY created_at DESC
```

**Via curl:**
```bash
curl -s "https://<PROJECT_REF>.supabase.co/rest/v1/feedback?select=*&resolved_at=is.null&order=created_at.desc" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" | python3 -m json.tool
```

## Step 3 — Handle Results

**If no unresolved items:** Report "No open feedback items" and stop. No further action needed.

**If items exist:** Continue to Step 4.

## Step 4 — Summarize and Prioritize

Present the results grouped by type in this priority order:

1. **Bugs** — these come first, they affect users right now
2. **Feature requests** — next priority
3. **General** — last

For each item, show:
- Type (bug/feature/general)
- Message (truncate to ~200 chars if very long, show full on request)
- When it was submitted (relative time: "2 hours ago", "3 days ago")

Then recommend which items to address this session. Default to working through bugs first unless the user has a different priority.

## Step 5 — Work Through Fixes

For each item the user agrees to address:
1. Understand the feedback — read the full message, identify the relevant code
2. Implement the fix or feature
3. Do a fresh-eyes review of all changed files before committing
4. Commit the changes

## Step 6 — Mark Resolved

After shipping a fix, mark the corresponding feedback item resolved:

**Via MCP:**
```sql
UPDATE public.feedback SET resolved_at = now() WHERE id = '<uuid>'
```

**Via curl/API:**
```bash
curl -s -X PATCH "https://<PROJECT_REF>.supabase.co/rest/v1/feedback?id=eq.<uuid>" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"resolved_at": "now()"}'
```

Or if the dev server is running, use the admin API:
```bash
curl -s -X PATCH http://localhost:3000/api/admin/feedback \
  -H "Content-Type: application/json" \
  -d '{"id": "<uuid>", "resolved": true}'
```

## Step 7 — Confirm Resolution

Re-run the query from Step 2. Confirm the resolved item no longer appears. Report the updated count of remaining open items to the user.

If all items are resolved, report "All feedback items resolved" and you're done.
