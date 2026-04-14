---
name: in-app-feedback
description: Use when adding a user feedback system to a Next.js + Supabase app — floating button, categorized submissions, admin dashboard with resolve/reopen workflow, rate limiting, and RLS.
---

# In-App Feedback System

## Overview

A complete feedback loop: floating button for users to submit categorized feedback (bug, feature, general), an API layer with rate limiting and Zod validation, and an admin dashboard to triage/resolve items. Built on Supabase with RLS for security.

## When to Use

- Adding user feedback collection to any Next.js + Supabase app
- Need a lightweight alternative to third-party feedback tools (Canny, UserVoice, etc.)
- Want admin visibility into user-reported bugs and feature requests

## Architecture

```
User → FeedbackButton (Dialog) → POST /api/feedback → Supabase feedback table
Admin → /admin (Feedback tab) → GET/PATCH /api/admin/feedback → feedback table (service role)
```

## Components

### 1. Floating Feedback Button

Client component. Fixed bottom-right, opens a Dialog modal.

```tsx
// components/feedback-button.tsx
"use client"

import { useState } from "react"
import { MessageSquare, Loader2 } from "lucide-react"
import { toast } from "sonner"
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger } from "@/components/ui/dialog"

type FeedbackType = "bug" | "feature" | "general"

const typeLabels: { value: FeedbackType; label: string }[] = [
  { value: "general", label: "General" },
  { value: "bug", label: "Bug" },
  { value: "feature", label: "Feature Request" },
]

export function FeedbackButton() {
  const [open, setOpen] = useState(false)
  const [type, setType] = useState<FeedbackType>("general")
  const [message, setMessage] = useState("")
  const [isSubmitting, setIsSubmitting] = useState(false)

  async function handleSubmit() {
    if (!message.trim()) return
    setIsSubmitting(true)
    try {
      const res = await fetch("/api/feedback", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ type, message: message.trim() }),
      })
      if (!res.ok) {
        const data = await res.json().catch(() => ({}))
        toast.error(data.error || "Failed to submit feedback")
        return
      }
      toast.success("Thanks for your feedback!")
      setMessage("")
      setType("general")
      setOpen(false)
    } catch {
      toast.error("Failed to submit feedback")
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <button
          className="fixed bottom-20 sm:bottom-6 right-4 z-50 flex items-center justify-center h-10 w-10 rounded-full bg-card border border-border shadow-md hover:shadow-lg transition-shadow text-muted-foreground hover:text-foreground"
          aria-label="Send feedback"
        >
          <MessageSquare className="h-4 w-4" />
        </button>
      </DialogTrigger>
      <DialogContent className="sm:max-w-md">
        <DialogHeader>
          <DialogTitle>Send Feedback</DialogTitle>
        </DialogHeader>
        <div className="space-y-3">
          {/* Type selector pills */}
          <div className="flex gap-2">
            {typeLabels.map((t) => (
              <button
                key={t.value}
                onClick={() => setType(t.value)}
                className={`rounded-lg px-3 py-1.5 text-xs font-medium transition-colors ${
                  type === t.value
                    ? "bg-primary text-primary-foreground"
                    : "bg-muted text-muted-foreground hover:text-foreground"
                }`}
              >
                {t.label}
              </button>
            ))}
          </div>
          <textarea
            value={message}
            onChange={(e) => setMessage(e.target.value)}
            placeholder="What's on your mind?"
            rows={4}
            maxLength={5000}
            className="w-full rounded-lg border border-input bg-background px-3 py-2.5 text-sm text-foreground placeholder:text-muted-foreground resize-none focus:outline-none focus:ring-2 focus:ring-ring"
          />
          <div className="flex justify-end gap-2">
            <button
              onClick={() => setOpen(false)}
              className="rounded-lg border border-border px-3 py-2 text-sm text-muted-foreground hover:text-foreground transition-colors"
            >
              Cancel
            </button>
            <button
              onClick={handleSubmit}
              disabled={!message.trim() || isSubmitting}
              className="rounded-lg bg-primary text-primary-foreground px-4 py-2 text-sm font-medium disabled:opacity-50 flex items-center gap-2"
            >
              {isSubmitting && <Loader2 className="h-3 w-3 animate-spin" />}
              Submit
            </button>
          </div>
        </div>
      </DialogContent>
    </Dialog>
  )
}
```

