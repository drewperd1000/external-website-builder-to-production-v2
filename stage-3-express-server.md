# Stage 3: Express Server (wraps Astro handler or serves Vite dist)

**Last verified: 2026-05-21.**

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) write `server.js`, install Express, update `package.json`, and verify the local server works correctly via my Bash tool + Preview MCP. The user does NOT type `npm install` / write code / run `curl` checks — I do all of that. The user's only action in this stage is approving the design choices I surface (tombstone SW yes/no, 301 redirect block yes/no, Basic Auth credentials for staging).

> **Upgrading from v1?** See [CHANGELOG.md](CHANGELOG.md). This stage adds Basic Auth middleware, the prerendered-directory resolver for Astro routes, and the Astro handler mount; v1 served only Vite dist files.

## The locked middleware order

`server.js` has 10 ordered middleware positions. Each one has a load-bearing reason to be where it is — reordering breaks something concrete. See Invariant #7 in [`_internal/claude-invariants.md`](_internal/claude-invariants.md) for the failure mode per misordering.

```
1. PostHog reverse proxy        ← BEFORE express.json() (raw body needed)
2. Basic Auth gate              ← no-op unless env vars set
3. express.json({ limit: ... }) ← can parse JSON safely now
4. 301 redirects (PWA paths)    ← if origin.domain_migration_scenario applies
5. Form endpoints               ← /api/<form-name> handlers from Stage 5
6. Admin endpoints              ← /api/_admin/* (Reoon usage, etc.)
7. Service worker headers       ← /sw.js, /registerSW.js cache-control
8. Prerendered-dir resolver     ← Astro path only — <path>/index.html
9. express.static(DIST_CLIENT)  ← static assets
10. Astro handler / Vite static ← FALLBACK (everything else)
```

## Implementation: Basic Auth middleware

```javascript
// server.js — Basic Auth middleware (immediately after PostHog proxy)
app.use((req, res, next) => {
  const user = process.env.BASIC_AUTH_USER;
  const pass = process.env.BASIC_AUTH_PASS;
  if (!user || !pass) return next();  // no-op in production where vars are unset

  const auth = req.headers.authorization;
  if (!auth || !auth.startsWith("Basic ")) {
    res.set("WWW-Authenticate", 'Basic realm="<your-brand> Staging"');
    return res.status(401).type("text/plain").send("Authentication required");
  }

  const decoded = Buffer.from(auth.slice(6), "base64").toString("utf-8");
  const idx = decoded.indexOf(":");
  const providedUser = idx >= 0 ? decoded.slice(0, idx) : decoded;
  const providedPass = idx >= 0 ? decoded.slice(idx + 1) : "";

  if (providedUser !== user || providedPass !== pass) {
    res.set("WWW-Authenticate", 'Basic realm="<your-brand> Staging"');
    return res.status(401).type("text/plain").send("Invalid credentials");
  }
  next();
});
```

