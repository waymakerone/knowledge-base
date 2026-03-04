---
sync:
  type: doc
  layer: Knowledge Base
build:
  status: todo
  phase: 4
  priority: P0
  depends_on: ["phase-3-search-and-contributors"]
  started_at: null
  completed_at: null
---

# Phase 4: Dashboard & Polish

**Goal:** Admin dashboard with key metrics, bulk article operations, SEO meta tags and Open Graph support, accessibility improvements, mobile-responsive layouts, and deploy-ready configuration with waymaker.config.ts manifest.

**PRD Reference:** `docs/01-planning/product-requirements/knowledge-base-prd.md` — Phase 4

---

## What to Build

### 1. Admin Dashboard (/admin/dashboard)

A summary view pulling together all knowledge base metrics and activity.

**Layout — grid of cards:**

**Row 1: Key Metrics (4 cards)**
- Total Articles (count, with breakdown: published/draft/archived)
- Total Views (sum of all view_count, + vs last month percentage)
- Helpful Rate (overall: total helpful_yes / (helpful_yes + helpful_no) * 100, displayed as percentage)
- Articles This Month (count of articles published this month)

**Row 2: Top Articles & Activity (2 cards, equal width)**
- Top 10 Articles by Views: table with rank, title, category badge, view count, helpful %
- Recent Drafts: last 5 draft articles with title, author, last updated — "Continue editing" link

**Row 3: Content Health (2 cards)**
- Low-Rated Articles: articles with helpful rate < 50% (need review/update)
- Popular Searches: top 10 searched terms from search analytics (Phase 3)
  - Highlight zero-result searches as "Content Gaps"

**Row 4: Quick Actions**
- "New Article" button
- "Manage Categories" button
- "View Knowledge Base" button (opens /kb in new tab)

**Dashboard calculations:**
```typescript
// src/lib/dashboard.ts

interface DashboardMetrics {
  totalArticles: number;
  publishedCount: number;
  draftCount: number;
  archivedCount: number;
  totalViews: number;
  viewsLastMonth: number;
  helpfulRate: number;
  articlesThisMonth: number;
}

export function calculateDashboardMetrics(articles: Article[]): DashboardMetrics {
  const now = new Date();
  const thisMonth = new Date(now.getFullYear(), now.getMonth(), 1);
  const lastMonth = new Date(now.getFullYear(), now.getMonth() - 1, 1);
  const lastMonthEnd = new Date(now.getFullYear(), now.getMonth(), 0);

  const totalHelpfulYes = articles.reduce((sum, a) => sum + a.helpful_yes, 0);
  const totalHelpfulNo = articles.reduce((sum, a) => sum + a.helpful_no, 0);
  const totalVotes = totalHelpfulYes + totalHelpfulNo;

  return {
    totalArticles: articles.length,
    publishedCount: articles.filter((a) => a.status === 'published').length,
    draftCount: articles.filter((a) => a.status === 'draft').length,
    archivedCount: articles.filter((a) => a.status === 'archived').length,
    totalViews: articles.reduce((sum, a) => sum + a.view_count, 0),
    viewsLastMonth: 0, // Would need historical tracking — use current views for MVP
    helpfulRate: totalVotes > 0 ? Math.round((totalHelpfulYes / totalVotes) * 100) : 0,
    articlesThisMonth: articles.filter(
      (a) => a.published_at && new Date(a.published_at) >= thisMonth
    ).length,
  };
}
```

