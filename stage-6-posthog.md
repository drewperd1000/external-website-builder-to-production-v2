# Stage 6: PostHog Setup with Reverse Proxy

**Last verified: 2026-05-08.** Re-research before this stage if older than 60 days. Search: `PostHog MCP server <year>`, `PostHog model context protocol`. Check `posthog.com/docs/model-context-protocol` for current capability list.

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) wire PostHog into the project — install the SDK, write the reverse proxy, the static-vendoring script, the analytics module structure, the `main.tsx` init with `loaded` callback, build-time deploy identifiers, env-var setting on Railway, and the verification end-to-end. The user's actions: (1) sign up at PostHog and create a project (one-time, ~5 min in dashboard), (2) install the PostHog MCP via `npx @posthog/wizard@latest mcp add` and click "Approve" in browser once. Everything else autonomous.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 6 needs:
- `analytics.posthog.region` — default `"us"` if unset; user confirms (only override if EU residency required).
- `analytics.proxy_path_slug` — I propose `/community-reading-room` and ask the user to accept or override (per Step 5 in this stage). I save the result back to config.
- `analytics.posthog.project_id` — populated by Stage 6 itself once the user creates the project; if missing, that's expected at the start of this stage.
- `analytics.stack` — `"both"` (default), `"posthog_only"`, or `"clarity_only"`. Drives whether Stage 7 runs.
- The user's PostHog Project API Key (the `phc_...` public key), pasted into `<project>/.secrets/posthog-public-key.txt`. **Parallel work I can do while waiting**: write the reverse-proxy module, the static-vendoring script, and the analytics module scaffolding. The `railway variable set VITE_POSTHOG_KEY=...` is the only step that blocks on the paste.
- `DEPLOY_TIMEZONE` env var on Railway — defaults to `UTC` from Stage 4. If the user wants per-stage override, I prompt; otherwise UTC stands.

## What I'm doing in this stage

Wiring PostHog product analytics into the site with full instrumentation:

1. **Client init with `loaded` callback** — fires the initial `$pageview` after init resolves, eliminating the race that drops bounce visitors' single event
2. **Same-origin reverse proxy** — events flow through a marketing-style path on the user's own domain instead of `*.i.posthog.com`, recovering 10–25% of events lost to ad-blocker fingerprinting
3. **Vendored `posthog-js/dist/*.js`** — served locally to dodge Cloudflare Error 1000 on Railway → PostHog egress
4. **Deploy identifiers** as super-properties — `deploy_id`, `deploy_variant`, `deploy_variant_sha` attached to every event for clean cohort analysis
5. **Custom UTM capture** — non-standard `utm_*` params (e.g., `utm_placement` for ad sub-placements) registered as super-properties because posthog-js auto-captures only the standard 5
6. **PostHog MCP server** — chat-driven CRUD over flags / experiments / dashboards / insights / HogQL

**Time**: 15–30 minutes of automated work, plus ~5 min user action for the PostHog dashboard signup.

## Prerequisites

- Stages 1–4 complete (Railway service deployed, server.js with reverse-proxy hookpoint, foundational env vars set).
- A PostHog account. PostHog **doesn't expose project creation via MCP** — the user must create the project in the dashboard first, then I instrument.

## A note on free-tier quotas (so the user knows what to expect)

PostHog's free plan as of 2026-05:
- 1,000,000 events/month
- **5,000 desktop session replays/month**
- **2,500 mobile session replays/month**
- 1,000,000 feature-flag requests/month
- Unlimited dashboards, insights, projects

The events quota is essentially unreachable for a marketing site (a 50k-pageview/month site emits roughly 200–300k events with custom events + autocapture). The replay quotas are reachable for a site doing a real launch — once you cross 5k desktop replays/month, the rest of the month's sessions stop recording in PostHog.

