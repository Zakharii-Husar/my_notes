# Next.js App Router: Complete File & Routing System

## Core Concept
- **Folder names = URL segments**
- **`page.tsx` = the page component for that URL**
- **Route organization**: Nest folders, each can have its own `page.tsx`

## Routing Files Overview
Next.js App Router uses special files in the `app/` directory:

| File | Extension | Purpose |
|------|-----------|---------|
| `layout` | `.js .jsx .tsx` | Shared UI (header, nav, footer) for route segments |
| `page` | `.js .jsx .tsx` | Page component - defines the actual route |
| `loading` | `.js .jsx .tsx` | Loading UI/skeletons |
| `not-found` | `.js .jsx .tsx` | 404 UI for undefined routes |
| `error` | `.js .jsx .tsx` | Error boundary for route segments |
| `global-error` | `.js .jsx .tsx` | Global error UI for entire application |
| `route` | `.js .ts` | API endpoints |
| `template` | `.js .jsx .tsx` | Re-rendered layout (unlike layout, re-mounts on navigation) |
| `default` | `.js .jsx .tsx` | Fallback page for parallel routes |

## File Location & Scope

### Where to Place Special Files
**Special files like `loading.tsx`, `not-found.tsx`, `error.tsx`, etc. are placed in the SAME directory as `page.tsx`.**

They apply to:
- **That specific route segment** and all its child routes
- **Nested routes inherit** from parent directories unless overridden

### Examples:

**Route-specific loading & error handling**:
```
app/
  dashboard/
    loading.tsx      # Shows during dashboard page loads
    error.tsx        # Handles errors in dashboard and sub-routes
    page.tsx         # /dashboard
    settings/
      page.tsx       # /dashboard/settings (inherits loading/error from parent)
```

**Override in child routes**:
```
app/
  blog/
    loading.tsx      # General blog loading
    [slug]/
      loading.tsx    # Specific loading for individual posts
      page.tsx       # /blog/[slug]
```

### Reusable Components (404, Loading, etc.)

**For reusable 404 pages**, create shared components:

```tsx
// components/NotFound.tsx
export default function NotFound() {
  return <div>404 - Page Not Found</div>;
}
```

**Global 404 (root level)**:
```
app/
  not-found.tsx     # Global 404 for entire app
```

**Import in route-specific files**:
```tsx
// app/blog/not-found.tsx
import NotFound from '@/components/NotFound';

export default function BlogNotFound() {
  return <NotFound />; // Reusable component
}
```

**Or create custom per-route**:
```tsx
// app/admin/not-found.tsx
export default function AdminNotFound() {
  return <div>Admin page not found. <a href="/admin">Back to admin</a></div>;
}
```

### Inheritance Rules
- **Layouts**: Inherited by child routes
- **Loading/Error/Not-found**: Can be overridden in child directories
- **Global-error**: Only at root level for entire app
- **Templates**: Re-mount on navigation (unlike layouts)

---

## How `page.tsx` Works

### 1. Basic Routes

**Root route (`/`)**:
```
app/
  page.tsx
```
➡️ `http://localhost:3000/`

**Static route (`/about`)**:
```
app/
  about/
    page.tsx
```
➡️ `http://localhost:3000/about`

### 2. Nested Routes

**Multiple levels (`/dashboard/settings`)**:
```
app/
  dashboard/
    page.tsx        # → /dashboard
    settings/
      page.tsx      # → /dashboard/settings
```
➡️ `/dashboard` and `/dashboard/settings`

### 3. Dynamic Routes

**Single dynamic segment (`/blog/[slug]`)**:
```
app/
  blog/
    [slug]/
      page.tsx
```
➡️ `/blog/anything-here`, `/blog/my-post`, etc.

Access params in `page.tsx`:
```tsx
export default function Page({ params }: { params: { slug: string } }) {
  return <div>Post: {params.slug}</div>;
}
```

**Multiple dynamic params (`/shop/[category]/[productId]`)**:
```
app/
  shop/
    [category]/
      [productId]/
        page.tsx
```
➡️ `/shop/shoes/123`, `/shop/electronics/456`

### 4. Catch-All Routes

**Catch-all (`/docs/[...slug]`)**:
```
app/
  docs/
    [...slug]/
      page.tsx
```
➡️ `/docs/a`, `/docs/a/b`, `/docs/a/b/c`

**Optional catch-all (`/docs/[[...slug]]`)**:
```
app/
  docs/
    [[...slug]]/
      page.tsx
```
➡️ `/docs`, `/docs/a`, `/docs/a/b/c`

### 5. Route Groups (Organization Only)

**Group routes without affecting URL**:
```
app/
  (marketing)/
    pricing/
      page.tsx      # → /pricing
  (app)/
    dashboard/
      page.tsx      # → /dashboard
```
➡️ `/pricing` and `/dashboard` (no `(marketing)` or `(app)` in URLs)

### 6. API Routes

**API endpoints with `route.ts`**:
```
app/
  api/
    health/
      route.ts       # → /api/health
```
```ts
// route.ts
export async function GET() {
  return Response.json({ status: 'ok' });
}
```

---

## Quick Reference

