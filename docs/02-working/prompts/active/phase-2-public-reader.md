---
sync:
  type: doc
  layer: Knowledge Base
build:
  status: todo
  phase: 2
  priority: P0
  depends_on: ["phase-1-editor-and-categories"]
  started_at: null
  completed_at: null
---

# Phase 2: Public Reader View

**Goal:** Clean, public-facing article view with category navigation, breadcrumbs, table of contents, "Was this helpful?" voting, view count tracking, related articles, and a knowledge base home page. No auth required for reading.

**PRD Reference:** `docs/01-planning/product-requirements/knowledge-base-prd.md` — Phase 2

---

## What to Build

### 1. Public Route Structure

Set up public routes that do NOT require Clerk authentication.

**Route structure:**
```
/kb                        — Knowledge base home (category grid)
/kb/:categorySlug          — Category landing page (articles in category)
/kb/:categorySlug/:slug    — Article reader
```

**Implementation — split auth boundary:**
```typescript
// src/App.tsx
import { ClerkProvider, SignedIn, SignedOut } from '@clerk/clerk-react'

function App() {
  return (
    <ClerkProvider publishableKey={CLERK_KEY}>
      <Routes>
        {/* Public routes — no auth required */}
        <Route path="/kb" element={<KBHome />} />
        <Route path="/kb/search" element={<KBSearch />} />
        <Route path="/kb/:categorySlug" element={<CategoryPage />} />
        <Route path="/kb/:categorySlug/:slug" element={<ArticleReader />} />

        {/* Admin routes — auth required */}
        <Route element={<RequireAuth />}>
          <Route path="/admin/*" element={<AdminLayout />} />
        </Route>
      </Routes>
    </ClerkProvider>
  )
}
```

**Public API calls (no auth token):**
```typescript
// src/services/public.ts

export async function getPublishedArticle(
  categorySlug: string,
  articleSlug: string
): Promise<Article | null> {
  const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      action: 'query',
      data: {
        table: 'kb_articles',
        filters: { slug: articleSlug, status: 'published' },
      },
    }),
  });
  const json = await res.json();
  return json.data?.[0] || null;
}

export async function getActiveCategories(): Promise<Category[]> {
  const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      action: 'query',
      data: {
        table: 'kb_categories',
        filters: { is_active: true },
        order: { sort_order: 'asc' },
      },
    }),
  });
  const json = await res.json();
  return json.data;
}
```

### 2. Knowledge Base Home (/kb)

The public landing page for the knowledge base.

**Layout:**
- Header: Knowledge base title + search bar (prominent)
- Category grid: cards for each active category

**Category cards:**
- Icon (Lucide icon matching category.icon)
- Category name (bold)
- Description (1-2 lines, truncated)
- Article count: "12 articles"
- Colour accent bar at top of card (matching category.colour)
- Click → navigate to `/kb/:categorySlug`

**Search bar:**
- Prominent search input: "Search articles..."
- On type or submit → navigate to `/kb/search?q=...`
- Keyboard shortcut: Cmd+K or / to focus search

**Featured section (optional):**
- If any articles have sort_order = 0 (featured), show them above categories
- "Popular Articles" section with top 5 by view_count

### 3. Category Landing Page (/kb/:categorySlug)

List of all published articles in a category.

**Layout:**
- Breadcrumbs: Home > Category Name
- Category header: icon, name, description
- Article list: cards for each published article in the category

**Article cards:**
- Title (link to article)
- Excerpt (first 200 chars or custom excerpt)
- Author name
- Published date (relative: "3 days ago")
- View count
- Helpful percentage (if votes exist)
- Tags (small badges)

**Sorting:**
- Default: sort_order ascending, then published_at descending
- Options: Newest, Most viewed, Most helpful

**Nested categories:**
- If category has child categories, show them as sub-sections
- Each sub-section: child category name + its articles
- Or show child categories as tabs within the parent page

**Empty state:** "No articles in this category yet."

### 4. Article Reader (/kb/:categorySlug/:slug)

The core public reading experience for a single article.

**Layout:**
- Left sidebar (20%): Category navigation (collapsible on mobile)
- Main content (60%): Article body
- Right sidebar (20%): Table of contents (sticky, collapsible on mobile)

**Article header:**
- Breadcrumbs: Home > Category > Article Title
- Title (H1)
- Author name + avatar (from Clerk user data)
- Published date
- View count
- Tags (clickable — filter to articles with same tag)
- Estimated read time (word count / 200 WPM)

**Article body:**
- Rendered from markdown content
- Styled headings (H1-H3)
- Code blocks with syntax highlighting (use a lightweight highlighter like Prism or highlight.js)
- Blockquotes styled
- Images responsive with max-width
- Links styled with purple underline
- Lists (bullet and numbered) properly styled

**Markdown rendering:**
```typescript
// src/lib/markdown.ts
import { marked } from 'marked';
// Or use TipTap's read-only mode for consistent rendering

export function renderArticle(content: string): string {
  return marked.parse(content, {
    gfm: true,        // GitHub Flavored Markdown
    breaks: true,      // Line breaks as <br>
  });
}
```

**Table of contents (right sidebar):**
- Auto-generated from H2 and H3 headings in the article
- Sticky position (follows scroll)
- Active heading highlighted based on scroll position
- Click to smooth-scroll to heading
- Collapse on mobile (hamburger toggle)

```typescript
// src/lib/toc.ts
interface TOCItem {
  id: string;
  text: string;
  level: 2 | 3;
}

export function extractTOC(content: string): TOCItem[] {
  const headingRegex = /^(#{2,3})\s+(.+)$/gm;
  const items: TOCItem[] = [];
  let match;
  while ((match = headingRegex.exec(content)) !== null) {
    items.push({
      id: slugify(match[2]),
      text: match[2],
      level: match[1].length as 2 | 3,
    });
  }
  return items;
}
```

