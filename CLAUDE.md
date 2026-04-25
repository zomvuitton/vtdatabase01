# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Harman** is a single-file HTML/CSS/JavaScript web application for company database management and email campaign tracking, built in Turkish. The entire application lives in `index.html` (~4000 lines, ~152KB).

There is no build system, no package manager, and no server-side code. Open `index.html` directly in any modern browser to run the application.

## Running the Application

```bash
# Open directly in browser (any of these)
open index.html          # macOS
xdg-open index.html      # Linux
start index.html         # Windows
```

There are no build, lint, or test commands — this project has no tooling infrastructure.

## Architecture

### Single-file structure

All HTML, CSS, and JavaScript are embedded in `index.html`. The JavaScript begins around line 2100 inside a `<script>` block at the bottom of the `<body>`.

### Data layer (dual storage)

The app uses a dual-storage strategy:

1. **Supabase (primary)** — REST calls to `szqoldtcinbqjgfyptmq.supabase.co`. The anon key and URL are hardcoded near line 3745. A thin wrapper object `supa` (with `.from(table).select()`, `.upsert()`, `.delete()`) wraps all Supabase REST calls.
2. **localStorage (fallback)** — Every write goes to both. On load, Supabase is tried first; localStorage is used when Supabase is unavailable.

Key localStorage keys: `harman_companies`, `harman_campaigns`, `harman_files`, `harman_sources`, `harman_duplicates`, `harman_auth`, `harman_users`.

### In-memory state

The canonical data lives in module-level arrays:
- `companies[]` — core entity; each entry has `id`, `name`, `domain`, `sector`, `city`, `quality` (clean/dirty/incomplete), `response` (positive/negative/none), `contacts[]`, `sources[]`, `history[]`
- `campaigns[]` — email campaigns with `status`, `segment`, `sends[]`
- `filesData[]` — uploaded assets, with `publicUrl` pointing to Supabase Storage (`harman-files` bucket)
- `duplicatesData[]` — fuzzy-match duplicate pairs

### Data flow

```
User action (click / form submit)
  → Event handler (inline onclick or addEventListener)
  → Mutate in-memory array
  → saveToStorage()          ← writes Supabase + localStorage
  → render function          ← rewrites relevant DOM section via innerHTML
```

### Screens and their render functions

| Screen | Render function |
|--------|----------------|
| Companies (default) | `renderList()` |
| Dashboard | `renderDashboard()` |
| Campaigns | `renderCampList()` |
| Files | `renderFiles()` |
| Duplicates | `renderDuplicates()` |
| Settings | rendered inline in nav handler |

Navigation is handled by a sidebar; clicking a nav item sets a `currentScreen` variable and calls the appropriate render function.

### Authentication

Hardcoded credential check at login. Default credentials: `harman` / `databasedemo`. User roles (admin/editor/viewer) gate certain UI actions but are not enforced server-side.

### Supabase schema

The app expects these tables (no migration files in repo):
- `companies` — columns: `id`, `name`, `domain`, `sector`, `city`, `data_quality`, `status`, `notes` (JSONB — stores response, contacts, sources, history)
- `campaigns` — columns: `id`, `name`, `subject`, `body`, `status`, `notes` (JSONB — stores segment, sentAt, sends array)
- `assets` — columns: `id`, `name`, `file_url`, `file_type`, `year`, `tags`, `description` (JSONB)
- Storage bucket: `harman-files` (public read)

### Initialization

The app boots at the very end of the script:
```javascript
loadFromStorage().then(() => renderList());
```

## Key Conventions

- **All UI is built with `innerHTML` string templates** — no virtual DOM, no framework. When editing UI, keep to this pattern.
- **Turkish UI language** — all user-facing labels, button text, and messages are in Turkish.
- **CSS custom properties** define the design system (dark theme). Colors, spacing, and typography are all controlled via variables declared in `:root`.
- **No module system** — all functions and variables are global within the script block. Avoid name collisions when adding new globals.
- **File uploads** go to Supabase Storage via a raw `fetch` to the Supabase storage REST endpoint, not through any SDK.
