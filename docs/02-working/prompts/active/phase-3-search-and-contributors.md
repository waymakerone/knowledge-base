---
sync:
  type: doc
  layer: Knowledge Base
build:
  status: todo
  phase: 3
  priority: P0
  depends_on: ["phase-2-public-reader"]
  started_at: null
  completed_at: null
---

# Phase 3: Search & Contributors

**Goal:** Full-text search across published articles with instant results, filters, and highlighting. Contributor profiles showing authors and their articles. Article request form that creates a Contact and triggers an automation email. Tag filtering across the knowledge base.

**PRD Reference:** `docs/01-planning/product-requirements/knowledge-base-prd.md` — Phase 3

---

## What to Build

### 1. Full-Text Search

Implement search across all published articles using Commander Search.

**Search bar (global):**
- Persistent search bar in the KB header on all public pages
- Placeholder: "Search articles..."
- Keyboard shortcut: `Cmd+K` or `/` to focus
- Instant results dropdown as user types (debounced 300ms)
- Press Enter or click "See all results" → navigate to `/kb/search?q=...`

**Instant search dropdown:**
```typescript
// src/components/SearchBar.tsx

interface SearchResult {
  id: string;
  title: string;
  excerpt: string;
  category_name: string;
  category_slug: string;
  slug: string;
  highlight: string; // Snippet with match highlighted
}

function SearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isOpen, setIsOpen] = useState(false);

  // Debounced search
  useEffect(() => {
    if (query.length < 2) { setResults([]); return; }
    const timer = setTimeout(() => searchArticles(query), 300);
    return () => clearTimeout(timer);
  }, [query]);

  return (
    <div className="relative">
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search articles..."
        className="w-full px-4 py-2 border rounded-lg"
        onFocus={() => setIsOpen(true)}
      />
      {isOpen && results.length > 0 && (
        <div className="absolute top-full mt-1 w-full bg-white border rounded-lg shadow-lg z-50">
          {results.slice(0, 5).map((result) => (
            <a
              key={result.id}
              href={`/kb/${result.category_slug}/${result.slug}`}
              className="block px-4 py-3 hover:bg-purple-50"
            >
              <div className="font-medium text-gray-900">{result.title}</div>
              <div className="text-sm text-gray-500">{result.category_name}</div>
              <div
                className="text-sm text-gray-600 mt-1"
                dangerouslySetInnerHTML={{ __html: result.highlight }}
              />
            </a>
          ))}
          <a
            href={`/kb/search?q=${encodeURIComponent(query)}`}
            className="block px-4 py-3 text-purple-600 font-medium border-t"
          >
            See all results for "{query}"
          </a>
        </div>
      )}
    </div>
  );
}
```

**Search API call:**
```typescript
// src/services/search.ts

export async function searchArticles(query: string, filters?: {
  category_id?: string;
  tags?: string[];
  sort?: 'relevance' | 'date' | 'views' | 'helpful';
}): Promise<SearchResult[]> {
  const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-search`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      action: 'search',
      data: {
        table: 'kb_articles',
        query,
        filters: {
          status: 'published',
          ...filters,
        },
        fields: ['title', 'content', 'tags', 'excerpt'],
        highlight: true,
      },
    }),
  });
  const json = await res.json();
  return json.data;
}
```

### 2. Search Results Page (/kb/search)

Full search results page with filters and sorting.

**Layout:**
- Search input (pre-filled with query from URL)
- Filter sidebar (left, 25%)
- Results list (right, 75%)

**Filter sidebar:**
- Category filter: checkboxes for each active category with article count
- Tags filter: tag cloud or checklist of common tags
- Date filter: Published in last week, month, 3 months, year, all time

**Results list:**
- Result count: "12 results for 'password reset'"
- Each result card:
  - Title (link to article, with search term bolded)
  - Category badge (coloured)
  - Excerpt with highlighted search terms (use `<mark>` tags)
  - Author name
  - Published date
  - View count + helpful percentage
  - Tags (small badges)
- Pagination: 10 results per page, numbered pagination at bottom

**Sort options (top right):**
- Relevance (default — search engine ranking)
- Newest first (published_at descending)
- Most viewed (view_count descending)
- Most helpful (helpful percentage descending)

**No results state:**
- "No articles found for '[query]'"
- Suggestions: "Try different keywords", "Browse by category", "Request an article"
- Link to article request form

**Highlight implementation:**
```typescript
// src/lib/highlight.ts