### 5. Category Sidebar Navigation

A persistent sidebar on the reader pages showing all categories and their articles.

**Sidebar structure:**
```
Getting Started (icon + colour)
  ├── Welcome to Acme
  ├── Quick Setup Guide
  └── Your First Project

How-To Guides (icon + colour)
  ├── Reset Your Password
  ├── Invite Team Members
  └── Export Your Data

Troubleshooting (icon + colour)
  └── Common Error Codes
```

**Behavior:**
- Categories are expandable/collapsible (accordion)
- Current category auto-expanded
- Current article highlighted in the list
- Article count shown next to category name
- Nested categories: parent → child indentation
- Scrollable if list is long
- Collapsible to icon-only on smaller screens

### 6. Breadcrumbs

Navigation breadcrumbs on all public pages.

**Pattern:**
- Home (/) > KB Home (/kb) > Category (/kb/:category) > Article (/kb/:category/:slug)

**Implementation:**
```typescript
// src/components/Breadcrumbs.tsx
interface BreadcrumbItem {
  label: string;
  href: string;
}

function Breadcrumbs({ items }: { items: BreadcrumbItem[] }) {
  return (
    <nav aria-label="Breadcrumb">
      <ol className="flex items-center gap-2 text-sm text-gray-500">
        {items.map((item, i) => (
          <li key={item.href} className="flex items-center gap-2">
            {i > 0 && <ChevronRight className="h-4 w-4" />}
            {i === items.length - 1 ? (
              <span className="text-gray-900 font-medium">{item.label}</span>
            ) : (
              <a href={item.href} className="hover:text-purple-600">{item.label}</a>
            )}
          </li>
        ))}
      </ol>
    </nav>
  );
}
```

**For nested categories:**
- Home > Parent Category > Child Category > Article

### 7. "Was This Helpful?" Voting

Feedback mechanism on every published article.

**UI:**
- Below article content: "Was this article helpful?"
- Two buttons: thumbs-up (Yes) + thumbs-down (No)
- On click: increment helpful_yes or helpful_no on the article record
- Show confirmation: "Thanks for your feedback!"
- Disable buttons after voting (use localStorage to prevent repeat votes)
- Show current stats: "X out of Y people found this helpful"

**Implementation:**
```typescript
// src/services/public.ts

export async function voteHelpful(articleId: string, isHelpful: boolean): Promise<void> {
  const field = isHelpful ? 'helpful_yes' : 'helpful_no';
  await fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      action: 'update',
      data: {
        table: 'kb_articles',
        id: articleId,
        record: { [field]: { increment: 1 } },
      },
    }),
  });
}

// Prevent duplicate votes with localStorage
function hasVoted(articleId: string): boolean {
  const votes = JSON.parse(localStorage.getItem('kb_votes') || '{}');
  return !!votes[articleId];
}

function recordVote(articleId: string, isHelpful: boolean): void {
  const votes = JSON.parse(localStorage.getItem('kb_votes') || '{}');
  votes[articleId] = isHelpful ? 'yes' : 'no';
  localStorage.setItem('kb_votes', JSON.stringify(votes));
}
```

### 8. View Count Tracking

Track how many times each article is viewed.

**Implementation:**
- On article reader page load, increment `view_count` by 1
- Debounce: only count once per session per article (use sessionStorage)
- Fire-and-forget: don't block rendering on the increment call

```typescript
// src/hooks/useTrackView.ts

export function useTrackView(articleId: string) {
  useEffect(() => {
    const key = `kb_viewed_${articleId}`;
    if (sessionStorage.getItem(key)) return; // Already counted this session

    sessionStorage.setItem(key, 'true');

    // Fire and forget — don't await
    fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        action: 'update',
        data: {
          table: 'kb_articles',
          id: articleId,
          record: { view_count: { increment: 1 } },
        },
      }),
    });
  }, [articleId]);
}
```

### 9. Related Articles

Show related articles at the bottom of each article.

**Logic:**
- Query articles in the same category (excluding current article)
- Order by view_count descending (most popular first)
- Limit to 3-5 articles
- If same category has fewer than 3, backfill from other categories with matching tags

**Display:**
- Section heading: "Related Articles"
- Card layout (horizontal): title, excerpt (truncated to 100 chars), category badge
- Click → navigate to that article

---

## Acceptance Criteria

- [ ] Public routes (/kb, /kb/:category, /kb/:category/:slug) load without auth
- [ ] Admin routes (/admin/*) still require Clerk auth
- [ ] KB home shows category grid with icons, colours, and article counts
- [ ] Search bar on KB home navigates to search page
- [ ] Category landing page shows published articles with excerpts
- [ ] Article reader renders markdown content with styled headings, code blocks, lists
- [ ] Table of contents auto-generated from H2/H3 headings
- [ ] Table of contents highlights active heading on scroll
- [ ] Click TOC item smooth-scrolls to heading
- [ ] Category sidebar navigation shows all categories with article lists
- [ ] Current category expanded and current article highlighted in sidebar
- [ ] Breadcrumbs display correctly: Home > Category > Article
- [ ] Nested category breadcrumbs work: Home > Parent > Child > Article
- [ ] "Was this helpful?" buttons increment helpful_yes/helpful_no
- [ ] Duplicate vote prevention via localStorage
- [ ] "Thanks for your feedback" shown after voting
- [ ] View count increments on article load (once per session)
- [ ] Related articles shown at bottom of article (same category)
- [ ] Read time estimate displayed in article header
- [ ] Empty states for categories with no articles
- [ ] Mobile: sidebar collapses, TOC collapses, full-width content