**Why Express-layer and not Astro middleware**: an Astro middleware can gate dynamic SSR routes, but it cannot gate `express.static` (the static asset served before Astro's handler even runs). Express-layer auth gates the WHOLE site including static, which is the desired behavior for staging.

**Why this comes BEFORE `express.json()`**: even unauthenticated POSTs with JSON bodies shouldn't get parsed. The 401 response should fire before any body reading.

**Why this comes AFTER the PostHog proxy**: PostHog ingest from staging needs to work without auth — otherwise the analytics layer is broken on staging and you can't test it. The PostHog proxy paths are intentionally NOT auth-gated; everything else is.

## Implementation: prerendered-directory resolver (Astro path only)

```javascript
// server.js — after service worker headers, before express.static
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

**Why `dotfiles: "allow"`**: dev paths on Windows worktrees often contain `.worktrees/<branch>/dist/client/blog/<slug>/index.html`. Without `dotfiles: "allow"`, Express's default `dotfiles: "ignore"` returns 404 (the `.worktrees/` segment makes the path look hidden). The `allow` option is safe because we're explicitly constructing the path from `DIST_CLIENT_DIR + req.path` — no user-controllable input directs file access outside the build output.

This is documented in [`_internal/reference-prerendered-windows.md`](_internal/reference-prerendered-windows.md) along with the related `express.static` dotfiles handling.

## Implementation: Astro handler mount (Astro path only)

```javascript
// server.js — at the very end, after express.static
import { handler as astroHandler } from "./dist/server/entry.mjs";

app.use(astroHandler);
```

The Astro handler serves any route the explicit Express middleware didn't handle: SSR pages, dynamic Astro routes, etc. The import path `./dist/server/entry.mjs` is correct for `@astrojs/node` in middleware mode — that's where Astro emits the request handler.

**Build dependency**: `npm run build` must run before `node server.js` starts. Stage 4's Railway config sets `build` as the Railway Build Command and `node server.js` as the Start Command, so this is automatic in production.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 3 needs:
- `hosting.deploy_target` — Express layout differs slightly for Railway (default) vs Vercel/Netlify (serverless functions). Default Railway if unset.
- `project.app_url` and `project.site_url` — referenced by 301-redirect block if applicable. If both unset and onboarding flagged a domain migration, I prompt; otherwise I omit the redirect block and note in a comment for Stage 4.
- `origin.domain_migration_scenario` — drives whether I include the tombstone Service Worker. Default `fresh_launch` if unset (omits the tombstone).

## What I'm doing in this stage

Adding a small Express server that handles four concerns BEFORE static file serving:

1. **PostHog reverse proxy mounting** (raw body, must come before `express.json()`) — Stage 6 populates this
2. **Webhook handlers** (raw body for HMAC verification) — Stage 9 populates if applicable
3. **Form submission endpoints** (`/api/newsletter`, `/api/contact`) — Stage 5 populates this
4. **301 redirects** for paths moving to a subdomain (only if Stage 0 identified domain migration)

Everything else falls through to `express.static` over the Vite build output, with `index.html` SPA fallback.

After this stage, `npm run start` serves the production-built bundle on `0.0.0.0:$PORT` with the structure ready for Stages 5/6/9 to populate. **Time**: 5–10 minutes of automated work.

## Why I add an Express server (not just `serve` or `vite preview`)

A static-only host doesn't work for this stack because:

- **PostHog reverse proxy needs an HTTP layer** — the proxy intercepts a same-origin path and forwards to PostHog's ingest. Static-only hosting can't do this.
- **Form endpoints need an HTTP layer** — `/api/newsletter` needs to land at the user's backend, not the source platform's.
- **301 redirects must happen before HTML delivery** — if a user bookmarks `https://mayasconsulting.com/login` after the app moves to `app.mayasconsulting.com`, the redirect should fire at the HTTP layer, not after the marketing site's React loads.
- **Webhook handlers need an HTTP layer** — Stripe, Whop, and other payment platforms POST signed bodies. Static-only hosting can't receive.

A 200-line Express server gives all of this with one runtime dep (`express`). Marketing sites with no forms/payments could skip this, but the cost is so small that I default to including it.

## Prerequisites

- Stage 2 complete (migration-punchlist resolved, no platform deps remaining).
- I know from Stage 0 / onboarding whether the user is migrating off an existing PWA- or otherwise-served domain or sub-domain (which requires the tombstone SW + 301 redirects). If onboarding marked `domain_migration_scenario: "fresh_launch"`, I skip the migration-only blocks.
- I know the actual Vite `outDir` from Stage 1 (`dist/` is Vite's default; some templates override to `out/`). I read `vite.config.ts` to confirm.

## My execution sequence

### Step 1: I install Express

```
npm install express
```

If TypeScript is being used on the server too, I also add `@types/express` as a dev dependency. For most marketing sites I default to plain JS for the server (lower complexity); the user can switch to TS later if they want.

### Step 2: I write `server.js` at repo root

Below is the template I write, with all the load-bearing structure documented inline. **The user doesn't need to read this code** — it's shown for developers/experts who want to inspect what I've created.

```js
/**
 * Maya's Consulting — server
 * ============================================================================
 * (I substitute the actual brand name when I write this file.)
 *
 * Tiny Express app handling four concerns before falling through to
 * static file serving:
 *
 *  1. PostHog reverse proxy — same-origin path forwards to PostHog ingest,
 *     dodging ad-blocker fingerprinting + Cloudflare 1000 errors. Mounted
 *     BEFORE express.json() because PostHog sends compressed binary.
 *  2. Webhook handlers — Whop / Stripe / etc. signed events. Same raw-body
 *     constraint as PostHog proxy.
 *  3. 301 redirects — paths moving to a subdomain. Optional; this block
 *     is omitted entirely if not migrating off an existing domain.
 *  4. Form endpoints — /api/newsletter, /api/contact, etc. Backed by
 *     an email provider in Stage 5.
 *
 * Everything else falls through to express.static + SPA fallback.
 *
 * Port binding: 0.0.0.0:$PORT — required by Railway container networking.
 */

import express from "express";
import { fileURLToPath } from "url";
import { dirname, join } from "path";
// Stage 5 imports (added when Stage 5 runs):
// import { Resend } from "resend";
// Stage 6 imports (added when Stage 6 runs):
// import { registerPosthogProxy } from "./posthog-proxy.js";
// Stage 9 imports (added when Stage 9 runs, if applicable):
// import { registerStripeWebhook } from "./stripe-webhook.js";  // default
// — or — import { registerWhopWebhook } from "./whop-webhook.js";  // if Whop

const __dirname = dirname(fileURLToPath(import.meta.url));

// Vite default outDir is "dist". Some templates override to "out".
// I detect the actual value from vite.config.ts before I write this file.
const OUT_DIR = join(__dirname, "dist");

const PORT = process.env.PORT || 3000;
const APP_URL = process.env.APP_URL || "https://app.<your-domain>";
const SITE_URL = process.env.SITE_URL || "https://<your-domain>";

// 301-redirect targets — paths that USED to live on this domain but
// now belong to a subdomain. I omit this entire block if not migrating.
const PWA_PATH_PREFIXES = [
  "/login",
  "/signup",
  "/forgot-password",
  "/reset-password",
  "/verify-email",
  "/account",
  "/admin",
  // I populate this list from the user's actual app routes during write.
];

const app = express();
app.set("trust proxy", 1);  // Railway sits behind a load balancer

// ─── 0. PostHog reverse proxy ───────────────────────────────────────────
// Stage 6 will populate this. MUST be mounted BEFORE express.json() so
// the proxy can read raw POST bodies (PostHog sends compressed binary,
// not JSON). See posthog-proxy.js for the full design notes.
//
// registerPosthogProxy(app, {
//   staticDir: join(OUT_DIR, "posthog-static"),
// });

// ─── 0b. Payment-platform webhook handler ───────────────────────────────
// Stage 9 populates if applicable. Same raw-body constraint as the
// PostHog proxy — Stripe, Whop, and most others use HMAC signatures
// computed against the exact request bytes, so this MUST be mounted
// before express.json() consumes the body.
//
// registerStripeWebhook(app);   // default — Stripe is most common
// registerWhopWebhook(app);     // alternative — content/community/course platforms

// ─── JSON body parser (after raw-body consumers above) ──────────────────
app.use(express.json({ limit: "32kb" }));

// ─── 1. 301 redirects for paths that moved subdomains ───────────────────
// Conditional block. Omit if not migrating off an existing domain.
app.use((req, res, next) => {
  const path = req.path;
  const isPwaPath = PWA_PATH_PREFIXES.some(
    (p) => path === p || path.startsWith(p + "/"),
  );
  if (!isPwaPath) return next();
  const qIdx = req.originalUrl.indexOf("?");
  const queryString = qIdx >= 0 ? req.originalUrl.slice(qIdx) : "";
  res.redirect(301, APP_URL + path + queryString);
});

// ─── 2. Form submission endpoints ───────────────────────────────────────
// Stage 5 populates these with handlers generated from the forms manifest
// produced by Stage 0. ONE endpoint per form; the path is derived from
// the form's `name` in the manifest (e.g., name: "newsletter_signup" →
// path: "/api/newsletter-signup"). The endpoints below are illustrative;
// the actual list comes from `forms-manifest.md`. Common examples:
//   app.post("/api/newsletter-signup", async (req, res) => { ... });
//   app.post("/api/contact",           async (req, res) => { ... });
//   app.post("/api/request-demo",      async (req, res) => { ... });
//   app.post("/api/event-rsvp",        async (req, res) => { ... });
//   app.post("/api/get-a-quote",       async (req, res) => { ... });
// Each handler: validate per the form's discovered schema → persist to
// form_submissions table (default; per Stage 5 / reference-forms-and-persistence)
// → notify-email (optional) → ESP segment sync (optional) → CRM sync
// (optional) → fire <form_name>_completed PostHog event.

// ─── 3. Service worker files — short cache + explicit headers ───────────
// Only relevant if migrating off a previously-served domain (PWA or otherwise). The tombstone SW
// at public/sw.js needs short-cache headers so existing users pick up the
// self-unregistering version on next visit, not days later.
app.get(["/sw.js", "/registerSW.js"], (req, res, next) => {
  res.set("Cache-Control", "public, max-age=0, must-revalidate");
  res.set("Service-Worker-Allowed", "/");
  next();
});

// ─── 4. Static files from the Vite build output ─────────────────────────
app.use(
  express.static(OUT_DIR, {
    index: false,                    // SPA fallback handles index.html
    maxAge: "1h",                    // 1h cache for hashed assets
    extensions: ["html"],            // /about → /about.html if it exists
  }),
);

// ─── 5. SPA fallback ────────────────────────────────────────────────────
// Asset-extension paths that don't exist (e.g., a missing /assets/foo.js)
// should 404 with a plain-text body, NOT serve index.html (which would
// have wrong Content-Type for a .js request and trigger console errors).
app.use((req, res) => {
  if (/\.[a-z0-9]+$/i.test(req.path)) {
    return res.status(404).type("text/plain").send("Not found");
  }
  res.sendFile(join(OUT_DIR, "index.html"));
});

app.listen(PORT, "0.0.0.0", () => {
  console.log(`[server] listening on 0.0.0.0:${PORT}`);
  console.log(`[server] APP_URL: ${APP_URL}`);
  console.log(`[server] SITE_URL: ${SITE_URL}`);
});
```

When I write the file, I substitute:
- The brand name in the header comment
- `OUT_DIR` to match the actual Vite outDir from `vite.config.ts`
- The `PWA_PATH_PREFIXES` list with the user's actual app routes (or remove the entire block if not migrating)
- `APP_URL` / `SITE_URL` defaults to the user's actual domain

### Step 3: I update `package.json`

I add the `"type": "module"` field (required for ESM `import` syntax) and the `start` script:

```json
{
  "type": "module",
  "scripts": {
    "build": "vite build",
    "start": "node server.js",
    "dev": "vite",
    "type-check": "tsc --noEmit"
  }
}
```

If the project is already CommonJS, I instead convert the server file's `import` statements to `require(...)` so I don't disturb the existing module config.

### Step 4 (conditional): I write the tombstone service worker

I do this ONLY if the user is migrating off an existing PWA- or otherwise-served domain or sub-domain (per onboarding's `domain_migration_scenario` field). The previous host could have been any service worker scope — a PWA at the apex, a marketing site that happened to register a worker for offline support, a different app at a subdomain we're now reclaiming, etc.

**Why it's needed**: existing PWA users have a service worker registered at the old domain's root scope. When their browser checks for SW updates, it pulls the new file. If the new file doesn't actively unregister itself + redirect to the new app domain, those users get stuck on stale cached PWA content for as long as they have the old SW. The tombstone SW fixes this by clearing all caches, redirecting open clients, and unregistering itself.

I write `public/sw.js`:

```js
/**
 * Tombstone service worker.
 *
 * Existing PWA users have an SW registered at this domain's root scope.
 * On activate: clear all caches, navigate any open clients to the new
 * app domain, then unregister itself.
 */
self.addEventListener("install", (event) => {
  event.waitUntil(self.skipWaiting());
});

self.addEventListener("activate", async (event) => {
  event.waitUntil((async () => {
    // Clear all caches
    const keys = await caches.keys();
    await Promise.all(keys.map((k) => caches.delete(k)));
    // Navigate any open clients to the new app domain.
    // I substitute the actual APP_URL value at write time.
    const clients = await self.clients.matchAll({ type: "window" });
    const APP_URL = "https://app.<your-domain>";
    clients.forEach((client) => {
      try {
        const u = new URL(client.url);
        client.navigate(APP_URL + u.pathname + u.search);
      } catch {
        client.navigate(APP_URL);
      }
    });
    // Unregister this SW
    await self.registration.unregister();
  })());
});
```

I also write `public/registerSW.js` (in case the old PWA registered the SW from this path):

```js
// Tombstone — old PWA registered an SW at /registerSW.js. Serve a no-op
// here so requests succeed without re-registering anything.
console.log("[sw] tombstone — no SW registered.");
```

The Express server's short-cache headers from Step 2 ensure existing users pick up the tombstone within minutes of visiting, not days.

### Step 5: I run local verification autonomously

I run all of these via my Bash tool + Preview MCP:

| Check | What I do | Pass criteria |
|---|---|---|
| Build succeeds | `npm run build` | Output directory populated, no errors |
| Server starts | `npm run start` (background) then I curl `localhost:$PORT/` | HTTP 200, expected log lines printed |
| 404 on missing assets | Curl `localhost:$PORT/assets/nonexistent.js` | HTTP 404 with `Content-Type: text/plain` |
| SPA fallback works | Curl `localhost:$PORT/some/random/path` | HTTP 200 with `Content-Type: text/html` and `index.html` body |
| 301 redirects (if applicable) | Curl `localhost:$PORT/login` | HTTP 301 with `Location: <APP_URL>/login` |
| Tombstone SW (if applicable) | Curl `localhost:$PORT/sw.js` | HTTP 200 with `Cache-Control: public, max-age=0, must-revalidate` |

### Step 6: I commit and report

After verification passes, I run:

```
git add server.js public/sw.js public/registerSW.js package.json
git commit -m "feat: Express server with reverse-proxy + redirect scaffolding (Stage 3)"
```

I don't push yet — Stage 4 will configure Railway to deploy from this commit.

I summarize for the user:

```
✅ Stage 3 complete:
   • server.js created at repo root (137 lines)
   • Outputs from dist/ (Vite's default; verified by reading vite.config.ts)
   • Listens on 0.0.0.0:3000
   • Tombstone SW: skipped (not migrating off any previously-served domain)
   • 301 redirects: skipped (fresh launch, no PWA paths to redirect)
   • All 4 verification curls pass
   • Committed locally; ready for Stage 4 (Railway deploy)

Ready to deploy?
```

## Common gotchas I handle automatically

- **`type: "module"` mismatches**: ESM imports + CommonJS deps occasionally collide. If `import express from "express"` errors with `ERR_REQUIRE_ESM`, I downgrade to `const express = await import("express")` or convert the whole file to CJS.
- **Port binding to localhost only**: `app.listen(PORT)` (without `"0.0.0.0"`) listens on the loopback interface only. Railway containers can't reach it externally; the deploy fails health checks. I always pass `"0.0.0.0"` explicitly.
- **`PORT` defaults to 3000 locally** but Railway provides its own `PORT` env var. The server reads `process.env.PORT || 3000`, which works in both contexts. I never override Railway's `PORT`.
- **Trailing slashes**: `express.static`'s `extensions: ["html"]` resolves `/about` to `/about.html` if it exists, but does NOT redirect `/about/` to `/about`. If the user wants consistent slashes, I add explicit redirect middleware (and ask first).
- **OUT_DIR mismatch**: Vite's default is `dist/`; some templates override to `out/`. I read `vite.config.ts`'s `build.outDir` value before writing `server.js` and substitute the right one.

## Outputs

After Stage 3:
- `server.js` at repo root with reverse-proxy / webhook / form / redirect / static / SPA-fallback structure ready for Stages 5/6/9 to populate
- `public/sw.js` + `public/registerSW.js` (only if tombstone SW needed)
- `package.json` updated with `"type": "module"` + `"start": "node server.js"`
- Local server running, all verification checks passing
- Status report shown to user

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. |
| 2026-05-08 | Reframed for non-coder audience: replaced "Use this template — adapt to your project's specifics" with "I write this file; the user can review the code if they want." Fixed `OUT_DIR` default from `out` to `dist` (Vite's actual default; some templates override to `out`, so I detect actual value from vite.config.ts). Replaced reference to author's PWA TS port with platform-agnostic note. Replaced literal example domains in template with `<your-domain>` placeholders. Verification section reframed from user-runs-curl to "I run these checks autonomously; here's the summary I report." |
| 2026-05-08 | Tombstone SW conditional broadened: previously gated on "PWA-served domain"; now triggers on any previously-served domain or sub-domain that may have registered a service worker (PWA at the apex, marketing site with offline support, an app at a subdomain we're reclaiming, etc.). The previous narrow framing missed real-world migration scenarios where the predecessor wasn't a PWA but still had an SW registered. |