**Metric card component:**
```typescript
// src/components/MetricCard.tsx

interface MetricCardProps {
  title: string;
  value: string | number;
  subtitle?: string;
  icon: React.ReactNode;
  trend?: { value: number; label: string };  // e.g., +12% vs last month
}

function MetricCard({ title, value, subtitle, icon, trend }: MetricCardProps) {
  return (
    <div className="bg-white rounded-xl border border-gray-200 p-6 shadow-sm">
      <div className="flex items-center justify-between">
        <div className="text-gray-500 text-sm font-medium">{title}</div>
        <div className="text-purple-600">{icon}</div>
      </div>
      <div className="mt-2 text-3xl font-bold text-gray-900">{value}</div>
      {subtitle && <div className="mt-1 text-sm text-gray-500">{subtitle}</div>}
      {trend && (
        <div className={`mt-2 text-sm ${trend.value >= 0 ? 'text-green-600' : 'text-red-600'}`}>
          {trend.value >= 0 ? '+' : ''}{trend.value}% {trend.label}
        </div>
      )}
    </div>
  );
}
```

### 2. Bulk Operations

Add bulk actions to the article list for efficient content management.

**Bulk actions on article list (/admin/articles):**
- Checkbox column for multi-select
- "Select All" checkbox in header
- When items selected, show bulk action bar:
  - "Publish Selected" — set status to published for all selected drafts
  - "Archive Selected" — set status to archived for all selected
  - "Delete Selected" — delete all selected drafts (with confirmation dialog)
  - "Change Category" — dropdown to reassign category for all selected

**Implementation:**
```typescript
// src/hooks/useBulkActions.ts

interface BulkActionState {
  selectedIds: Set<string>;
  isAllSelected: boolean;
  toggleSelect: (id: string) => void;
  toggleSelectAll: (ids: string[]) => void;
  clearSelection: () => void;
}

export function useBulkActions(): BulkActionState {
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());

  return {
    selectedIds,
    isAllSelected: false,
    toggleSelect: (id) => {
      setSelectedIds((prev) => {
        const next = new Set(prev);
        if (next.has(id)) next.delete(id);
        else next.add(id);
        return next;
      });
    },
    toggleSelectAll: (ids) => {
      setSelectedIds((prev) =>
        prev.size === ids.length ? new Set() : new Set(ids)
      );
    },
    clearSelection: () => setSelectedIds(new Set()),
  };
}

export async function bulkPublish(token: string, articleIds: string[]): Promise<void> {
  await Promise.all(
    articleIds.map((id) =>
      updateArticle(token, id, {
        status: 'published',
        published_at: new Date().toISOString(),
      })
    )
  );
}

export async function bulkArchive(token: string, articleIds: string[]): Promise<void> {
  await Promise.all(
    articleIds.map((id) =>
      updateArticle(token, id, { status: 'archived' })
    )
  );
}
```

**Bulk action bar UI:**
- Fixed bar at bottom of the page when items are selected
- Shows: "X articles selected" + action buttons
- Purple background with white text/buttons
- "Clear selection" link

### 3. SEO Meta Tags

Add meta tags for each public article page to improve search engine visibility.

**Per-article meta tags:**
```typescript
// src/hooks/useSEO.ts

interface SEOMeta {
  title: string;
  description: string;
  og_title: string;
  og_description: string;
  og_type: string;
  og_url: string;
  canonical: string;
}

export function useArticleSEO(article: Article, category: Category): void {
  useEffect(() => {
    // Title tag
    document.title = `${article.title} | ${category.name} | Knowledge Base`;

    // Meta description
    setMeta('description', article.excerpt || article.title);

    // Open Graph tags
    setMeta('og:title', article.title);
    setMeta('og:description', article.excerpt || article.title);
    setMeta('og:type', 'article');
    setMeta('og:url', `${window.location.origin}/kb/${category.slug}/${article.slug}`);

    // Article-specific OG tags
    setMeta('article:published_time', article.published_at || '');
    setMeta('article:author', article.author_id);
    setMeta('article:section', category.name);
    if (article.tags) {
      article.tags.forEach((tag) => appendMeta('article:tag', tag));
    }

    // Canonical URL
    setLink('canonical', `${window.location.origin}/kb/${category.slug}/${article.slug}`);

    return () => {
      // Clean up on unmount
      document.title = 'Knowledge Base';
    };
  }, [article, category]);
}

function setMeta(property: string, content: string): void {
  let meta = document.querySelector(`meta[property="${property}"], meta[name="${property}"]`);
  if (!meta) {
    meta = document.createElement('meta');
    if (property.startsWith('og:') || property.startsWith('article:')) {
      meta.setAttribute('property', property);
    } else {
      meta.setAttribute('name', property);
    }
    document.head.appendChild(meta);
  }
  meta.setAttribute('content', content);
}

function setLink(rel: string, href: string): void {
  let link = document.querySelector(`link[rel="${rel}"]`);
  if (!link) {
    link = document.createElement('link');
    link.setAttribute('rel', rel);
    document.head.appendChild(link);
  }
  link.setAttribute('href', href);
}
```

