# Document 05: Serverpod Backend & Database Specifications

## 🏛️ 1. Serverpod Workspace Structure (`^4.1.0-beta.1`)

The backend architecture is implemented as a Serverpod monorepo containing three interconnected packages:

```
blogger/
├── blogger_server/    # Backend server app (Dart 3.13 / Serverpod ORM & Endpoints)
├── blogger_client/    # Generated Dart client library
└── blogger_flutter/   # Cross-platform Flutter Studio & Reader client
```

---

## 🗄️ 2. Complete Serverpod Model Specifications (`.spy.yaml`)

### A. `BlogPost` Model (`post.spy.yaml`)
```yaml
class: BlogPost
table: blog_post
fields:
  blogId: int
  authorId: int
  title: String
  slug: String
  schemaType: String? # e.g. "BlogPosting", "Product", "Service", "LocalBusiness"
  jsonLdPayload: String # Full Type-Safe Schema.org JSON-LD Payload
  status: String # "DRAFT", "PUBLISHED", "SCHEDULED"
  publishedDate: DateTime?
  updatedDate: DateTime?
  labels: List<String>?
  searchDescription: String?
  commentCount: int
indexes:
  blog_post_slug_idx:
    fields: blogId, slug
    unique: true
  blog_post_schema_type_idx:
    fields: schemaType
```

### B. `BlogPage` Model (`page.spy.yaml`)
```yaml
class: BlogPage
table: blog_page
fields:
  blogId: int
  authorId: int
  title: String
  slug: String
  schemaType: String? # e.g. "WebPage", "AboutPage", "ContactPage", "ItemPage"
  jsonLdPayload: String
  status: String
  publishedDate: DateTime?
indexes:
  blog_page_slug_idx:
    fields: blogId, slug
    unique: true
```

### C. `BlogSite` Model (`blog.spy.yaml`)
```yaml
class: BlogSite
table: blog_site
fields:
  ownerId: int
  name: String
  subdomain: String
  customDomain: String?
  description: String?
  isPrivate: bool
indexes:
  blog_subdomain_idx:
    fields: subdomain
    unique: true
```

### D. `UserProfile` Model (`user_profile.spy.yaml`)
```yaml
class: UserProfile
table: user_profile
fields:
  userInfoId: int # Linked to Serverpod UserInfo
  displayName: String
  role: String # "SUPER_ADMIN", "VENDOR", "AUTHOR", "READER"
  bio: String?
  avatarUrl: String?
  locationCity: String?
  locationCountry: String?
```

### E. `Comment` Model (`comment.spy.yaml`)
```yaml
class: Comment
table: comment
fields:
  blogId: int
  postId: int
  authorId: int?
  authorName: String
  authorEmail: String
  content: String
  publishedDate: DateTime
  isApproved: bool
  isSpam: bool
  parentId: int? # Threaded replies
```

### F. `BlogStat` Model (`blog_stat.spy.yaml`)
```yaml
class: BlogStat
table: blog_stat
fields:
  blogId: int
  pageviews: int
  uniqueVisitors: int
  referrerUrl: String?
  countryCode: String?
  recordedDate: DateTime
```

### G. `BlogEarning` Model (`blog_earning.spy.yaml`)
```yaml
class: BlogEarning
table: blog_earning
fields:
  blogId: int
  publisherId: String # Google AdSense `pub-XXXXXXXXXXXXXXXX`
  estimatedRevenue: double
  impressions: int
  clicks: int
  recordedDate: DateTime
```

### H. `LayoutWidget` Model (`layout_widget.spy.yaml`)
```yaml
class: LayoutWidget
table: layout_widget
fields:
  blogId: int
  section: String # "HEADER", "SIDEBAR", "MAIN", "FOOTER"
  widgetType: String # "HTML_GADGET", "LABEL_LIST", "FEATURED_POST", "SEARCH_BAR"
  title: String
  configJson: String
  displayOrder: int
```

---

## 🔌 3. Google Blogger API v3 REST Specification Endpoints (`BloggerV3Endpoint`)

To guarantee seamless backward compatibility with existing Blogger tooling, mobile SDKs, and feed processors, Serverpod implements the `BloggerV3Endpoint`:

| Endpoint Route | Serverpod Protocol Method | Description |
| :--- | :--- | :--- |
| `GET /blogger/v3/users/{userId}` | `usersGet` | Retrieves user profile metadata. |
| `GET /blogger/v3/users/{userId}/blogs/{blogId}` | `blogUserInfosGet` | Retrieves blog info for a specific user. |
| `GET /blogger/v3/blogs/{blogId}` | `blogsGet` | Fetches blog site details by ID. |
| `GET /blogger/v3/blogs/byurl?url={url}` | `blogsGetByUrl` | Resolves blog site by subdomain or custom domain. |
| `GET /blogger/v3/blogs/{blogId}/posts` | `postsList` | Returns paginated blog posts. |
| `GET /blogger/v3/blogs/{blogId}/posts/{postId}` | `postsGet` | Retrieves single post by ID or slug. |
| `POST /blogger/v3/blogs/{blogId}/posts` | `postsInsert` | Publishes new post with JSON-LD validation. |
| `DELETE /blogger/v3/blogs/{blogId}/posts/{postId}` | `postsDelete` | Soft-deletes or purges a post. |
| `GET /blogger/v3/blogs/{blogId}/posts/search?q={q}` | `postsSearch` | Performs full-text search on titles, labels, & JSON-LD. |
| `GET /blogger/v3/blogs/{blogId}/pages/{pageId}` | `pagesGet` | Fetches static page content. |
| `GET /blogger/v3/blogs/{blogId}/posts/{postId}/comments` | `commentsList` | Fetches post comment threads. |

---

## ⚡ 4. Database Indexing & Migration Strategy

```sql
-- Fast lookup index for post routes (/blog/:blogId/post/:postId)
CREATE UNIQUE INDEX blog_post_slug_idx ON blog_post (blog_id, slug);

-- Fast lookup index for Schema.org @type queries
CREATE INDEX blog_post_schema_type_idx ON blog_post (schema_type);

-- Fast lookup index for subdomain routing
CREATE UNIQUE INDEX blog_subdomain_idx ON blog_site (subdomain);
```

Serverpod manages migrations via `serverpod create-migration`. Production server boot sequence applies non-destructive schema migrations automatically.
