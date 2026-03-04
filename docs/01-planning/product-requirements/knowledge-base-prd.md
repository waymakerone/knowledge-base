# Knowledge Base — Product Requirements

**Status:** Approved
**Author:** Waymaker
**Date:** 2026-03-04
**Last Updated:** 2026-03-04

## Problem Statement

Every organisation needs a knowledge base. The pattern is universal: someone writes down how something works — a process, a policy, a troubleshooting guide — and everyone else needs to find it when they need it. Yet most SMBs are stuck between two bad options: pay $8-15/user/month for Guru, Confluence, or Notion and fight with clunky editors, poor search, and content that goes stale — or dump everything into a shared Google Drive folder where articles go to die.

The core workflow is simple: write an article with a rich text editor, assign it to a category, publish it, and let people search and read it. Good knowledge bases add helpful voting ("Was this helpful?"), view tracking, and contributor management. But the data still needs to live somewhere structured, connected to the rest of the business.

This blueprint builds a knowledge base as a WaymakerOS app. Articles live in Commander Tables (the same database that holds tasks, contacts, and deals). Search runs through Commander Search. Contributor profiles come from Commander Contacts. The app is the view and logic layer — Commander is the data layer. The result is a knowledge base that's part of the operating system, not a standalone tool with its own silo.

## Goals

1. **One place for all knowledge** — Every article recorded in a searchable, categorised knowledge base with status tracking and engagement metrics
2. **Rich text editing** — Write articles with a modern editor supporting markdown, headings, code blocks, images, and links
3. **Category management with nesting** — Organise articles into categories with icons, colours, and parent/child hierarchy
4. **Public reader view** — Clean, public-facing article pages with breadcrumbs, table of contents, and helpful voting
5. **Full-text search** — Find any published article instantly with filters, highlighting, and relevance ranking

## Non-Goals

- This is NOT a full CMS — no page builder, custom themes, or template engine
- This is NOT a documentation site generator — no versioning, API docs auto-generation, or multi-language support
- This is NOT a forum — no comments, threads, or discussion on articles
- This is NOT a ticketing system — article requests create contacts, not support tickets

## Data Model

### Core Objects

**Article** — A single knowledge base article written by a team member.
- Core fields: title, slug, content (markdown/rich text), excerpt
- Organisation: category_id, tags, sort_order
- Status: draft → published → archived
- Engagement: helpful_yes, helpful_no, view_count
- Ownership: author_id (Clerk user ID)
- Timestamps: published_at, created_at, updated_at

**Category** — A way to organise articles. Supports nesting for hierarchical navigation.
- Fields: name, slug, description, icon, colour, sort_order, parent_id
- Tracking: article_count, is_active
- Timestamps: created_at, updated_at

### Tables Schema (Commander Tables)

```
kb_articles
├── id (uuid, PK)
├── organization_id (text, FK → Clerk org)
├── title (text, required)
├── slug (text, unique per org)
├── content (text — markdown or rich text)
├── excerpt (text, nullable — first 200 chars auto-generated)
├── category_id (uuid, FK → kb_categories)
├── status (text: draft, published, archived)
├── author_id (text, Clerk user ID)
├── helpful_yes (integer, default 0)
├── helpful_no (integer, default 0)
├── view_count (integer, default 0)
├── tags (text[], nullable)
├── sort_order (integer, default 0)
├── published_at (timestamptz, nullable)
├── created_at (timestamptz)
└── updated_at (timestamptz)

kb_categories
├── id (uuid, PK)
├── organization_id (text)
├── name (text, required)
├── slug (text, unique per org)
├── description (text, nullable)
├── icon (text, nullable)
├── colour (text, nullable — hex code)
├── sort_order (integer, default 0)
├── parent_id (uuid, nullable — self-reference for nesting)
├── article_count (integer, default 0)
├── is_active (boolean, default true)
├── created_at (timestamptz)
└── updated_at (timestamptz)
```

### Default Category Seeds

| Name | Icon | Colour |
|------|------|--------|
| Getting Started | rocket | #7C3AED |
| How-To Guides | book-open | #2563EB |
| Troubleshooting | wrench | #DC2626 |
| Policies & Procedures | shield | #059669 |
| FAQs | help-circle | #D97706 |
| Product Updates | megaphone | #EC4899 |
| Best Practices | star | #F59E0B |
| Other | circle | #9CA3AF |

### Commander Integration Map

| Action | Commander Tool | How |
|--------|---------------|-----|
| All article data | Tables | `commander-table-operations` → CRUD on kb_articles, kb_categories |
| Article content | Documents | `commander-document-operations` → rich text storage and retrieval |
| Full-text search | Search | `commander-search` → search published articles by title, content, tags |
| Contributor profiles | Contacts | `commander-contact-operations` → author lookup and display |
| Article request emails | Automations | `commander-automation-operations` → trigger email on new request |
| Email delivery | Email | `commander-email-send` → send notification emails |

## Proposed Solution

### Overview

A customer-facing (CX) app deployed to Waymaker Host that provides a complete knowledge base: article editor with rich text, category management with nesting, a public reader view, full-text search, helpful voting, and an admin dashboard. All data lives in Commander Tables. Search via Commander Search. Author profiles via Commander Contacts.

The app has two modes: an authenticated admin side for writing and managing articles, and a public reader side for anyone to browse and search.

### Key Views

