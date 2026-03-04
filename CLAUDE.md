# Knowledge Base

Build a searchable knowledge base with articles, categories, and contributor management — powered by Commander Documents, Tables, and Search. Replaces Guru, Confluence, and Notion wikis.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React + TypeScript + Vite + Tailwind CSS |
| Auth | Clerk (via WaymakerOS) |
| Data | Commander Tables (Supabase PostgreSQL) |
| Documents | Commander Documents (article content) |
| Search | Commander Search (full-text article search) |
| Rich Text | TipTap (or similar markdown/WYSIWYG editor) |
| Hosting | Waymaker Host (CX app) |

## Documentation

| Folder | Contents |
|--------|----------|
| `docs/01-planning/product-requirements/` | PRD — data model, views, integration map |
| `docs/02-working/prompts/active/` | Build prompts — 4 phases with YAML front matter |
| `docs/02-working/sessions/` | Session briefs as you build |
| `docs/03-knowledge/` | Patterns discovered during the build |

## Build Phases

| Phase | Prompt | Status |
|-------|--------|--------|
| 1 — Editor & Categories | `docs/02-working/prompts/active/phase-1-editor-and-categories.md` | todo |
| 2 — Public Reader | `docs/02-working/prompts/active/phase-2-public-reader.md` | todo |
| 3 — Search & Contributors | `docs/02-working/prompts/active/phase-3-search-and-contributors.md` | todo |
| 4 — Dashboard & Polish | `docs/02-working/prompts/active/phase-4-dashboard-and-polish.md` | todo |

## Data Model (Quick Reference)

```
kb_categories ──< kb_articles
     │                │
  (nesting)      (status, tags,
                  helpful votes,
                  view counts)
```

- **Category** defines article groupings with optional nesting, icons, and colours
- **Article** is the core record — one row per article with rich text content, status, and engagement metrics
- Articles link to categories via `category_id`
- Nested categories supported via `parent_id` self-reference

## Commander Integration

| Action | Commander Tool | API |
|--------|---------------|-----|
| Article content | Documents | `commander-document-operations` → CRUD |
| All structured data | Tables | `commander-table-operations` → CRUD on kb_articles, kb_categories |
| Full-text search | Search | `commander-search` → search published articles |
| Contributor profiles | Contacts | `commander-contact-operations` → author lookups |
| Article request notifications | Automations | `commander-automation-operations` → email on new requests |
| Email delivery | Email | `commander-email-send` → send notification emails |

## Critical Rules

- All data lives in Commander Tables — the app is the view + logic layer
- Auth via Clerk: `useAuth().getToken()` → Bearer token on all API calls
- API calls POST to `${SUPABASE_URL}/functions/v1/{function-name}` with `{ action, data }` body
- Rich text stored as markdown in the `content` field on kb_articles
- Design tokens: Purple `#7C3AED` primary, White `#FFFFFF` background, Green `#059669` published, Amber `#D97706` draft, Red `#DC2626` danger
- Use Geist font family
- All tables scoped by `organization_id` with RLS policies
- Public reader view does NOT require auth — published articles are public
- Article slugs must be unique per organization

## How to Build

1. Read the PRD: `docs/01-planning/product-requirements/knowledge-base-prd.md`
2. Work through each phase prompt in order
3. Update the YAML `status` field as you go: `todo` → `in-progress` → `review` → `done`
4. Write session briefs in `docs/02-working/sessions/completed/` between sessions

## Quick Reference

| What | How |
|------|-----|
| Dev server | `npm run dev` |
| Build | `npm run build` |
| Tables API | `commander-table-operations` → `query`, `insert`, `update`, `delete` |
| Documents API | `commander-document-operations` → article content |
| Search API | `commander-search` → full-text search |
| Contacts API | `commander-contact-operations` → contributor profiles |
| Automations API | `commander-automation-operations` → notification triggers |
