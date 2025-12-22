# React Rendering Strategies: Mental Model (HTML/JS/CSS + Hydration)

## When to Care About the Buckets (Practical Guide)

**You don't need to obsess over them day-to-day**, but keep buckets in mind: Next.js "handles it for you" based on your code choices (data fetching, caching, cookies/auth, client components). Small choices can quietly push routes into different buckets.

### When You Can Mostly Ignore It
For simple sites (portfolio/blog/marketing, mostly static CMS content, little personalization):
- Treat as **SSG/ISR by default**
- Sprinkle Client Components for interactivity
- You'll be fine without thinking about buckets.

### When You Must Think About It
Care about buckets when you need:
- **SEO/social previews**: Content/metadata must be server-generated (Server Components/Next metadata), not client-only JS
- **Performance & cost**: SSR-every-request can be slower/more expensive than cached/static
- **Freshness**: CMS updates instantly (SSR), every minute (ISR revalidate), or only on redeploy (SSG)?
- **Personalization**: Cookies/auth/user data usually means SSR territory (or keep personalization client-side)

### Next.js Rule-of-Thumb (3 Questions Per Route)
1. **Is the page public and should it rank?** Yes → ensure content + metadata are server-generated
2. **Does the content change often?** Rarely → SSG | Sometimes → ISR (revalidate) | Always/per-user → SSR
3. **Is it personalized per user?** Yes → SSR (or keep personalization client-side while leaving core content static)

### Why Buckets Still Help
Even with Next.js defaults, buckets are your "debug model":
- "Why is this page slow?" → accidentally became SSR
- "Why isn't my CMS update showing?" → SSG with no revalidate
- "Why is SEO text missing?" → client-only fetching

## Quick Summary

**CSR:** browser creates the HTML.

**SSR:** server creates the HTML on each request.

**SSG:** build step creates the HTML once.

**ISR:** like SSG, but occasionally rebuilds HTML after deploy.

**Hydration:** only matters when HTML came first (SSR/SSG/ISR), and it "activates" that HTML with React JS.

---

## The 4 SEO "rendering buckets"

Every React/Next setup is basically one of these:

* ###### CSR SPA (Client-Side Rendering) — uses JavaScript to build and display the website's content on the client's side
* ###### SSR (Server-Side Rendering) — renders full HTML on the server for each request, then hydrates on the client
* ###### SSG (Static Site Generation) — pre-builds all pages as static HTML files at build time, served from CDN (Content Delivery Network)
* ###### ISR (Incremental Static Regeneration) — combines SSG (3) and SSR (2) with on-demand revalidation and smart caching

---

BUILD TIME (always happens)
────────────────────────────────────────────────────────────────────
Your code (TS/JS + JSX + CSS/SCSS)
        │
        │ transpile (TS->JS, JSX->JS)  [SWC/Babel/tsc]
        ▼
JS modules + CSS
        │
        │ bundle (resolve imports, tree-shake, code-split, minify)
        │ [Webpack / Turbopack / Vite / etc.]
        ▼
STATIC ASSETS:

- JS bundles (include React runtime + your client code when needed). React is still React in the browser: JSX/TS transpiled to JS, but bundle includes React runtime implementing components, hooks, reconciliation, hydration, and event system.

React runtime size is usually okay: minified + compressed (gzip/brotli), tree-shaking removes unused exports, code-splitting loads only needed JS per route. Next.js App Router reduces bundle: Server Components ship no client JS, only Client Components add to browser bundle.
- CSS files
- images/fonts/etc

Now the 4 buckets differ mainly in: "Do we also produce HTML ahead of time,
or do we create HTML per request, or do we let the browser create HTML?"

---

## 1) CSR SPA (classic React/Vite/CRA)

────────────────────────────────────────────────────────────────────
SERVER sends FIRST:   HTML shell (mostly empty) + `<script src=app.js>` + CSS
BROWSER then:         downloads JS → React renders DOM from scratch
INTERACTIVITY:        available after JS runs
HYDRATION:            not really (no server React HTML to reuse)

Flow:
Request → [HTML shell] → download JS/CSS → React builds UI → interactive

