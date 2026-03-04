---
sync:
  type: doc
  layer: Knowledge Base
build:
  status: todo
  phase: 1
  priority: P0
  depends_on: []
  started_at: null
  completed_at: null
---

# Phase 1: Article Editor & Categories

**Goal:** App scaffold with auth, data schema, article CRUD with a rich text editor, category management with nesting, auto-slug generation, and draft/published/archived status management.

**PRD Reference:** `docs/01-planning/product-requirements/knowledge-base-prd.md` — Phase 1

---

## What to Build

### 1. Project Setup

Create a React + Vite + TypeScript + Tailwind app.

**Files:**
- `src/main.tsx` — Clerk provider wrapper
- `src/App.tsx` — Router with routes: `/admin/articles`, `/admin/articles/new`, `/admin/articles/:id/edit`, `/admin/categories`, `/admin/dashboard`, `/kb`, `/kb/:category`, `/kb/:category/:slug`, `/kb/search`
- `src/lib/api.ts` — Authenticated fetch helper for Commander Tables
- `src/lib/types.ts` — TypeScript interfaces for Article, Category
- `src/lib/slugify.ts` — Slug generation utility
- `src/index.css` — Tailwind base + design tokens

**Auth pattern:**
```typescript
import { useAuth } from '@clerk/clerk-react'

const { getToken } = useAuth()
const token = await getToken()

const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({ action: 'query', data: { table: 'kb_articles', filters: {} } }),
})
```

**TypeScript interfaces:**
```typescript
interface Article {
  id: string;
  organization_id: string;
  title: string;
  slug: string;
  content: string;
  excerpt: string | null;
  category_id: string;
  status: 'draft' | 'published' | 'archived';
  author_id: string;
  helpful_yes: number;
  helpful_no: number;
  view_count: number;
  tags: string[] | null;
  sort_order: number;
  published_at: string | null;
  created_at: string;
  updated_at: string;
}

interface Category {
  id: string;
  organization_id: string;
  name: string;
  slug: string;
  description: string | null;
  icon: string | null;
  colour: string | null;
  sort_order: number;
  parent_id: string | null;
  article_count: number;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}
```

### 2. Database Schema

Create these tables in Commander Tables. Use the API or run migrations directly.

**kb_categories** — Seed with defaults:

| name | icon | colour |
|------|------|--------|
| Getting Started | rocket | #7C3AED |
| How-To Guides | book-open | #2563EB |
| Troubleshooting | wrench | #DC2626 |
| Policies & Procedures | shield | #059669 |
| FAQs | help-circle | #D97706 |
| Product Updates | megaphone | #EC4899 |
| Best Practices | star | #F59E0B |
| Other | circle | #9CA3AF |

**kb_articles** — See PRD for full schema. Key fields: title, slug, content, excerpt, category_id, status, author_id, helpful_yes, helpful_no, view_count, tags, sort_order, published_at.

### 3. API Layer

Create a service module for each entity:

```
src/services/
├── articles.ts    — list, get, create, update, delete, publish, archive
├── categories.ts  — list, get, create, update, delete, toggleActive, reorder
└── slugify.ts     — generateSlug, checkSlugUnique
```

All table operations go through `commander-table-operations` edge function with the appropriate table name and action.

**Article service example:**
```typescript
// src/services/articles.ts

export async function listArticles(token: string, filters?: {
  status?: string;
  category_id?: string;
  search?: string;
}): Promise<Article[]> {
  const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'query',
      data: { table: 'kb_articles', filters: filters || {} },
    }),
  });
  const json = await res.json();
  return json.data;
}

export async function createArticle(token: string, article: Partial<Article>): Promise<Article> {
  const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'insert',
      data: { table: 'kb_articles', record: article },
    }),
  });
  const json = await res.json();
  return json.data;
}

export async function publishArticle(token: string, id: string): Promise<Article> {
  return updateArticle(token, id, {
    status: 'published',
    published_at: new Date().toISOString(),
  });
}
```

**Slug generation:**
```typescript
// src/lib/slugify.ts

export function generateSlug(title: string): string {
  return title
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-+|-+$/g, '')
    .slice(0, 80);
}

export async function ensureUniqueSlug(
  token: string,
  slug: string,
  organizationId: string,
  excludeId?: string
): Promise<string> {
  // Query for existing slugs matching this base
  // If collision, append -2, -3, etc.
  let candidate = slug;
  let counter = 1;
  while (await slugExists(token, candidate, organizationId, excludeId)) {
    counter++;
    candidate = `${slug}-${counter}`;
  }
  return candidate;
}
```

### 4. Article List (/admin/articles)

The admin home page. A table of all articles with summary stats and quick actions.

**Summary bar at top:**
- Total articles (count)
- Published (count)
- Drafts (count)

**Table columns:**
- Title (link to editor, searchable)
- Category (colored badge matching category colour)
- Status (badge: draft=amber, published=green, archived=grey)
- Author (name from Clerk user)
- Views (count, right-aligned)
- Helpful % (calculated: helpful_yes / (helpful_yes + helpful_no) * 100)
- Updated (relative date: "2 hours ago")

**Filters:**
- Category dropdown (multi-select)
- Status dropdown (multi-select: draft, published, archived)
- Search by title

**Actions:**
- "New Article" button → navigate to article editor
- Click row → navigate to article editor
- Bulk select → bulk publish, bulk archive, bulk delete (drafts only)