**Integration:** Add `<FeedbackButton />` and `<Toaster />` in `app/layout.tsx` after `{children}`.

### 2. Database Schema

Two migration scripts:

```sql
-- 007_create_feedback_table.sql
CREATE TABLE IF NOT EXISTS public.feedback (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  type TEXT NOT NULL DEFAULT 'general' CHECK (type IN ('bug', 'feature', 'general')),
  message TEXT NOT NULL,
  page TEXT,                              -- optional: capture which page feedback came from
  created_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE public.feedback ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can insert their own feedback"
  ON public.feedback FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can read their own feedback"
  ON public.feedback FOR SELECT USING (auth.uid() = user_id);
```

```sql
-- 008_feedback_resolved.sql
ALTER TABLE public.feedback ADD COLUMN IF NOT EXISTS resolved_at TIMESTAMPTZ DEFAULT NULL;

CREATE POLICY "Admins can read all feedback"
  ON public.feedback FOR SELECT
  USING ((SELECT (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'));

CREATE POLICY "Admins can update feedback"
  ON public.feedback FOR UPDATE
  USING ((SELECT (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'));
```

### 3. Rate Limiter

In-memory sliding window. Swap to Redis/KV for distributed deployments.

```ts
// lib/rate-limit.ts
interface RateLimitEntry { count: number; resetAt: number }

const buckets = new Map<string, RateLimitEntry>()
let lastCleanup = Date.now()

function cleanup() {
  const now = Date.now()
  if (now - lastCleanup < 60_000) return
  lastCleanup = now
  for (const [key, entry] of buckets) {
    if (now > entry.resetAt) buckets.delete(key)
  }
}

export function rateLimit(
  userId: string,
  endpoint: string,
  { maxRequests = 10, windowMs = 60_000 } = {}
): { allowed: true } | { allowed: false; retryAfterSeconds: number } {
  cleanup()
  const key = `${userId}:${endpoint}`
  const now = Date.now()
  const entry = buckets.get(key)

  if (!entry || now > entry.resetAt) {
    buckets.set(key, { count: 1, resetAt: now + windowMs })
    return { allowed: true }
  }
  if (entry.count < maxRequests) {
    entry.count++
    return { allowed: true }
  }
  return { allowed: false, retryAfterSeconds: Math.ceil((entry.resetAt - now) / 1000) }
}
```

### 4. API Routes

**User submission** — `app/api/feedback/route.ts`:
- Auth required (401 if not logged in)
- Rate limit: 3 per minute per user
- Zod validation: `type` enum + `message` string (1-5000 chars)
- Insert to `feedback` table via authenticated Supabase client

**Admin management** — `app/api/admin/feedback/route.ts`:
- Admin check: `user.app_metadata?.role === "admin"` (403 otherwise)
- `GET`: List all feedback (newest first), enriched with user emails via service role
- `PATCH`: Toggle `resolved_at` (set to ISO timestamp or null)

### 5. Admin Dashboard

Add a "Feedback" tab to `/admin` with:
- **Open feedback** — listed with type badge, user email, message, timestamp, and a CheckCircle2 button to resolve
- **Resolved feedback** — `opacity-60`, `line-through` on message text, RotateCcw button to reopen
- **Type badges** — colored pills: `bug` = destructive, `feature` = primary, `general` = muted
- **Badge in tab** — show unresolved count as a small pill next to "Feedback" tab label

## Quick Reference