---

**SEO outcome:**

- Google can index (often), but can be slower/less reliable for edge cases
- Many crawlers/OG (Open Graph) social previews may see almost nothing
- Use it when: app-like pages behind login, dashboards, internal tools
- Big SEO footgun: fetching your actual page content only on the client.

---

**Pros:**

- Simple hosting (CDN/static hosting works)
- Fast navigation after initial load (client-side routing)
- Great developer experience for "app" UIs (stateful, interactive)
- Easier to cache/deploy (just static assets)

**Cons:**

- SEO can be weaker/less predictable (especially for non-Google bots and social previews)
- Slow "first meaningful content" on weak devices or slow networks (JS has to load/execute)
- If your main content depends on client fetching, initial HTML may be empty/placeholder
- Performance bottleneck risk: large JS bundle and hydration cost

---

## 2) SSR (Server-Side Rendering)

────────────────────────────────────────────────────────────────────
GENERATED AT REQUEST TIME (each request):
SERVER does:           run React on server for this URL → generate HTML
SERVER sends FIRST:    full HTML (content visible) + JS/CSS links
BROWSER then:          downloads JS → hydrates client parts (events/state)
INTERACTIVITY:         after hydration of those parts
HYDRATION:             yes (reuse server HTML + attach React behavior)

Flow:
Request → server renders HTML → [HTML content shown] → download JS/CSS → hydrate

---