### Want a page at `/x/y`?
Create: `app/x/y/page.tsx`

### Need dynamic segments?
Use: `app/x/[id]/page.tsx`

### Multiple dynamics?
Use: `app/shop/[category]/[productId]/page.tsx`

### Catch-all paths?
Use: `app/docs/[...slug]/page.tsx`

### Optional catch-all?
Use: `app/docs/[[...slug]]/page.tsx`

### API endpoint?
Create: `app/api/endpoint/route.ts`

### Organize without URL impact?
Use: `app/(group)/route/page.tsx`

---

## Important Notes
- You generally don't put `page.tsx` and `route.ts` in the same folder for the same path
- Folder structure directly maps to URL structure
- Dynamic segments use square brackets: `[param]`
- Route groups use parentheses: `(group)` - not part of URL

---

## Advanced Routing Features

### Layouts & Public Route Rule

**"Public route" rule**: A route becomes public only if it contains `page.tsx` (UI route) or `route.ts` (API route).

**Layouts wrap child segments**: `layout.tsx` at any level wraps all child segments below it (nested layouts stack).

```txt
app/
  layout.tsx              # Root layout (wraps everything)
  page.tsx                # → / (public route)

  blog/
    layout.tsx            # Wraps /blog and all descendants
    page.tsx              # → /blog (public route)

    authors/
      page.tsx            # → /blog/authors (public route)

    _components/          # Private folder (not routable)
      Post.tsx            # Not a route
```

### Private Folders (_folder)

**Private folders** start with `_` and are safe places to colocate code that should NOT become route segments.

```txt
app/
  blog/
    _components/          # Not routable
      Post.tsx
      CommentForm.tsx

    _lib/                 # Not routable
      data.ts
      utils.ts

    [slug]/
      page.tsx            # → /blog/[slug] (public)
```

### Parallel Routes (@slot) & default.tsx

**Parallel routes** use `@slot` folders to render multiple "areas" from a parent layout (slot-based UI).

```txt
app/dashboard/
  layout.tsx              # Parent layout renders @sidebar and @main
  page.tsx                # → /dashboard

  @sidebar/               # Named slot (not part of URL)
    page.tsx              # Sidebar content

  @main/                  # Named slot (not part of URL)
    page.tsx              # Main content

  @sidebar/
    default.tsx           # Fallback if sidebar slot not provided
```

### Intercepted Routes ((.), (..), etc.)

**Intercepted routes** intercept another route and render it inside the current layout (modal/overlay routing).

**Patterns**:
- **(.)folder**: Intercept same level
- **(..)folder**: Intercept parent level
- **(..)(..)folder**: Intercept two levels up
- **(...)folder**: Intercept from root

```txt
app/
  items/
    page.tsx             # → /items (list view)
    [id]/
      page.tsx           # → /items/[id] (detail page)

  dashboard/
    items/
      (.)[id]/
        page.tsx         # Intercept /items/[id] as modal over /dashboard/items
```

**Typical use**: Show `/items/123` as a modal over `/items` without leaving the list UI.

### Metadata File Conventions

**App Icons**:
```
app/
  favicon.ico                    # Favicon file
  icon.png                       # App icon (static)
  icon.tsx                       # Generated app icon
  apple-icon.png                 # Apple app icon (static)
  apple-icon.tsx                 # Generated Apple app icon
```

**Social Preview Images**:
```
app/
  opengraph-image.jpg            # Open Graph image (static)
  opengraph-image.tsx            # Generated Open Graph image
  twitter-image.png              # Twitter image (static)
  twitter-image.tsx              # Generated Twitter image
```

**SEO Files**:
```
app/
  sitemap.xml                    # Sitemap file (static)
  sitemap.ts                     # Generated sitemap
  robots.txt                     # Robots file (static)
  robots.ts                      # Generated robots file
```

---

## Complete Route Analysis Example

Given this `app/` structure:
```
app/
  layout.tsx
  page.tsx
  sitemap.ts

  (marketing)/
    about/
      page.tsx
    pricing/
      page.tsx

  dashboard/
    layout.tsx
    page.tsx

    @sidebar/
      page.tsx
      default.tsx

    @main/
      analytics/
        page.tsx

  blog/
    _components/
      Post.tsx
    _lib/
      data.ts

    layout.tsx
    page.tsx

    [slug]/
      page.tsx

    authors/
      (..)(..)items/
        [id]/
          page.tsx
```

**Public routes**:
- `/` (has page.tsx)
- `/about` (has page.tsx, group omitted)
- `/pricing` (has page.tsx, group omitted)
- `/dashboard` (has page.tsx)
- `/dashboard/analytics` (has page.tsx in @main slot)
- `/blog` (has page.tsx)
- `/blog/[slug]` (has page.tsx)

**Not routable**:
- `_components/`, `_lib/` folders
- `@sidebar/`, `@main/` slots (not part of URL)

**Layouts wrap**:
- Root `layout.tsx` wraps everything
- `dashboard/layout.tsx` wraps `/dashboard/*`
- `blog/layout.tsx` wraps `/blog/*`

**Intercepted route**:
- `authors/(..)(..)items/[id]/page.tsx` intercepts from two levels up