| Piece | Location | Key Details |
|-------|----------|-------------|
| Feedback button | `components/feedback-button.tsx` | Fixed bottom-right, Dialog modal, 3 type pills |
| User API | `app/api/feedback/route.ts` | POST, auth + rate limit (3/min) + Zod |
| Admin API | `app/api/admin/feedback/route.ts` | GET (list) + PATCH (resolve/reopen), admin role check |
| Admin UI | `app/admin/page.tsx` | Tabbed, open/resolved sections, toggle buttons |
| DB schema | `scripts/007_*.sql`, `008_*.sql` | RLS: users own rows, admins see all |
| Rate limiter | `lib/rate-limit.ts` | In-memory sliding window, periodic cleanup |

## 6. CLAUDE.md Startup Routine

After scaffolding all code above, write a startup routine into the project's `CLAUDE.md` (create the file if it doesn't exist). This closes the feedback loop — Claude checks for unresolved feedback at the start of every session.

**Detect the right query method:**

1. Check if Supabase MCP is available: look for `mcp__supabase` tools in the session, or a Supabase MCP entry in `.claude/settings.local.json`
2. If MCP is available, write the MCP variant:

```markdown
## Startup Routine

At the start of each session, check for unresolved user feedback:

Run via Supabase MCP:
`SELECT * FROM public.feedback WHERE resolved_at IS NULL ORDER BY created_at DESC`

Summarize open feedback items for the user and suggest which to address. Prioritize bugs, then feature requests, then general feedback.

After fixing an item, mark it resolved:
`UPDATE public.feedback SET resolved_at = now() WHERE id = '<uuid>'`

Then re-query to confirm it dropped off. Do a fresh-eyes review of all changed code before committing.
```

3. If MCP is not available, read `NEXT_PUBLIC_SUPABASE_URL` from `.env.local` to extract the project ref (the subdomain: `https://<PROJECT_REF>.supabase.co`), and write the curl variant:

````markdown
## Startup Routine

At the start of each session, check for unresolved user feedback:

```bash
curl -s "https://<PROJECT_REF>.supabase.co/rest/v1/feedback?select=*&resolved_at=is.null&order=created_at.desc" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" | python3 -m json.tool
```

The `SUPABASE_SERVICE_ROLE_KEY` is in `.env.local`. Summarize open feedback items for the user and suggest which to address. Prioritize bugs, then feature requests, then general feedback.

After fixing an item, mark it resolved via PATCH `/api/admin/feedback` with `{ "id": "<uuid>", "resolved": true }`, or run:
`UPDATE public.feedback SET resolved_at = now() WHERE id = '<uuid>'` in the Supabase SQL editor.

Then re-query to confirm it dropped off. Do a fresh-eyes review of all changed code before committing.
````

**Replace `<PROJECT_REF>` with the actual value** before writing to CLAUDE.md.

## 7. Verification

Before declaring the setup complete, verify all five steps pass:

1. **TypeScript check** — Run the project's typecheck (`npx tsc --noEmit`, `pnpm typecheck`, etc.). Fix any errors this change introduced.
2. **Dev server boots** — Start the dev server and confirm no runtime errors in the console.
3. **Smoke test** — Sign in, see the feedback button bottom-right, open the dialog, submit a test message. Confirm the toast says "Thanks for your feedback!" and the row appears in the `feedback` table with correct `user_id`, `type`, `message`.
4. **Startup query works** — Run the startup query (MCP or curl). Confirm the test row comes back.
5. **Resolution loop closes** — Mark that row resolved and re-run the startup query. Confirm the row no longer appears.

Do not claim the setup is done based on code compilation alone — the whole point is the closed loop.

## Common Mistakes

- **Forgetting RLS policies** — without admin SELECT/UPDATE policies, the admin API returns empty results even with service role
- **Missing Toaster** — feedback button uses `sonner` toasts; add `<Toaster />` in layout
- **No rate limiting** — users can spam feedback; always add rate limiting on the POST endpoint
- **Single-click resolve** — use toggle (resolve/reopen) so admins can undo mistakes
- **Not writing the CLAUDE.md startup routine** — the feedback system is only half the value without the closed loop. Always scaffold the startup routine as part of setup.