**SEO outcome:** strong, consistent (even for bots that don't run JS)

**Use it when:** pages are personalized, frequently changing, or require request-time logic.

---

**Pros:**

- Strong SEO reliability (content is in initial HTML)
- Good perceived performance (content shows up immediately)
- Works well for dynamic/personalized pages
- Social previews/meta scraping is more reliable (server can output correct tags/content)

**Cons:**

- Higher server cost/complexity (rendering per request unless heavily cached). SSR is viable because you often don't render for every request: most served from cache layers (CDN/edge HTML caching, data caching in memory/platform/framework caches). Redis optional for small sites.
- Slower TTFB possible compared to pure static (depends on server + data fetching)
- More moving parts (caching, load spikes, cold starts if serverless)
- Still pays hydration cost in the browser for interactive parts

---

## 3) SSG (Static Site Generation)

────────────────────────────────────────────────────────────────────
GENERATED AT BUILD TIME (once per deploy):
BUILD step does:       run React for each route → pre-generate HTML files
SERVER/CDN sends FIRST: prebuilt full HTML + JS/CSS links
BROWSER then:          downloads JS → hydrates client parts
HYDRATION:             yes (same client behavior as SSR)

Flow:
Build → generate HTML files → Request → [HTML content shown] → download JS/CSS → hydrate

---

**SEO outcome:** strongest + fastest + cheapest to serve

**Use it when:** marketing pages, docs, blogs, portfolio, product pages that don't change every minute.

---

**Pros:**

- Best performance for most users (served from CDN as static files)
- Excellent SEO (full HTML instantly)
- Very cheap and scalable hosting
- High reliability (no server rendering required at request time)

**Cons:**

- Content can become stale until you rebuild/redeploy (unless using ISR)
- Build time can get long for very large sites (many pages)
- Not great for per-user personalization (unless you add client-side logic)
- Any truly real-time data usually needs client fetching (which can reintroduce SEO risk if it's core content)

---

## 4) ISR (Incremental Static Regeneration / Hybrid)

────────────────────────────────────────────────────────────────────
Mostly like SSG, but HTML can refresh after deploy:

- first request (or after timeout): server regenerates HTML
- otherwise: serve cached HTML

SERVER/CDN sends FIRST: cached/prebuilt full HTML + JS/CSS links
BROWSER then:          downloads JS → hydrates client parts
HYDRATION:             yes

Flow:
Build → HTML generated → Request → [cached HTML shown]
  ↳ occasionally: regenerate HTML → update cache → future requests get new HTML
  then: download JS/CSS → hydrate

---

**SEO outcome:** usually excellent if the initial HTML contains the content.

---

**Pros:**

- Best of both worlds: static speed with controlled freshness
- Scales well (most requests hit cached/static output)
- Good SEO if the server output contains the content
- Flexible: choose per route (static for some, dynamic for others)

**Cons:**

- More complexity (cache rules, revalidation timing, stale-while-revalidate behavior)
- Potential for serving slightly stale content by design (depends on settings)
- Debugging can be harder (why did this page re-generate now vs later?)
- Still possible to mess up SEO if you rely on client-only fetching for core content

---

### What is Hydration?

**Hydration:** the step where the browser loads React's JavaScript and "activates" already-rendered HTML (from the server/build) by attaching event handlers/state so the page becomes interactive. It applies to SSR/SSG/ISR (hybrid), not a classic CSR SPA, where React renders from scratch in the browser.

---

**Hydration process:** When HTML comes from the server (SSR/SSG/ISR), React runs in the browser to "activate" the static HTML by:

1. Rebuilding its internal UI model (React tree)
2. Walking the existing DOM and matching nodes by structure/order (tags, nesting, key text/attributes). DOM nodes already exist: after HTML arrives, browser parses it into DOM tree (Element/Text nodes) before React runs. Visible in DevTools Elements; `document.querySelector()` returns these pre-existing nodes.
3. Attaching event handlers and component state to make the UI interactive

**Important:** Hydration is pure CPU work, not a network operation. It happens after the HTML and JS are already loaded. The network part is just downloading the JS bundle initially (or fetching it from cache).

**Hydration success:** "DOM already matches → just attach handlers/state."

**Attach handlers:** React ensures its root event listeners are set up (event delegation). Because browser events bubble, React can "intercept" clicks/inputs at the root and route them to the correct component's onClick/onChange without adding listeners to every element.

**Attach state:** React creates its in-memory component/hook state (e.g. useState) and links that component instance to the already-existing DOM nodes it controls.

**Hydration mismatch:** "DOM doesn't match → rebuild/patch DOM to match, then attach handlers/state."

If tags/text/structure differ (e.g. Date.now(), Math.random(), locale differences, data mismatch), React can't safely adopt the DOM.

It falls back to client rendering for that subtree: it patches small differences (text/attributes) or replaces larger mismatching parts so the real DOM matches what the client render expects, then it attaches the same delegated handlers + state as above.

---

### Next.js Specifics

**CMS data fetching (SSG vs SSR):**
- SSG: CMS fetch during build (or ISR regeneration), HTML cached as static. ISR refreshes static pages post-deploy.
- SSR: CMS fetch at request time (unless cached).
- Next.js App Router: fetch() responses not cached by default, but HTML can be pre-rendered/cached. Opt into data caching with `{ cache: 'force-cache' }` or `{ next: { revalidate: N } }`.

**Next.js choosing SSG vs SSR:**
- App Router uses heuristics based on API usage.
- `dynamic = 'auto'` (default): cache as much as possible without blocking dynamic behavior.
- Dynamic APIs (cookies/headers/search params) or uncached fetching (`cache: 'no-store'`) → dynamic/SSR behavior.
- Cacheable data → SSG/ISR behavior (pre-render + cached output).
- Override with route config (dynamic, revalidate, fetchCache).

**React vs Next.js differences:**
- React alone: naturally CSR; wire own server/build pipeline for SSR/SSG/ISR using React's server rendering + hydrateRoot.
- Next.js: first-class SSR/SSG/ISR features + routing, bundling, deployment defaults.
- Next.js adds Server Components + RSC payload; hydrates Client Components for interactivity.
- Next.js has built-in caching (request memoization, data cache, full route cache, router cache) so "SSR" often isn't from scratch.

**Build vs Render clarification:**
- Build (compile/bundle): TS/JSX → JS, bundle → optimized JS/CSS/assets. Always needed for all buckets.
- Render (generate HTML): React running on server to produce HTML. Needed for SSR/SSG/ISR.
- SSG: render-to-HTML at build time → write HTML files.
- SSR: render-to-HTML at request time → send HTML response.
- Same rendering mechanism, different schedule.

---

### Batching

**Batching:** React groups multiple state updates into a single render pass to avoid unnecessary re-renders and DOM work.