export function highlightMatches(text: string, query: string): string {
  if (!query) return text;
  const words = query.split(/\s+/).filter(Boolean);
  let highlighted = text;
  words.forEach((word) => {
    const regex = new RegExp(`(${escapeRegex(word)})`, 'gi');
    highlighted = highlighted.replace(regex, '<mark class="bg-purple-100 text-purple-900 rounded px-0.5">$1</mark>');
  });
  return highlighted;
}

function escapeRegex(str: string): string {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```

### 3. Tag System

Tags on articles for cross-category organisation and filtering.

**Tag display on articles:**
- Tags shown as small badges (pill-shaped, grey background, purple text)
- Click a tag → navigate to search filtered by that tag: `/kb/search?tag=onboarding`

**Tag input in article editor (enhance from Phase 1):**
- Autocomplete: suggest existing tags as user types
- Create new tags by typing and pressing Enter or comma
- Remove tags with X button or backspace
- Display as pills in the input field

**Tag cloud on search page:**
- Show top 20 most-used tags
- Size based on frequency (larger = more articles)
- Click → filter search results by tag

**Tag filter implementation:**
```typescript
// src/services/tags.ts

export async function getPopularTags(token?: string): Promise<{ tag: string; count: number }[]> {
  // Query all published articles, extract tags, count frequency
  const articles = await listPublishedArticles();
  const tagCounts: Record<string, number> = {};
  articles.forEach((article) => {
    (article.tags || []).forEach((tag) => {
      tagCounts[tag] = (tagCounts[tag] || 0) + 1;
    });
  });
  return Object.entries(tagCounts)
    .map(([tag, count]) => ({ tag, count }))
    .sort((a, b) => b.count - a.count);
}
```

### 4. Contributor Profiles

Show article authors with their profiles and article lists.

**Contributors page** (`/kb/contributors`):
- Grid of contributor cards
- Each card: avatar, name, role/title, article count
- Click → contributor detail

**Contributor detail** (`/kb/contributors/:authorId`):
- Author avatar (from Clerk user data)
- Full name
- Role/title (if available from Contacts)
- Bio (if available from Contacts)
- Article count
- List of their published articles (same card format as category page)

**Author data sources:**
```typescript
// src/services/contributors.ts

interface Contributor {
  id: string;
  name: string;
  avatar_url: string | null;
  role: string | null;
  bio: string | null;
  article_count: number;
}

export async function getContributors(): Promise<Contributor[]> {
  // 1. Get unique author_ids from published articles
  const articles = await listPublishedArticles();
  const authorIds = [...new Set(articles.map((a) => a.author_id))];

  // 2. Look up each author in Commander Contacts
  const contributors: Contributor[] = [];
  for (const authorId of authorIds) {
    const contact = await getContactByClerkId(authorId);
    const authorArticles = articles.filter((a) => a.author_id === authorId);
    contributors.push({
      id: authorId,
      name: contact?.name || 'Unknown Author',
      avatar_url: contact?.avatar_url || null,
      role: contact?.role || null,
      bio: contact?.bio || null,
      article_count: authorArticles.length,
    });
  }
  return contributors;
}
```

**Author byline on articles:**
- Show author name + avatar on article reader page (already in Phase 2 header)
- Make author name a link to `/kb/contributors/:authorId`

### 5. Article Request Form

Allow visitors to request new articles on topics not yet covered.

**Request form** (accessible from search "no results" page and KB footer):
- Name (text input, required)
- Email (email input, required)
- Topic (text input, required — "What would you like to learn about?")
- Details (textarea, optional — "Tell us more about what you need")
- Submit button: "Request Article"

**On submit:**
1. Create a Contact in Commander Contacts:
```typescript
await fetch(`${SUPABASE_URL}/functions/v1/commander-contact-operations`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`, // Public anon key or service key
  },
  body: JSON.stringify({
    action: 'create',
    data: {
      name: formData.name,
      email: formData.email,
      source: 'knowledge-base-request',
      notes: `Article request: ${formData.topic}\n\n${formData.details}`,
    },
  }),
});
```

2. Trigger an automation email to the KB admin:
```typescript
await fetch(`${SUPABASE_URL}/functions/v1/commander-automation-operations`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({
    action: 'trigger',
    data: {
      event: 'article_request',
      payload: {
        requester_name: formData.name,
        requester_email: formData.email,
        topic: formData.topic,
        details: formData.details,
      },
    },
  }),
});
```

3. Show confirmation: "Thanks! We'll review your request and get back to you."

**Request form placement:**
- Link in KB footer: "Can't find what you're looking for? Request an article."
- Button on search "no results" page
- Standalone route: `/kb/request`

### 6. Search Analytics (Simple)

Track what people are searching for to inform content strategy.

**Implementation — lightweight, localStorage-based:**
```typescript
// src/lib/search-analytics.ts

interface SearchEvent {
  query: string;
  results_count: number;
  timestamp: string;
}

export function logSearch(query: string, resultsCount: number): void {
  // For MVP: store in localStorage for admin viewing
  const events: SearchEvent[] = JSON.parse(
    localStorage.getItem('kb_search_analytics') || '[]'
  );
  events.push({
    query,
    results_count: resultsCount,
    timestamp: new Date().toISOString(),
  });
  // Keep last 100 searches
  localStorage.setItem('kb_search_analytics', JSON.stringify(events.slice(-100)));
}

export function getPopularSearches(): { query: string; count: number }[] {
  const events: SearchEvent[] = JSON.parse(
    localStorage.getItem('kb_search_analytics') || '[]'
  );
  const counts: Record<string, number> = {};
  events.forEach((e) => {
    const normalised = e.query.toLowerCase().trim();
    counts[normalised] = (counts[normalised] || 0) + 1;
  });
  return Object.entries(counts)
    .map(([query, count]) => ({ query, count }))
    .sort((a, b) => b.count - a.count);
}
```

**Admin view (on dashboard — Phase 4):**
- Top 10 searched terms
- Searches with zero results (content gaps)

---

## Acceptance Criteria

- [ ] Search bar appears on all public KB pages
- [ ] Instant search dropdown shows top 5 results as user types (debounced 300ms)
- [ ] Search highlights matching terms in title and excerpt
- [ ] Cmd+K or / focuses the search bar
- [ ] Search results page shows filtered results with pagination (10 per page)
- [ ] Category filter on search results works (checkbox multi-select)
- [ ] Tag filter on search results works
- [ ] Date filter on search results works
- [ ] Sort options: relevance, newest, most viewed, most helpful
- [ ] No results state shows helpful suggestions and link to request form
- [ ] Tag cloud shows top 20 tags on search page
- [ ] Click a tag filters results by that tag
- [ ] Tag autocomplete works in article editor
- [ ] Contributors page shows authors with avatars, names, and article counts
- [ ] Contributor detail page shows author bio and their articles
- [ ] Author name on article reader links to contributor profile
- [ ] Article request form captures name, email, topic, details
- [ ] Request creates a Contact in Commander Contacts
- [ ] Request triggers automation email notification
- [ ] Confirmation message shown after submitting request
- [ ] Search analytics logged to localStorage
- [ ] Popular searches available for dashboard (Phase 4)