**Empty state:** "No articles yet. Write your first article to get started." with New Article button.

### 5. Article Editor (/admin/articles/new and /admin/articles/:id/edit)

A full-page editor with rich text and metadata sidebar.

**Layout:**
- Left column (70%): Rich text editor
- Right column (30%): Metadata sidebar

**Rich text editor (TipTap recommended):**
- Toolbar: Bold, Italic, Strikethrough, Heading (H1-H3), Bullet list, Numbered list, Code block, Blockquote, Link, Image, Horizontal rule, Undo/Redo
- Markdown shortcuts: `#` for headings, `**` for bold, `-` for lists, ``` for code blocks
- Support pasting markdown content
- Auto-save every 30 seconds (update the record silently)
- Content stored as markdown in the `content` field

**Metadata sidebar:**
- Title (text input, required — updating title auto-updates slug suggestion)
- Slug (text input, auto-generated from title, editable, with uniqueness validation)
- Category (dropdown from kb_categories where is_active = true)
- Tags (multi-select or tag input — comma-separated, stored as text[])
- Excerpt (textarea, auto-generated from first 200 chars of content, editable)
- Status indicator: current status with actions
  - If draft: "Publish" button, "Archive" button
  - If published: "Unpublish" (→ draft) button, "Archive" button
  - If archived: "Restore to Draft" button
- Author: display current user name (auto-set)
- Created: display date
- Last updated: display date

**On save:**
1. Auto-generate excerpt from first 200 characters of plain text content (if not manually set)
2. Validate slug uniqueness
3. Save article to Commander Tables
4. Show toast: "Article saved"

**On publish:**
1. Set status to `published`
2. Set `published_at` to current timestamp
3. Show toast: "Article published"

### 6. Category Management (/admin/categories)

List of all categories with management actions.

**Grid view (cards):**
- Category icon (Lucide icon) + colour dot
- Category name
- Description (truncated)
- Article count
- Active/inactive indicator
- Parent category (if nested)
- Sort order handle (drag to reorder)

**Actions:**
- "Add Category" → modal: name, slug (auto-generated), description, icon (picker), colour (picker), parent_id (dropdown of existing categories), sort_order
- Edit category → same modal in edit mode
- Toggle active/inactive (inactive categories hidden from article editor dropdown and public navigation)
- Delete category (only if article_count = 0, otherwise show warning)

**Category nesting:**
- Parent category dropdown in add/edit modal (optional)
- Nested display: indent child categories under parents
- Breadcrumb hierarchy: "Getting Started > Quick Start Guide"
- Maximum depth: 2 levels (parent → child)

**Icon picker:**
- Grid of common Lucide icons relevant to knowledge bases
- Suggested icons: rocket, book-open, wrench, shield, help-circle, megaphone, star, circle, file-text, lightbulb, code, users, settings, zap, globe, lock

**Colour picker:**
- Predefined palette of 12 colours (matching design system)
- Custom hex input option

### 7. Auto-Slug Generation

Automatic slug creation from article title and category name.

**Article slug rules:**
- Generated from title on first save
- Lowercase, alphanumeric + hyphens only
- Max 80 characters
- Unique per organization (append -2, -3 if collision)
- Editable by user (with uniqueness re-validation)
- Example: "How to Reset Your Password" → `how-to-reset-your-password`

**Category slug rules:**
- Generated from name on first save
- Same sanitization as article slugs
- Unique per organization
- Example: "Getting Started" → `getting-started`

### 8. Design Tokens

Use a clean, professional design:
- Font: `Geist` (load from Google Fonts)
- Primary: `#7C3AED` (purple) — primary buttons, links, active states
- Background: `#FFFFFF` (white) — page background
- Surface: `#F9FAFB` (off-white) — card backgrounds, sidebar
- Text: `#111827` (near-black)
- Text Muted: `#6B7280` (grey)
- Success: `#059669` (green) — published status
- Warning: `#D97706` (amber) — draft status
- Danger: `#DC2626` (red) — delete actions, archived status
- Border: `#E5E7EB`
- Cards: white, `border-radius: 12px`, subtle shadow
- Spacing: 8px base grid

---

## Acceptance Criteria

- [ ] App loads with Clerk auth (sign-in required for admin routes)
- [ ] kb_articles and kb_categories tables created via commander-table-ddl
- [ ] Default categories seeded on first load (8 categories)
- [ ] Article list shows all articles with correct columns
- [ ] Search by title works in article list
- [ ] Filters work: category, status
- [ ] Rich text editor loads with TipTap (or similar) and full toolbar
- [ ] Markdown shortcuts work in editor (# for headings, ** for bold, etc.)
- [ ] Auto-save fires every 30 seconds
- [ ] Article metadata sidebar: category, tags, status, slug, excerpt all functional
- [ ] Auto-slug generated from title, editable, uniqueness validated
- [ ] Excerpt auto-generated from first 200 chars of content
- [ ] Publish button sets status to published and records published_at
- [ ] Archive and restore status transitions work correctly
- [ ] Category list shows all categories with icons, colours, article counts
- [ ] Add/edit category modal works with icon picker and colour picker
- [ ] Category nesting works (parent_id dropdown, indented display)
- [ ] Toggle category active/inactive works
- [ ] Delete category blocked if articles exist in it
- [ ] Empty states for article list and category list
- [ ] Design tokens applied: purple primary, Geist font, white/surface backgrounds
- [ ] Responsive: sidebar collapses on mobile