**This is the context for installing Clarity in Stage 7**: Clarity has no recording cap on its free tier, so it keeps recording past the PostHog quota. Two-replay-vendor setup means full visibility regardless of traffic spikes. (If the user opted for `posthog_only` in onboarding, I'll honor that choice but flag this tradeoff once at the end of this stage so they can change their mind cheaply.)

## Automation surface I have available (verified 2026-05)

### PostHog MCP — `https://mcp.posthog.com/sse`

Installer: `npx @posthog/wizard@latest mcp add`. Auth via OAuth (browser approve). Tools available:

- Feature flag CRUD: `create-feature-flag`, `update-feature-flag`, `delete-feature-flag`, `feature-flag-get-definition`, `feature-flag-get-all`
- Experiment CRUD: `experiment-create`, `experiment-update`, `experiment-delete`, `experiment-get`, `experiment-launch`, `experiment-pause`, `experiment-end`, `experiment-results-get`, `experiment-ship-variant`
- Dashboard CRUD: `dashboard-create`, `dashboard-update`, `dashboard-delete`, `dashboard-get`, `dashboards-get-all`
- Insight CRUD: `insight-create`, `insight-update`, `insight-delete`, `insight-get`, `insights-list`
- HogQL: `query-run`, `query-validate`, `execute-sql`, `query-generate-hogql-from-question`
- Session recording: `session-recording-get`, `session-recording-delete`, etc.
- Project management: `switch-project`, `project-get`, `project-settings-update`

**Tools NOT in MCP**: project create, organization billing, user/permissions management. Those require dashboard.

### PostHog REST API + Personal API Keys

Personal API Keys are **dashboard-only** to create. Generate at `app.posthog.com/settings/user-api-keys?preset=mcp_server`. **My default**: I use the MCP for all interactive work; I only fall back to the REST API + Personal API Key if the user explicitly opts out of the MCP.

## My execution sequence

### Step 1: User creates the PostHog project (one-time dashboard work)

I tell the user (with click-through path):

```
First — quick PostHog dashboard work, about 3 minutes:

  1. Please open https://posthog.com/signup in your browser (I'll open it in a tab via Chrome MCP if it's connected; otherwise visit
     directly if it's open already).
  2. Sign up; pick the US region (us.posthog.com) unless your audience
     is exclusively EU — then EU works too.
  3. Create a new project. Name it after this site (e.g.,
     "Maya's Consulting Marketing").
  4. After project creation, you'll be on the Settings → Project page.
     Look for "Project API Key" — it starts with "phc_". Copy it.
  5. Paste the key here, and I'll save it to a project-local secrets
     file so the value doesn't end up in chat history.
```

When the user pastes:
1. I write the key to `<project>/.secrets/posthog-public-key.txt`
2. I confirm `.secrets/` is gitignored (set up in Stage 4)
3. I capture the project ID from PostHog's URL or via a quick `mcp__posthog__projects-get` after Step 2

### Step 2: I install the PostHog MCP

I run:

```
npx @posthog/wizard@latest mcp add
```

The wizard:
1. Opens a browser for OAuth approval (user clicks "Approve" once)
2. Detects which Claude Code config to write to
3. Registers `https://mcp.posthog.com/sse` as an MCP server
4. Confirms with a `whoami` call

After install, I verify by calling `mcp__posthog__projects-get` — this returns the user's project list, and I match the project ID for this site.

If the user explicitly declined the MCP during onboarding, I skip this step and use the Personal API Key fallback. In that case I have them generate a Personal API Key at `app.posthog.com/settings/user-api-keys?preset=mcp_server`, save it to `<project>/.secrets/posthog-personal-api-key.txt`, and use it for direct REST calls.

### Step 3: I install posthog-js

```
npm install posthog-js posthog-js/react
```

The React provider lives in the same package as the core SDK in current versions.

### Step 4: I write the static-vendoring script

Cloudflare Error 1000 is the load-bearing reason for this step. When Railway's egress fetches from `us-assets.i.posthog.com`, both endpoints sit behind Cloudflare; CF treats the server-to-server request as a loop and 1000-errors. Vendoring the dist files at build time sidesteps the entire issue.

I create `scripts/copy-posthog-static.mjs`:

```js
/**
 * Vendor posthog-js's runtime-fetched static files into public/posthog-static/.
 *
 * posthog-js loads recorder.js, surveys.js, etc. on demand from
 * `${api_host}/static/<file>`. With our same-origin reverse proxy, those
 * requests hit our Express server. Forwarding them to us-assets.i.posthog.com
 * from Railway's egress triggers Cloudflare Error 1000. Vendoring the dist
 * files at build time and serving them via express.static sidesteps the
 * issue entirely.
 */
import { copyFileSync, mkdirSync, readdirSync, existsSync, rmSync } from "node:fs";
import { join, dirname } from "node:path";
import { createRequire } from "node:module";
import { fileURLToPath } from "node:url";

const require = createRequire(import.meta.url);
const __dirname = dirname(fileURLToPath(import.meta.url));

const phPkgPath = require.resolve("posthog-js/package.json");
const phPkg = require("posthog-js/package.json");
const srcDir = join(dirname(phPkgPath), "dist");
const destDir = join(__dirname, "..", "public", "posthog-static");

if (!existsSync(srcDir)) {
  console.error(`[copy-posthog-static] FATAL: posthog-js dist not found at ${srcDir}`);
  process.exit(1);
}

if (existsSync(destDir)) rmSync(destDir, { recursive: true, force: true });
mkdirSync(destDir, { recursive: true });

let copied = 0;
for (const f of readdirSync(srcDir)) {
  if (!f.endsWith(".js")) continue;
  copyFileSync(join(srcDir, f), join(destDir, f));
  copied++;
}
console.log(`[copy-posthog-static] copied ${copied} .js files from posthog-js@${phPkg.version} → ${destDir}`);
```

I update `package.json` to run this on every dev start and build:

```json
{
  "scripts": {
    "predev": "node scripts/copy-posthog-static.mjs",
    "prebuild": "node scripts/copy-posthog-static.mjs",
    "dev": "vite",
    "build": "vite build",
    "start": "node server.js"
  }
}
```

I add `public/posthog-static/` to `.gitignore` (auto-regenerated; no need to commit).

### Step 5: I write the reverse-proxy module

**Path naming is load-bearing.** The proxy lives at a path on the user's own domain (e.g., `https://<user-domain>/<proxy-slug>/...`), and that slug needs to read like real marketing content so heuristic ad-blockers (uBlock Origin, Brave Shields, Pi-Hole, AdGuard) don't fingerprint it as analytics.

#### My slug-picker rule

I propose **one** slug that:
- Is 2–4 words, hyphen-separated
- Sounds like a real content section the site might have
- Contains generic editorial vocabulary (`about-`, `community-`, `resources-`, `articles-`, `guides-`, `insights-`, `reading-room`, `knowledge-base`)
- Avoids the ad-blocker fingerprint patterns below

**Default proposal**: `/community-reading-room` — neutral, brand-agnostic, multi-segment, doesn't match any known blocklist patterns I'm aware of as of 2026-05.

I show the user one proposal and ask:

```
For the analytics reverse proxy, I'm going to use the path
"/community-reading-room" on your domain. This is invisible to visitors —
it's just where analytics traffic flows under the hood — but it needs to
look like real content so privacy tools don't block it.

Sound good? You can also:
  (a) Accept default → "/community-reading-room"
  (b) Suggest your own — anything 2–4 words, hyphen-separated, sounds
      content-y, avoids "/api" / "/track" / "/collect" / etc.
```

If the user wants to suggest their own, I run a sanity check on the suggestion against the fingerprint blacklist below before accepting it.

#### Fingerprint patterns I avoid (ad-blocker blocklists)

- ❌ Single-word generic: `/api`, `/track`, `/log`, `/collect`, `/event`, `/events`, `/metrics`, `/analytics`, `/pixel`, `/p`, `/t`, `/e`
- ❌ Tracking-related vocabulary: `/tracking`, `/capture`, `/recorder`, `/replay`, `/heatmap`, `/session`
- ❌ Tool-name leaks: `/posthog`, `/ph`, `/clarity`, `/ms-clarity`, `/segment`, `/mixpanel`, `/amplitude`, `/hotjar`, `/plausible`
- ❌ API-version patterns: `/v1`, `/v2`, `/api/v1`, `/i/v0`

After the user picks (default or override), I:
1. Run `grep -r "<chosen-slug>" src/` to confirm zero collision with real routes
2. Save the chosen slug to `<project>/.skill-config.json` as `analytics.proxy_path_slug`
3. Use the slug in the rest of this stage and in subsequent stages (Stage 11's variant tagging references it too)

#### I write `posthog-proxy.js`

```js
/**
 * PostHog reverse-proxy middleware.
 *
 * Forwards traffic from a same-origin path to PostHog's ingestion host so
 * events look like first-party traffic to ad blockers. /static/* served
 * locally (vendored posthog-js/dist) instead of proxied because
 * us-assets.i.posthog.com triggers Cloudflare Error 1000 on Railway egress.
 *
 * Mounted in server.js BEFORE express.json() so we can read raw POST bodies
 * (PostHog sends compressed binary, not JSON).
 */
import express from "express";

const POSTHOG_INGEST = "https://us.i.posthog.com";  // change to https://eu.i.posthog.com if EU project

export const PROXY_BASE = process.env.POSTHOG_PROXY_PATH || "/<your-proxy-slug>";

const STRIP_REQUEST_HEADERS = new Set([
  "host", "connection", "content-length", "transfer-encoding",
  "upgrade", "te", "trailer", "proxy-authorization", "proxy-authenticate",
  "x-forwarded-for", "x-forwarded-host", "x-forwarded-proto", "x-real-ip",
  "cookie",  // don't leak our session cookie to PostHog
]);

const STRIP_RESPONSE_HEADERS = new Set([
  "connection", "content-length", "content-encoding", "transfer-encoding",
  "upgrade", "te", "trailer", "keep-alive",
]);

function readRawBody(req) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    req.on("data", (chunk) => chunks.push(chunk));
    req.on("end", () => resolve(Buffer.concat(chunks)));
    req.on("error", reject);
  });
}

async function handleProxy(req, res) {
  const subPath = req.path;
  if (subPath.startsWith("/static/")) {
    res.status(404).type("text/plain").send("PostHog static asset not found");
    return;
  }
  const qIndex = req.originalUrl.indexOf("?");
  const queryString = qIndex >= 0 ? req.originalUrl.slice(qIndex) : "";
  const upstreamUrl = `${POSTHOG_INGEST}${subPath}${queryString}`;

  const fwdHeaders = {};
  for (const [name, value] of Object.entries(req.headers)) {
    if (STRIP_REQUEST_HEADERS.has(name.toLowerCase())) continue;
    if (Array.isArray(value)) fwdHeaders[name] = value.join(", ");
    else if (typeof value === "string") fwdHeaders[name] = value;
  }

  let body;
  if (req.method !== "GET" && req.method !== "HEAD") {
    body = await readRawBody(req);
  }

  let upstreamRes;
  try {
    upstreamRes = await fetch(upstreamUrl, { method: req.method, headers: fwdHeaders, body });
  } catch (err) {
    console.warn(`[posthogProxy] upstream fetch failed for ${req.method} ${subPath}:`, err);
    res.status(502).json({ error: "Upstream unavailable" });
    return;
  }

  res.status(upstreamRes.status);
  upstreamRes.headers.forEach((value, name) => {
    if (STRIP_RESPONSE_HEADERS.has(name.toLowerCase())) return;
    res.setHeader(name, value);
  });
  const buf = Buffer.from(await upstreamRes.arrayBuffer());
  res.end(buf);
}

export function registerPosthogProxy(app, options) {
  const staticDir = options?.staticDir;
  if (!staticDir) throw new Error("[posthogProxy] options.staticDir is required");

  app.use(`${PROXY_BASE}/static`, express.static(staticDir, { maxAge: "1h", fallthrough: false }));
  app.use(PROXY_BASE, (req, res) => { void handleProxy(req, res); });

  console.log(`[posthogProxy] mounted at ${PROXY_BASE} → ${POSTHOG_INGEST} (/static from ${staticDir})`);
}
```

I substitute `<your-proxy-slug>` with the user's actual chosen slug at write time.

### Step 6: I mount the proxy in `server.js`

I update Stage 3's `server.js` to import + register:

```js
import { registerPosthogProxy } from "./posthog-proxy.js";

// ... after const app = express()
// ... BEFORE express.json():
registerPosthogProxy(app, { staticDir: join(OUT_DIR, "posthog-static") });

app.use(express.json({ limit: "32kb" }));
```

The order is critical (Stage 3's invariant): proxy mounted BEFORE `express.json()` so it can read raw POST bodies.

### Step 7: I add build-time deploy identifiers to `vite.config.ts`

I update `vite.config.ts` to inject `deploy_id` + `deploy_variant_sha` (and read `deploy_variant` from a JSON file when Stage 11 lands; for now just the SHA-based fallback):

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

function deriveDeployId(): string {
  const parts = Object.fromEntries(
    new Intl.DateTimeFormat("sv-SE", {
      timeZone: process.env.DEPLOY_TIMEZONE || "UTC",  // Configurable via DEPLOY_TIMEZONE env var (any IANA timezone like "America/New_York" or "Europe/London"); defaults to UTC if unset
      year: "numeric", month: "2-digit", day: "2-digit",
      hour: "2-digit", minute: "2-digit", hour12: false,
    }).formatToParts(new Date()).map((p) => [p.type, p.value]),
  );
  const rawSha = process.env.RAILWAY_GIT_COMMIT_SHA;
  const sha = rawSha ? rawSha.slice(0, 7) : "devlocal";
  return `${parts.year}${parts.month}${parts.day}-${parts.hour}${parts.minute}-${sha}`;
}

function deriveDeployVariantSha(): string {
  const rawSha = process.env.RAILWAY_GIT_COMMIT_SHA;
  if (rawSha) return `variant-${rawSha.slice(0, 7)}`;
  return "variant-devlocal";
}

export default defineConfig({
  define: {
    "import.meta.env.VITE_DEPLOY_ID": JSON.stringify(deriveDeployId()),
    "import.meta.env.VITE_DEPLOY_VARIANT_SHA": JSON.stringify(deriveDeployVariantSha()),
    // Stage 11 adds: VITE_DEPLOY_VARIANT (from current-variants.json) + __PAGE_VARIANTS__
  },
  plugins: [react()],
  // ... rest of config
});
```

### Step 8: I create the analytics module structure

I create these files (Stage 11 expands them; for now, scaffold):

```
src/_core/analytics/
├── deploy-tag.ts        # super-property registration
├── utm-capture.ts       # custom utm_* capture
├── affiliate-capture.ts # ?ref= / ?a= capture (Stage 10 expands)
├── track.ts             # typed event wrapper
└── index.ts             # barrel + initAnalytics()
```

The contents (all in TypeScript):

**`deploy-tag.ts`** — registers deploy identifiers as PostHog super-properties:

```ts
import posthog from "posthog-js";

export function registerDeployTagOnPostHog(): void {
  if (!posthog.__loaded) return;
  const props: Record<string, string> = {};
  const id = import.meta.env.VITE_DEPLOY_ID as string | undefined;
  const variant = import.meta.env.VITE_DEPLOY_VARIANT as string | undefined;
  const sha = import.meta.env.VITE_DEPLOY_VARIANT_SHA as string | undefined;
  if (id) props.deploy_id = id;
  if (variant) props.deploy_variant = variant;
  if (sha) props.deploy_variant_sha = sha;
  if (Object.keys(props).length > 0) posthog.register(props);
}
```

**`utm-capture.ts`** — captures custom UTMs that posthog-js doesn't auto-capture:

```ts
import posthog from "posthog-js";

const CUSTOM_UTM_KEYS = ["utm_placement"] as const;  // extend as needed

export function registerCustomUtmsOnPostHog(): void {
  if (typeof window === "undefined" || !posthog.__loaded) return;
  const params = new URLSearchParams(window.location.search);
  const props: Record<string, string> = {};
  for (const key of CUSTOM_UTM_KEYS) {
    const value = params.get(key);
    if (value !== null && value !== "") props[key] = value;
  }
  if (Object.keys(props).length > 0) posthog.register(props);
}
```

posthog-js auto-captures only the standard 5 UTMs (`utm_source / medium / campaign / term / content`). Custom UTMs (e.g., `utm_placement` for ad sub-platforms) need explicit registration.

**`affiliate-capture.ts`** — Stage 10 fully implements this. For now, stub:

```ts
import posthog from "posthog-js";

export function registerAffiliateCodeOnPostHog(): string | null {
  // Stage 10 fills this in
  return null;
}
```

**`track.ts`** — typed event wrapper:

```ts
import posthog from "posthog-js";

type EventProps = Record<string, string | number | boolean | null | undefined>;

export function track(event: string, props?: EventProps): void {
  try {
    if (posthog.__loaded) posthog.capture(event, props);
  } catch {
    // analytics never breaks the site
  }
}

export function extractEmailDomain(email: string): string {
  try {
    const at = email.lastIndexOf("@");
    if (at < 0 || at === email.length - 1) return "unknown";
    return email.slice(at + 1).toLowerCase().trim();
  } catch {
    return "unknown";
  }
}
```

**`index.ts`** — barrel + orchestrator:

```ts
export { registerDeployTagOnPostHog } from "./deploy-tag";
export { registerCustomUtmsOnPostHog } from "./utm-capture";
export { registerAffiliateCodeOnPostHog } from "./affiliate-capture";
export { track, extractEmailDomain } from "./track";

// Stage 7 will populate this with Clarity init + flag-bridge
export function initAnalytics(): void {
  // Stage 7 fills in
}
```

### Step 9: I init PostHog in `main.tsx`

This is the **load-bearing** step. SKILL.md invariant #3 lives here: super-properties register BEFORE first capture, all inside the `loaded` callback.

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import posthog from "posthog-js";
import { PostHogProvider } from "posthog-js/react";
import App from "./App";
import {
  initAnalytics,
  registerDeployTagOnPostHog,
  registerCustomUtmsOnPostHog,
  registerAffiliateCodeOnPostHog,
} from "./_core/analytics";

const posthogKey = import.meta.env.VITE_POSTHOG_KEY as string | undefined;
const posthogHost =
  (import.meta.env.VITE_POSTHOG_HOST as string | undefined) ??
  "https://<user-domain>/<your-proxy-slug>";  // I substitute real values

if (posthogKey) {
  posthog.init(posthogKey, {
    api_host: posthogHost,
    ui_host: "https://us.posthog.com",  // ← DASHBOARD origin (stays *.posthog.com even when api_host is proxied)
    capture_pageview: false,            // we fire it manually (see loaded callback)
    capture_pageleave: true,            // accurate bounce-rate + session duration
    disable_session_recording: false,
    session_recording: { maskAllInputs: true },  // mask form values
    persistence: "localStorage+cookie",
    respect_dnt: false,                 // explicit opt: DNT users still tracked unless they opt out via banner
    loaded: (ph) => {
      // ← BEFORE first capture: register super-properties
      registerDeployTagOnPostHog();
      registerCustomUtmsOnPostHog();
      registerAffiliateCodeOnPostHog();

      // ← Initial pageview (the loaded-callback pattern that avoids the
      //   useEffect race documented in SKILL.md invariant #3)
      ph.capture("$pageview", { $current_url: window.location.href });

      // Expose for live debugging
      (window as unknown as { posthog: typeof ph }).posthog = ph;

      // Stage 7 will populate initAnalytics() with Clarity init + flag bridge
      initAnalytics();
    },
  });
}

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <PostHogProvider client={posthog}>
      <App />
    </PostHogProvider>
  </StrictMode>,
);
```

#### Why `capture_pageview: false`

posthog-js's auto-pageview captures fire on init AND on history changes. I want manual control because:
- I want to add custom properties to the pageview (`deploy_variant`, `page_variant`, etc.)
- The auto-pageview races with `register()` — registered super-properties might not attach to the FIRST event
- The auto-pageview-on-history-change is buggy with React Router (depends on version)

So I manually fire `$pageview` once in the `loaded` callback (initial load) and once in the router's `useEffect([pathname])` (subsequent navigation).

### Step 10: I capture pageviews on route change

In the project's router (typically `src/router/index.ts` or inline in `src/App.tsx`), I add:

```tsx
import { useLocation } from "react-router-dom";
import { useEffect } from "react";
import posthog from "posthog-js";

export function AppRoutes() {
  const { pathname } = useLocation();
  useEffect(() => {
    window.scrollTo(0, 0);
    // Fire pageview unconditionally. posthog-js internally queues captures
    // if init hasn't completed yet and flushes them when ready.
    posthog.capture("$pageview", { $current_url: window.location.href });
  }, [pathname]);
  // ... render routes
}
```

The unconditional capture is intentional — earlier versions guarded with `if (posthog.__loaded)`, which created a race-condition bug on initial load where the guard short-circuited and `[pathname]` never changed to retry. SKILL.md invariant #3.

### Step 11: I set Railway env vars

I run (per Stage 4's secrets-file pattern):

```
railway variable set VITE_POSTHOG_KEY="$(cat .secrets/posthog-public-key.txt)"
railway variable set VITE_POSTHOG_HOST="https://<user-domain>/<your-proxy-slug>"
railway variable set RAILWAY_CHECKOUT_DEPTH=0
```

Then I trigger a redeploy: `railway up`. Vite needs to rebuild with the new env vars in scope (build-time bake — SKILL.md invariant #1).

### Step 12: I verify end-to-end (autonomous)

I run all of these myself via Preview MCP / Bash:

| Check | What I do | Pass criteria |
|---|---|---|
| Build picks up `VITE_POSTHOG_KEY` | Curl deployed `/index.html`, search bundle for the phc_ prefix | Match found in JS bundle |
| Reverse proxy live | `curl -X POST https://<user-domain>/<your-proxy-slug>/e/?...` | HTTP 200 (forwarded to PostHog) |
| No requests to `*.i.posthog.com` | Open deployed site via Preview MCP, check `preview_network` | No `i.posthog.com` requests; all on `<user-domain>/<your-proxy-slug>/...` |
| `recorder.js` served locally | Check `preview_network` for `/static/recorder.js` request | HTTP 200, served from same origin (verified via `Server` header or `x-served-from`) |
| PostHog Live Events fire | `mcp__posthog__query-run` with HogQL: `select event, properties.deploy_id from events order by timestamp desc limit 5` | `$pageview` event with non-null `deploy_id`, `deploy_variant_sha` |
| Session replay records test session | Visit deployed site, check Recordings list via MCP | Test session listed |
| Form inputs masked in recording | Open recording playback (need browser; user confirms) | Newsletter email shows as `***` |

If events aren't flowing, common causes I diagnose:
- DevTools shows requests to `*.i.posthog.com` instead of proxy → `VITE_POSTHOG_HOST` didn't bake. I check Railway → Variables and redeploy.
- DevTools shows `/<proxy-slug>/config.js` 404 → PROXY_BASE in posthog-proxy.js doesn't match what posthog-js is requesting. I check both match exactly.
- DevTools shows `/<proxy-slug>/static/recorder.js` 404 → `copy-posthog-static.mjs` didn't run during build, or Express's static dir is wrong. I check `out/posthog-static/` exists in deployed container.
- DevTools shows `/<proxy-slug>/e/` 502 → upstream fetch is failing. I check Railway logs for `[posthogProxy] upstream fetch failed`.

### Step 13: I verify MCP works (after Step 2 install)

I run a simple query through the MCP:

```
mcp__posthog__dashboards-get-all
```

Expected: returns the project's dashboards (initially empty for a new project; any I create in Stage 11 will appear here).

If MCP errors:
- I re-run `npx @posthog/wizard@latest mcp add` to re-authenticate
- I check `~/.claude.json` (or the user's MCP config) for the SSE entry
- I prompt the user to restart Claude Code if needed

### Step 14: I add starter dashboards to the user's account

While I'm here, I create a few starter insights so the user has something useful to look at on day one:

- **Trend**: `$pageview` count, last 7 days, breakdown by `deploy_variant`
- **Trend**: distinct visitors, last 7 days
- **Live events tile**: top 20 events from the last 24h

Stage 11 builds out the full dashboard set (7 dashboards). Step 14 just confirms data is flowing in usable form.

I summarize for the user:

```
✅ Stage 6 complete:
   • PostHog account active; project "Maya's Consulting Marketing" created
   • PostHog MCP installed + authenticated
   • Reverse proxy live at https://mayasconsulting.com/<your-proxy-slug>/
   • posthog-js dist files vendored (Cloudflare Error 1000 mitigation)
   • Loaded-callback pattern in main.tsx (race-free initial pageview)
   • Deploy identifiers attached to every event:
     - deploy_id: 20260508-1430-<sha7>
     - deploy_variant_sha: variant-<sha7>
     - (deploy_variant added in Stage 11)
   • Custom UTM (utm_placement) registered as super-property
   • Session replay active with maskAllInputs: true
   • 3 starter insights created: Pageviews, Visitors, Live Events
   • End-to-end test: $pageview events flowing through proxy, visible in
     PostHog Live Events with non-null deploy_id

Ready for Stage 7 (Clarity)?
```

## Common gotchas I handle automatically

- **Forgot the `loaded` callback**: silent first-pageview drop. SKILL.md invariant #3 — I never use `useEffect` for the initial pageview.
- **Set env vars AFTER first build**: bundle has `undefined` for `VITE_POSTHOG_KEY`. I trigger redeploy after every env-var change. SKILL.md invariant #1.
- **`api_host` set to dashboard URL** (`https://us.posthog.com`) instead of ingest URL: events 404. The proxy path or `https://us.i.posthog.com` for direct ingest is what works.
- **Same-origin proxy on a different domain than the site**: defeats the purpose. The proxy path must be served from the SAME origin as the site.
- **Forgot `RAILWAY_CHECKOUT_DEPTH=0`**: Stage 11's `git rev-list --count HEAD` returns 1 silently on shallow clone, breaking variant labels. I set it in Stage 4's foundational env vars precisely to prevent this.
- **`maskAllInputs` masks too aggressively**: legit fields like dropdown selects come through as `***`. If the user needs a select unmasked, I add `data-ph-no-mask="false"` to the specific element.

## Self-research instruction

Before running this stage, I web-search:
- `PostHog MCP server <year>` — capability list grows; check what's new
- `posthog-js loaded callback <year>` — confirm pattern still recommended
- `PostHog reverse proxy <year>` — check for any official guidance changes

## Outputs

After Stage 6:
- PostHog client wired with same-origin proxy + loaded-callback pattern
- `posthog-proxy.js` mounted in `server.js`
- `scripts/copy-posthog-static.mjs` vendoring on every build
- `src/_core/analytics/` scaffolded (deploy-tag, utm-capture, affiliate-capture stub, track, index)
- Live events flowing with `deploy_id` + `deploy_variant_sha` super-properties
- PostHog MCP authenticated for chat-driven dashboard / flag / experiment work
- Session replay active with masked inputs
- 3 starter insights visible in PostHog dashboard

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Verified PostHog MCP at `mcp.posthog.com/sse` supports OAuth. Tool surface covers flag/experiment/dashboard/insight/HogQL CRUD; project-create not exposed. Personal API Keys remain dashboard-only. |
| 2026-05-08 | Step 5 expanded with reverse-proxy slug naming guidance: brand-category-specific examples, fingerprint-pattern blacklist, slug-collision check, blocklist-validation sanity check. |
| 2026-05-08 | Reframed for non-coder audience: every "Run X" / "Create file Y" reframed to first-person Claude voice. Replaced literal author-domain proxy path with `<your-proxy-slug>` substitution marker; replaced literal author-key-prefix examples with `phc_xxxxxxxxxxxx` placeholder. Replaced literal author-timezone with "I substitute the user's timezone". Verification section reframed from user-runs-curl to "I run these checks autonomously via Preview MCP + MCP queries; here's the summary I report." |
| 2026-05-08 | Simplified Step 5 slug picker: removed "propose 2–3 candidates" + per-brand-category table. Replaced with single neutral default proposal (`/community-reading-room`) + accept-or-suggest path. The fingerprint-blacklist sanity check still runs against any user override. Decision rationale: brand-category table padded the user's mental load for ~zero benefit — "neutral content-y string" picks are interchangeable, and the user choosing one over another is decision fatigue rather than meaningful customization. |