**Category page meta tags:**
- Title: "[Category Name] | Knowledge Base"
- Description: category.description or "Browse [X] articles in [Category Name]"

**KB home meta tags:**
- Title: "Knowledge Base | [Organization Name]"
- Description: "Find answers, how-to guides, and documentation."

### 4. Article URL Sharing

Make articles easy to share with preview cards.

**Share button on article reader:**
- "Share" button with dropdown:
  - Copy link to clipboard (show toast: "Link copied!")
  - Share via email (mailto: link with subject and URL)
  - Share via Twitter/X (pre-filled tweet with title + URL)
  - Share via LinkedIn (pre-filled post with URL)

**Implementation:**
```typescript
// src/components/ShareButton.tsx

function ShareButton({ article, category }: { article: Article; category: Category }) {
  const url = `${window.location.origin}/kb/${category.slug}/${article.slug}`;

  const copyLink = async () => {
    await navigator.clipboard.writeText(url);
    toast.success('Link copied!');
  };

  const shareEmail = () => {
    const subject = encodeURIComponent(article.title);
    const body = encodeURIComponent(`Check out this article: ${article.title}\n\n${url}`);
    window.open(`mailto:?subject=${subject}&body=${body}`);
  };

  const shareTwitter = () => {
    const text = encodeURIComponent(`${article.title} ${url}`);
    window.open(`https://twitter.com/intent/tweet?text=${text}`);
  };

  const shareLinkedIn = () => {
    window.open(`https://www.linkedin.com/sharing/share-offsite/?url=${encodeURIComponent(url)}`);
  };

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <button className="flex items-center gap-2 text-gray-500 hover:text-purple-600">
          <Share2 className="h-4 w-4" /> Share
        </button>
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        <DropdownMenuItem onClick={copyLink}>Copy link</DropdownMenuItem>
        <DropdownMenuItem onClick={shareEmail}>Share via email</DropdownMenuItem>
        <DropdownMenuItem onClick={shareTwitter}>Share on Twitter</DropdownMenuItem>
        <DropdownMenuItem onClick={shareLinkedIn}>Share on LinkedIn</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

### 5. Accessibility

Ensure the knowledge base is accessible to all users.

**Keyboard navigation:**
- All interactive elements focusable with Tab
- Enter/Space activates buttons and links
- Escape closes modals and dropdowns
- Arrow keys navigate within menus and search results
- Skip-to-content link at top of page

**Screen reader labels:**
- All images have alt text
- Buttons have aria-label when icon-only
- Form inputs have associated labels
- Status badges have aria-label: "Status: Published"
- Search results announce count: "12 results found"
- Table of contents has aria-label="Table of contents"

**Focus management:**
- Focus moves to search results dropdown when it opens
- Focus returns to trigger when modals close
- Focus trapped inside modals while open
- Visible focus ring on all interactive elements (purple outline)

**Color contrast:**
- All text meets WCAG AA contrast ratios (4.5:1 for normal text, 3:1 for large text)
- Status badges: ensure text is readable on coloured backgrounds
- Don't rely on colour alone — use icons/text alongside colour indicators

