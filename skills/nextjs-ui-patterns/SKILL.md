---
name: nextjs-ui-patterns
description: CJ's preferred UI/UX patterns for all Next.js apps — navigation, mobile UX, status badges, chip filters, OG images, favicons, and more.
---

# Next.js UI Patterns — CJ's Preferences

Apply these preferences when building ANY Next.js app for CJ. These are non-negotiable defaults unless CJ explicitly says otherwise.

## Tech Stack
- Next.js (App Router), TypeScript, Tailwind CSS v4, shadcn/ui
- Prefer `"use client"` single-page apps for small tools
- localStorage for persistence in simple apps (no backend unless needed)

## Navigation / Header
- **Sticky header**: `sticky top-0 z-50` with blur backdrop (`bg-background/95 backdrop-blur supports-backdrop-filter:bg-background/60`)
- **Hamburger menu**: Always on the **RIGHT** side (CJ is right-handed)
- **Mobile menu**: Full-screen overlay, not a tiny dropdown. Big tappable buttons (`py-5`, `text-xl`). Active item highlighted with inverted colors (`bg-foreground text-background rounded-lg`)
- **Desktop**: Standard inline tabs or nav links. Use `hidden sm:grid` to hide on mobile
- **Tab bar on mobile**: Never use a cramped 4-column grid. Either use the hamburger full-screen menu, or a 2x2 grid at minimum

## Status Badges
- Color-coded, **tap to cycle** through states
- Three states:
  - **Todo / Needs Action**: red — `bg-red-100 text-red-700 border-red-200 hover:bg-red-200`
  - **In Progress / Booked**: amber — `bg-amber-100 text-amber-700 border-amber-200 hover:bg-amber-200`
  - **Done / Confirmed**: green — `bg-green-100 text-green-700 border-green-200 hover:bg-green-200`
- Badge style: `rounded-full px-2 py-0.5 text-xs font-medium border cursor-pointer`

## Chip-Style Filters
- Rounded pill buttons: `rounded-full px-3 py-1.5 text-sm font-medium border`
- **Active chip**: filled with the corresponding status color (or inverted `bg-foreground text-background` for "All")
- **Inactive chip**: outlined, muted hover — `bg-background text-foreground border-border hover:bg-muted`
- Show count next to each chip label (e.g., "Todo (5)")
- Always include an **"All"** chip as the first option
- Place chip filters between the section header and the item list

## Delete Confirmation
- **Always two-step delete** — never single-click delete for important data
- First click shows inline "Delete" + "Cancel" buttons replacing the original delete button
- Red "Delete" button, gray "Cancel" button
- Auto-reset back to initial state after 3 seconds if no action taken

## General Mobile UX Principles
- Big tap targets. Never tiny controls
- Full-screen overlays over dropdowns/popovers on mobile
- Right-side placement for primary actions (right-handed user)
- `h-auto` on any grid that might wrap on mobile

## OG Images, Favicons & Metadata
- **Always set up OG/Twitter metadata** — every app should look good when shared via iMessage/Slack/Twitter
- **OG image**: Use Next.js `opengraph-image.tsx` convention (generates from JSX via `ImageResponse`, no external image files needed). Dark gradient background, bold white text, key details. Size: 1200x630.
- **Twitter image**: Re-export from OG image file (`export { default, alt, contentType, runtime, size } from "./opengraph-image"`)
- **Favicon**: SVG `icon.svg` in app directory for crisp scaling at all sizes
- **Apple touch icon**: `apple-icon.tsx` using `ImageResponse`, 180x180, matching brand style
- **Metadata in layout.tsx**: Rich description with key details. `openGraph.type: "website"`. `twitter.card: "summary_large_image"`.
