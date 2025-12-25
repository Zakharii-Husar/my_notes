#### Next.js has two different routers:

* **App Router** : The newer router that supports new React features like Server Components.
* **Pages Router** : The original router, still supported and being improved.

**Differences:**

* App Router = future-facing / where new React features land (Server Components, newer conventions).
* Pages Router = stable, supported, not "dead," but not the "latest-feature-first" path.

If you're starting new work today, choosing **App Router** is still the safest bet for learning and long-term alignment.

**React version handling**

The App Router and Pages Router handle React versions differently:

* **App Router**: Uses React canary releases built-in, which include all the stable React 19 changes, as well as newer features being validated in frameworks, prior to a new React release.
* **Pages Router**: Uses the React version installed in your project's package.json.

**File-system routing**

Next.js uses file-system-based routing, where routes in your application are determined by how you structure your files.

For the **App Router**, create an `app` folder in your project root. Inside the `app` folder, create a `layout.tsx` file - this is the root layout and is required. It must contain the `<html>` and `<body>` tags.

Pages are mapped to routes by creating `page.tsx` files within subfolders of the `app` directory. For example:
- `app/page.tsx` → `/` (home page)
- `app/blog/page.tsx` → `/blog`
- `app/blog/[slug]/page.tsx` → `/blog/[slug]` (dynamic route)

The layout wraps all pages in its directory and subdirectories, providing shared UI like headers and navigation.
