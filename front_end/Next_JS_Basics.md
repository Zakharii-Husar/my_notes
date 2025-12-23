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
