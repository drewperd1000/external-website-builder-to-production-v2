# Prerendered Directory Resolution + Windows Worktree Gotcha

> **🤖 This file is for Claude only.** It documents the Express middleware that resolves `<path>/index.html` for prerendered Astro blog routes, and why `dotfiles: "allow"` is needed when dev paths contain `.worktrees/` or any dot-segment.

## When to read this file

I read this file:
- When wiring [`stage-3-express-server.md`](../stage-3-express-server.md)'s middleware order on an Astro project.
- When debugging "404 on /blog/<slug>/" responses when the file clearly exists at `dist/client/blog/<slug>/index.html`.
- When the user reports "the blog works in dev but 404s in prod" — usually means the prerendered-dir resolver is missing.
- When debugging "ENOENT on dist/client/.worktrees/..." or 403 responses on paths containing a dot-segment.

## The problem

Astro's `[slug].astro` route with `export const prerender = true` emits files like:

```
dist/client/blog/<slug>/index.html
```

The expected URL is `https://<domain>/blog/<slug>/` (trailing slash). Browsers requesting that URL expect to receive the contents of `index.html`.

`express.static` has an `index` option (default: `["index.html"]`) that's SUPPOSED to handle this. In practice it has two issues that bite Astro projects:

1. **Inconsistent trailing-slash handling**: depending on Express version + Node version + OS, `express.static` sometimes serves `/blog/<slug>/` correctly and sometimes returns 404. The behavior depends on whether the request includes a trailing slash, whether the static middleware sees the path as a directory request, and how the filesystem reports the path.

2. **No support for paths containing dot-segments**: Express's `serve-static` (the underlying library) defaults to `dotfiles: "ignore"`, which returns 404 for any path component starting with `.`. On Windows, `git worktree` creates working directories under `<repo>/.worktrees/<branch>/`. The dev server runs from that worktree, and the `dist/client/` path during dev is `<repo>/.worktrees/<branch>/dist/client/`. Express's static middleware refuses to serve anything from a `.worktrees/`-containing path.

## The fix: explicit prerendered-dir middleware

Add this middleware AFTER the service worker headers block and BEFORE `express.static`:

```javascript
import { existsSync } from "fs";
import { join } from "path";

const DIST_CLIENT_DIR = join(__dirname, "dist", "client");

app.use((req, res, next) => {
  if (req.method !== "GET" && req.method !== "HEAD") return next();
  if (!req.path.endsWith("/") || req.path === "/") return next();
  const filePath = join(DIST_CLIENT_DIR, req.path, "index.html");
  if (existsSync(filePath)) {
    res.set("Cache-Control", "public, max-age=3600");
    return res.sendFile(filePath, { dotfiles: "allow" }, (err) => {
      if (err) next(err);
    });
  }
  next();
});
```