**Implementation checklist:**
```typescript
// Skip to content link (in layout)
<a href="#main-content" className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4 focus:bg-purple-600 focus:text-white focus:px-4 focus:py-2 focus:rounded focus:z-50">
  Skip to content
</a>

// Main content landmark
<main id="main-content" role="main">

// Article landmarks
<article role="article" aria-label={article.title}>
<nav aria-label="Breadcrumb">
<nav aria-label="Table of contents">
<nav aria-label="Category navigation">
```

### 6. Mobile-Responsive Layouts

Ensure all views work well on mobile devices.

**Article list (admin):**
- Card view on mobile (instead of table)
- Each card: title + category badge (prominent), status + author + date (secondary)
- Floating action button: "+" to create new article
- Filters collapse into a filter drawer (slide up from bottom)

**Article editor:**
- Full-width editor on mobile (no sidebar)
- Metadata: collapsible bottom sheet (swipe up to access category, tags, status)
- Toolbar: horizontal scroll for editor toolbar buttons

**Public reader:**
- Category sidebar: hidden by default, hamburger menu to toggle
- Table of contents: collapsible, toggle with button above article
- Article body: full-width with comfortable reading margins
- Helpful voting: full-width buttons
- Related articles: vertical stack (single column)

**KB home:**
- Category grid: single column on mobile (stacked cards)
- Search bar: full-width, sticky at top

**Search results:**
- Filters: collapsible drawer
- Results: single column cards
- Pagination: simplified (prev/next only)

**Dashboard (admin):**
- Single column layout
- Metric cards: 2x2 grid on tablet, single column on phone
- Tables: horizontal scroll
- Charts: full-width

**Breakpoints:**
```css
/* Tailwind defaults */
sm: 640px   /* Single column → 2 columns */
md: 768px   /* Sidebar visible, 2-column layouts */
lg: 1024px  /* Full 3-column layout (sidebar + content + TOC) */
xl: 1280px  /* Wide content area */
```

### 7. Reading Mode

Optimised reading experience for article content.

**Reading mode toggle:**
- Button in article header: "Reading mode" (or book icon)
- On toggle: hide sidebar, hide TOC, center content, max-width 680px
- Larger font size (18px body text)
- Increased line height (1.8)
- Distraction-free: hide navigation, show only back button and share

**Implementation:**
```typescript
// src/hooks/useReadingMode.ts

export function useReadingMode() {
  const [isReading, setIsReading] = useState(false);

  useEffect(() => {
    if (isReading) {
      document.body.classList.add('reading-mode');
    } else {
      document.body.classList.remove('reading-mode');
    }
  }, [isReading]);

  return { isReading, toggle: () => setIsReading((prev) => !prev) };
}
```

```css
/* src/index.css */
.reading-mode .kb-sidebar,
.reading-mode .kb-toc {
  display: none;
}

.reading-mode .kb-article {
  max-width: 680px;
  margin: 0 auto;
  font-size: 18px;
  line-height: 1.8;
}
```

### 8. waymaker.config.ts Manifest

Create the manifest file for Host Schema integration.