1. **Article List** (`/admin/articles`) — Admin view. Table of all articles, filterable by category, status, author. Quick-add button. Summary stats at top (total articles, published count, draft count).

2. **Article Editor** (`/admin/articles/:id/edit` or `/admin/articles/new`) — Rich text editor with TipTap or similar. Sidebar panel for metadata: category, tags, status, slug, excerpt. Auto-save. Preview toggle.

3. **Category Manager** (`/admin/categories`) — List/grid of all categories with icons, colours, article counts, and nesting. Add, edit, reorder, toggle active/inactive.

4. **Public Reader** (`/kb/:category/:slug`) — Clean, public-facing article page. No auth required. Category sidebar navigation, breadcrumbs, table of contents from headings, "Was this helpful?" buttons, related articles.

5. **Search** (`/kb/search?q=...`) — Search results page with highlighted matches, category filters, and relevance ranking. Instant search-as-you-type in the header.

6. **Dashboard** (`/admin/dashboard`) — Total articles, total views, helpful percentage, top 10 articles by views, recent drafts, articles needing review.

### User Flow

1. Admin opens Knowledge Base from Host
2. Clicks "New Article" → rich text editor opens
3. Writes article with headings, code blocks, images, links
4. Assigns category "How-To Guides", adds tags ["onboarding", "setup"]
5. Clicks "Publish" → article goes live with auto-generated slug
6. Visitor browses knowledge base, navigates by category
7. Finds article via search or category browsing
8. Reads article, clicks "Yes, this was helpful"
9. Another visitor searches for "setup", finds the article instantly
10. Admin sees dashboard: 50 articles, 1,200 views this month, 87% helpful rate

## Scope

### Phase 1 (MVP) — Editor & Categories

- [ ] App scaffold: React + Vite + Tailwind + Clerk auth
- [ ] Tables schema: kb_articles, kb_categories
- [ ] API layer: CRUD operations for all tables via authenticated edge function
- [ ] Seed default categories
- [ ] Article list with search, filter by category/status
- [ ] Rich text editor (TipTap or similar) with markdown support
- [ ] Article metadata sidebar: category, tags, status, slug, excerpt
- [ ] Auto-slug generation from title
- [ ] Draft/published/archived status management
- [ ] Category management page (list, add, edit, reorder, toggle active)
- [ ] Category nesting with parent_id

### Phase 2 — Public Reader

- [ ] Public reader route (`/kb/:category/:slug`) — no auth required
- [ ] Category sidebar navigation with article counts
- [ ] Article rendering: markdown/rich text → styled HTML
- [ ] Breadcrumbs: Home > Category > Article
- [ ] "Was this helpful?" Yes/No buttons → increment helpful_yes/helpful_no
- [ ] Related articles (same category, excluding current)
- [ ] Table of contents auto-generated from headings
- [ ] View count increment on article load
- [ ] Category landing page: list of articles in a category
- [ ] Knowledge base home: featured categories with article counts

### Phase 3 — Search & Contributors

- [ ] Search bar with instant results and category/status/date filters
- [ ] Search results page with highlighted matches
- [ ] Contributor list: authors from Contacts with article counts
- [ ] Article request form → creates Contact + triggers automation email
- [ ] Search analytics: popular queries (stored in a simple log table or local state)
- [ ] Tag cloud/filter on article list and search results
- [ ] Sort options: relevance, date, views, helpful rating

### Phase 4 — Dashboard & Polish

- [ ] Admin dashboard: total articles, views, helpful %, top 10 by views, recent drafts
- [ ] Bulk publish/archive from article list
- [ ] SEO: meta title/description per article, Open Graph tags
- [ ] Article URL sharing with preview cards
- [ ] Accessibility: keyboard navigation, screen reader labels, focus management
- [ ] Mobile responsive: card view for article list, reading mode for articles
- [ ] waymaker.config.ts manifest for Host Schema
- [ ] Deploy-ready configuration

### Out of Scope

- Article versioning and revision history
- Multi-language / i18n support for articles
- Custom themes or template engine
- Comments or discussion threads on articles
- API documentation auto-generation
- Integration with external knowledge base tools (Zendesk, Intercom)
- AI-generated article drafts (future platform feature via One AI)

## Success Criteria

| Metric | Target |
|--------|--------|
| Article creation | < 60 seconds to create and publish a basic article |
| Search speed | Results returned in < 500ms |
| Public page load | < 2 seconds for any published article |
| Category navigation | All categories visible with article counts in < 1 second |
| Build time (with AI) | < 3 hours for all 4 phases |

## Dependencies

- WaymakerOS organization with Commander access
- Commander Tables for structured data storage
- Commander Documents for article content
- Commander Search for full-text search
- (Optional) Commander Contacts for contributor profiles
- (Optional) Commander Automations for notification emails

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Rich text editor complexity | Medium | Medium | Use TipTap — mature, well-documented, extensible |
| Search performance on large article sets | Low | Medium | Commander Search handles indexing; paginate results |
| Content going stale | Medium | Low | Dashboard highlights articles with low helpful ratings |
| Public access security | Low | Medium | Only published articles visible publicly; admin routes require Clerk auth |

## Open Questions

- [x] Internal (EX) or external (CX)? **CX — public reader + internal admin**
- [x] Rich text editor choice? **TipTap — markdown + WYSIWYG, extensible**
- [x] Article versioning? **Out of scope — future enhancement**
- [ ] AI-generated article suggestions? **Out of scope — future One AI feature**