**What this does**:
- Intercepts GET/HEAD requests for paths ending in `/` (other than the root `/`).
- Constructs the expected file path: `${DIST_CLIENT_DIR}${req.path}index.html`.
- Checks if the file exists synchronously (fast — node's `existsSync` is a stat call).
- If yes, sends it with `dotfiles: "allow"` so dot-segments don't 403.
- If no, falls through to `express.static` (which may or may not handle it; usually doesn't for the trailing-slash case).

**Why `existsSync` is safe here**: we're checking files we control (build output), not user-supplied paths. The performance overhead is one stat per request to a trailing-slash URL, which is negligible at the request volumes a marketing site sees.

**Why this comes BEFORE `express.static`**: if `express.static` mounts first and returns its own 404, the request never reaches our middleware. Order matters.

## Why `dotfiles: "allow"` is the right choice

`dotfiles: "allow"` permits paths containing dot-segments (`.worktrees/`, `.cache/`, etc.) to be served. The default `"ignore"` would 404 them; the alternative `"deny"` would 403.

The safety concern with `"allow"` is that it could serve files like `.env` or `.git/HEAD` if those were in the static directory. In practice:
- `dist/client/` is the build output. It should not contain `.env` files (build steps that copy `.env` into build output are a security bug; the env vars should be baked into the bundle at build time via `import.meta.env`).
- `dist/client/` should not contain `.git/`. If it does, the `.gitignore` is missing `dist/`.
- The path construction `join(DIST_CLIENT_DIR, req.path, "index.html")` only serves files explicitly named `index.html` — not arbitrary dotfiles within those paths.

The combination of "we only serve files we constructed paths for" + "the file is always `index.html` literally, not a user-supplied filename" means `dotfiles: "allow"` does NOT introduce a vulnerability in this specific middleware.

For the downstream `express.static` middleware (which serves arbitrary files from `DIST_CLIENT_DIR`), I keep the default `dotfiles: "ignore"` because that DOES serve user-pathable URLs. If the user's request path is `/.git/HEAD`, `express.static` with `ignore` will 404 it (correct); with `allow` it would serve the file (very bad).

```javascript
// CORRECT: explicit prerendered-dir middleware allows dotfiles; static middleware doesn't
app.use(prerenderedDirMiddleware);  // dotfiles: "allow" (safe because we control the path)
app.use(express.static(DIST_CLIENT_DIR, {
  index: false,        // we handle index.html ourselves above
  maxAge: "1h",
  extensions: ["html"],
  // dotfiles: "ignore" is the default — DO NOT change to "allow" here
}));
```

## Windows-specific notes

- **Worktree paths**: `git worktree add ../<repo>-articles aio/articles` on Windows creates `<parent-of-repo>/<repo>-articles/.git/` (a worktree marker file, not a full repo). Dev servers running from that worktree have `<absolute-path>/.git/` somewhere in their static-asset resolution. `dotfiles: "ignore"` blocks any request whose resolved path contains `.git/` — which can be most requests on a worktree.
- **The `.worktrees/` convention**: some users run `git worktree add <repo>/.worktrees/<branch>` (worktree as a sibling of `.git/` inside the main repo). This produces paths like `<repo>/.worktrees/<branch>/dist/client/...` which trigger the same blocking behavior.
- **Forward slashes vs backslashes**: Node's `path.join` returns OS-native separators. On Windows that's backslashes. `req.path` is always forward-slashes (URL semantics). The `join(DIST_CLIENT_DIR, req.path, "index.html")` call handles the conversion correctly because `join` does normalization. Don't try to manually replace slashes — `join` handles it.
- **Case sensitivity**: Windows filesystems are typically case-insensitive (NTFS) but case-preserving. `req.path = "/Blog/My-Slug/"` will match a file at `dist/client/blog/my-slug/index.html` on Windows but NOT on Linux. The deployed prod environment is Linux (Railway containers), so test casing carefully.

## Symptoms that point at this issue

- "Works in `npm run dev`, 404 in `npm run start`"
- "Works on macOS, 404 on Windows"
- "Works at `/blog/my-post`, 404 at `/blog/my-post/`" (or vice versa)
- "ENOENT on file path containing `.worktrees`"
- "403 Forbidden on `/blog/<slug>/`"
- "Astro page renders blank or shows server error"
- "Blog index works, individual posts 404"

All point at the prerendered-dir middleware being missing or the `dotfiles` option being set wrong.

## When you don't need this middleware

- **Pure Vite path** (`aio.content_hub = false`): no prerendered routes, no need for this middleware. Vite's dev server + `express.static` for the SPA bundle is sufficient.
- **Astro `output: "static"` (full static build, no SSR)**: same — every route is `<path>/index.html` and the static middleware handles them all because there's no SSR fallback eating the requests first.
- **Astro `output: "server"` deployed serverless** (Vercel, Netlify, Cloudflare Pages): the serverless handler handles routing; Express isn't in the picture.

This middleware is specifically for: Astro `output: "server"` with `@astrojs/node` adapter in `mode: "middleware"`, wrapped by Express, deployed to Railway.

## Cross-references

- [Stage 3](../stage-3-express-server.md) — the canonical middleware order
- [Stage 13](../stage-13-content-hub.md) — sets up the Astro routes that produce prerendered dirs
- [`_internal/claude-invariants.md`](claude-invariants.md) — Invariant #7 (middleware order locked)
