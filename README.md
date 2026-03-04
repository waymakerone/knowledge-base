# Knowledge Base

A complete knowledge base app built on WaymakerOS. Write articles with a rich text editor, organise by category, let visitors search and vote on helpfulness — all integrated with Commander Documents, Tables, Search, and Contacts.

## What You Get

- **Article Editor** — Rich text editor with markdown support, auto-save, category assignment, and draft/published/archived status
- **Public Reader** — Clean, public-facing article view with category navigation, breadcrumbs, and table of contents
- **Category Manager** — Organise articles into nested categories with icons, colours, and custom sort order
- **Search** — Full-text search across all published articles with filters, highlighting, and instant results
- **Helpful Ratings** — "Was this helpful?" Yes/No voting on every article with aggregated feedback
- **Dashboard** — Total articles, views, helpful percentage, top articles by views, and recent drafts

## Commander Tools Used

| Tool | How the Knowledge Base Uses It |
|------|------|
| **Tables** | All structured data (articles, categories) |
| **Documents** | Article content storage and retrieval |
| **Search** | Full-text search across published articles |
| **Contacts** | Contributor/author profiles |
| **Automations** | Notification emails for article requests |

## How to Build

1. Clone this blueprint into your project
2. Open `CLAUDE.md` — it's the router file for your AI coding tool
3. Read the PRD in `docs/01-planning/product-requirements/`
4. Work through the 4 phase prompts in `docs/02-working/prompts/active/`
5. Point Claude Code, Cursor, or Codex at each phase and build

**Estimated build time:** 2-3 hours across all 4 phases.

## Build Phases

| Phase | What You Get |
|-------|-------------|
| 1 — Editor & Categories | App scaffold, schema, article CRUD with rich text editor, category management |
| 2 — Public Reader | Public article view, category navigation, breadcrumbs, helpful voting, view counts |
| 3 — Search & Contributors | Full-text search with filters, contributor profiles, article request form |
| 4 — Dashboard & Polish | Admin dashboard, bulk operations, SEO meta tags, accessibility, mobile responsive |

## Prerequisites

- WaymakerOS organization with Commander access
- Commander Tables enabled
- Commander Documents enabled
- Commander Search enabled
- (Optional) Commander Contacts for contributor profiles
- (Optional) Commander Automations for notification emails