```typescript
// waymaker.config.ts

export default {
  name: 'Knowledge Base',
  slug: 'knowledge-base',
  type: 'cx',  // Customer-facing
  version: '1.0.0',
  tables: [
    {
      name: 'kb_articles',
      description: 'Knowledge base articles with rich text content, categories, and engagement metrics',
      fields: [
        { name: 'title', type: 'text', required: true },
        { name: 'slug', type: 'text', required: true, unique: true },
        { name: 'content', type: 'text', required: true },
        { name: 'excerpt', type: 'text' },
        { name: 'category_id', type: 'uuid', references: 'kb_categories' },
        { name: 'status', type: 'text', enum: ['draft', 'published', 'archived'] },
        { name: 'author_id', type: 'text' },
        { name: 'helpful_yes', type: 'integer', default: 0 },
        { name: 'helpful_no', type: 'integer', default: 0 },
        { name: 'view_count', type: 'integer', default: 0 },
        { name: 'tags', type: 'text[]' },
        { name: 'sort_order', type: 'integer', default: 0 },
        { name: 'published_at', type: 'timestamptz' },
      ],
    },
    {
      name: 'kb_categories',
      description: 'Categories for organising knowledge base articles with nesting support',
      fields: [
        { name: 'name', type: 'text', required: true },
        { name: 'slug', type: 'text', required: true, unique: true },
        { name: 'description', type: 'text' },
        { name: 'icon', type: 'text' },
        { name: 'colour', type: 'text' },
        { name: 'sort_order', type: 'integer', default: 0 },
        { name: 'parent_id', type: 'uuid', references: 'kb_categories' },
        { name: 'article_count', type: 'integer', default: 0 },
        { name: 'is_active', type: 'boolean', default: true },
      ],
    },
  ],
  commander_tools: ['Tables', 'Documents', 'Search', 'Contacts', 'Automations'],
  routes: {
    public: ['/kb', '/kb/:categorySlug', '/kb/:categorySlug/:slug', '/kb/search', '/kb/request', '/kb/contributors', '/kb/contributors/:authorId'],
    admin: ['/admin/dashboard', '/admin/articles', '/admin/articles/new', '/admin/articles/:id/edit', '/admin/categories'],
  },
};
```

### 9. Final Polish

- [ ] Loading skeletons for all data-fetching views (article list, reader, dashboard)
- [ ] Error boundaries with friendly error messages
- [ ] Toast notifications for all actions (saved, published, archived, deleted, copied link, etc.)
- [ ] Keyboard shortcuts: Cmd+N (new article), Cmd+S (save article), Cmd+K (search)
- [ ] URL-based filters (article list filters in query params for shareable links)
- [ ] 404 page for invalid article/category slugs
- [ ] Print stylesheet for articles (clean, no navigation)
- [ ] Favicon and app title set correctly
- [ ] Verify all CRUD operations work end-to-end
- [ ] Test public reader without auth
- [ ] Test admin routes require auth
- [ ] Deploy-ready configuration

---

## Acceptance Criteria

- [ ] Dashboard loads with 4 metric cards: total articles, total views, helpful rate, articles this month
- [ ] Top 10 articles by views displayed with rank, title, and stats
- [ ] Recent drafts section shows last 5 drafts with "Continue editing" links
- [ ] Low-rated articles section highlights articles with < 50% helpful rate
- [ ] Popular searches section shows top 10 search terms
- [ ] Zero-result searches highlighted as content gaps
- [ ] Bulk select works with checkbox column and "Select All"
- [ ] Bulk publish changes all selected drafts to published
- [ ] Bulk archive works for selected articles
- [ ] Bulk delete works for selected drafts with confirmation dialog
- [ ] Bulk change category reassigns all selected articles
- [ ] SEO meta tags set correctly on article pages (title, description, og:*)
- [ ] Open Graph tags render correct preview cards when shared
- [ ] Canonical URLs set on all article pages
- [ ] Share button with copy link, email, Twitter, LinkedIn options
- [ ] Copy link shows toast "Link copied!"
- [ ] Skip-to-content link present and functional
- [ ] All interactive elements keyboard-accessible
- [ ] Screen reader labels on icons, badges, and form inputs
- [ ] Focus management correct for modals and dropdowns
- [ ] Mobile: card view for article list with floating action button
- [ ] Mobile: editor metadata in bottom sheet
- [ ] Mobile: reader sidebar and TOC collapsible
- [ ] Mobile: full-width search bar
- [ ] Reading mode: centered content, larger text, no distractions
- [ ] waymaker.config.ts manifest created with correct table and route declarations
- [ ] Loading skeletons on all views
- [ ] Toast notifications on all actions
- [ ] 404 page for invalid slugs
- [ ] Deploy succeeds to Waymaker Host
