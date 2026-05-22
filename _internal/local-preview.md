# Local Preview Workflows

**Last verified: 2026-05-08.**

## Goal

Iterate on the site locally before pushing to Railway. Production deploys cost build time + Railway resources; local iteration is instant and free. This document covers:

- Running the dev server with hot module reload
- Running the production server locally (with prod-like env vars) for end-to-end testing
- Testing analytics events locally (PostHog, Clarity)
- Testing the consent banner across geo locales
- Testing webhook handlers without exposing local ports
- Using Claude Preview MCP (if available) for in-Claude-Code browser preview
- Multi-device testing patterns

## Workflow 1: `npm run dev` — Vite hot module reload

The fastest iteration loop. Vite's dev server picks up file changes within milliseconds.

```bash
npm run dev
# Server starts at http://localhost:5173 (default Vite port)
```

**Best for**:
- UI iteration (CSS, copy, layout)
- React component logic changes
- Most analytics wiring (PostHog events fire correctly via the loaded callback even in dev)

**Limitations**:
- The Express server (`server.js`) doesn't run in this mode — only Vite's dev server does
- Reverse-proxy at `/<your-proxy-slug>/*` doesn't work locally in this mode (Vite serves the SPA, not the proxy)
- Form endpoints (`/api/newsletter`, `/api/contact`) return 404
- 301 redirects from `server.js` don't fire

So `npm run dev` is great for the "is the React UI working" loop but not the full production-parity loop. For that, see Workflow 2.

## Workflow 2: `npm run start` — production-mode local server

Run the actual `server.js` against the built output. This is what Railway runs in prod.

```bash
npm run build      # produce out/ directory
npm run start      # node server.js — listens on PORT (default 3000)
```

Now `http://localhost:3000` is structurally identical to production:
- Reverse proxy at `/<your-proxy-slug>/*` works
- Form endpoints work (if `RESEND_API_KEY` is set in `.env.local`)
- 301 redirects work
- Static-asset 404s behave correctly

**Best for**:
- End-to-end testing before push
- Verifying the reverse proxy + analytics flow
- Testing form submissions without spamming Railway logs
- Catching `server.js` bugs (missing routes, mounted-too-late middleware)

**Limitations**:
- No hot reload — must rebuild on every change (`npm run build && npm run start`)
- Slower iteration than `npm run dev`

Workflow: use `npm run dev` for 80% of iteration; switch to `npm run start` for the final pre-push verification.

## Workflow 3: `.env.local` for prod-parity

Local-mode Vite reads from `.env.local` (and `.env`, `.env.development`, `.env.production` based on mode). Add a `.env.local` at repo root with your real (non-prod) keys:

```bash
# .env.local (gitignored — never commit)
VITE_POSTHOG_KEY=phc_your_real_key
VITE_POSTHOG_HOST=https://us.i.posthog.com    # use direct ingest locally, not the proxy
VITE_CLARITY_PROJECT_ID=                       # leave empty to disable Clarity locally
RESEND_API_KEY=re_your_real_key                # NOT VITE_-prefixed; server-side only
EMAIL_FROM=Brand <hello@yourdomain.com>
EMAIL_NOTIFY_TO=your-personal-email@gmail.com
SITE_URL=http://localhost:3000
APP_URL=http://localhost:3000
PORT=3000
```

**Why direct PostHog ingest locally instead of the proxy?**

The same-origin proxy lives at `/<your-proxy-slug>/*` — that path is mounted by Express in `server.js`. In `npm run dev` mode, Express isn't running, so events to that path 404. Two solutions:

- **Solution A (recommended)**: set `VITE_POSTHOG_HOST=https://us.i.posthog.com` in `.env.local`. Local events go direct (some get blocked by your local ad-blocker if you have one, but you'll see most).
- **Solution B**: use `npm run start` for analytics testing (proxy works because Express runs)

Solution A is faster for normal dev. Use Solution B when specifically debugging the proxy itself.

**`.gitignore` must include**:

```
.env
.env.local
.env.*.local
```

## Workflow 4: Testing analytics events locally

PostHog events fire from local dev sessions identically to production — they appear in PostHog Live Events with the same shape. To distinguish dev from prod:

```ts
// In main.tsx posthog.init or in a setup helper:
posthog.register({ env: import.meta.env.DEV ? "development" : "production" });
```

Now every event has `env: "development"` locally vs `env: "production"` on Railway. Filter PostHog by `env != "development"` to exclude your dev noise from dashboards.

For Clarity:
- Set `VITE_CLARITY_PROJECT_ID=` (empty) in `.env.local` to disable Clarity locally — keeps your test sessions out of the recording archive
- Or set it and accept that your dev recordings appear; filter by `deploy_variant=variant-devlocal` to ignore them

## Workflow 5: Testing the cookie banner across locales

The Standard tier banner only shows in GDPR locales detected via `cdn-cgi/trace`. The skill's standard `geo.ts` (Stage 8) ships a **permanent dev-only override** so you can test the GDPR path from localhost. No VPN, no manual code editing per test.

The override is part of the standard `detectCountry()` implementation — see Stage 8 for the full code. Behavior:

| URL | What `detectCountry()` returns (in dev only) |
|---|---|
| `http://localhost:5173/` | normal `cdn-cgi/trace` lookup (returns whatever your real IP geolocates to) |
| `http://localhost:5173/?force_gdpr` | `"FR"` — banner appears (Tier 2 Standard) or all-tiers behavior unchanged |
| `http://localhost:5173/?force_country=US` | `"US"` — banner does NOT appear in Tier 2 Standard |
| `http://localhost:5173/?force_country=DE` | `"DE"` — banner appears |
| `http://localhost:5173/?force_country=CH` | `"CH"` — Switzerland (FADP, currently NOT in our GDPR set) — banner does NOT appear (intentional; see Stage 8) |

The `import.meta.env.DEV` guard means Vite drops the override block as dead code during production minification. Safe to commit.

**First-time verification on this project** (do once per new project to confirm Vite drops the override correctly):

1. `npm run build`
2. `grep -r "force_gdpr" out/` — expect ZERO matches in the minified bundle
3. If matches found, your Vite version may handle dead-code-elimination differently — switch to a stricter tree-shake config or move the override to a `.development.ts` import path.

**Test Decline + Accept buttons**:

```js
// Open DevTools console:
localStorage.removeItem("<your-storage-prefix>_analytics_consent");  // I substitute with the project's actual storage-key prefix from Stage 8
location.reload();
// Banner appears. Click Decline. Verify:
posthog.has_opted_out_capturing()  // should be true
```

## Workflow 6: Testing webhook handlers without exposing local ports

Webhook handlers (Stage 9 — Whop / Stripe / etc.) need to receive POST requests from external services. Two approaches:

### Approach A: ngrok / Cloudflare Tunnel — expose localhost publicly

```bash
# ngrok (paid for stable URLs, free for ephemeral)
ngrok http 3000
# → https://abc123.ngrok.io → localhost:3000

# Cloudflare Tunnel (free, requires CF account)
cloudflared tunnel --url http://localhost:3000
```

Then in Whop dashboard → Webhooks → temporarily point at the ngrok URL. Send a test event. Watch your local terminal logs.

**Pro tip**: Whop dashboards usually let you "Send test event" without needing a tunnel — the tunnel is needed only if you want REAL events flowing to your local handler.

### Approach B: Replay captured webhooks locally

Capture a webhook payload in Railway logs (or paste from the vendor's dashboard's "View payload" button), save to `tmp/webhook-test.json`, then:

```bash
curl -X POST http://localhost:3000/api/webhooks/whop \
  -H "Content-Type: application/json" \
  -H "webhook-id: test-id" \
  -H "webhook-signature: v1,fake_sig" \
  -H "webhook-timestamp: 1700000000" \
  --data @tmp/webhook-test.json

# Will fail signature check — adjust your handler temporarily to skip
# verification in dev mode if needed for iteration:
if (process.env.NODE_ENV === "development" && process.env.SKIP_WEBHOOK_VERIFY === "true") {
  // skip HMAC check
}
```

**Never** commit code that disables verification unconditionally; gate on env vars and remove before push.

## Workflow 7: Automated verification via Preview MCP and Chrome MCP

Many of the "open DevTools and check X" verification steps across the skill can be automated via two MCPs. Where these MCPs are available, the skill prefers them — Claude verifies autonomously without requiring the user to open a browser, switch contexts, copy/paste from DevTools, or manually run Lighthouse.

⚠️ **These MCPs are NOT automatically available in every environment.** Activation steps below for each. If your environment doesn't have them and you don't want to set them up, skip to the "Manual fallback" section at the end — every automated verification has a documented manual equivalent.

### Activation — Preview MCP (`mcp__Claude_Preview__*`)

**Status check**: ask your Claude Code instance to list its available tools. If `mcp__Claude_Preview__preview_start` appears, you have it. If not:

1. **Update Claude Code to the latest version**:
   ```bash
   npm install -g @anthropic-ai/claude-code@latest
   # OR your platform's equivalent (Homebrew, winget, etc.)
   ```
   Preview MCP ships with recent versions (2026 onwards). Older versions don't have it.

2. **Restart Claude Code**. The new MCP becomes available.

3. **Verify**: in chat, ask Claude to take a screenshot of localhost. If the tool is available, Claude calls `mcp__Claude_Preview__preview_screenshot` and returns the image.

4. **First-time permission grants**: Claude Code prompts for approval the first time each Preview tool is invoked. Approve. Or pre-approve all at once via the settings.json template at the bottom of this workflow.

### Activation — Chrome MCP (`mcp__Claude_in_Chrome__*`)

Chrome MCP is more involved than Preview MCP — it drives your actual Chrome browser, so it requires a browser extension.

1. **Install the Claude for Chrome browser extension** from the Chrome Web Store. Search for "Claude for Chrome" or follow the link in your Claude Code's MCP setup docs (path: Claude Code Settings → MCPs → Claude in Chrome → Install).

2. **Pair the extension** with this Claude Code instance:
   - Open the extension's popup (click the icon in Chrome's toolbar)
   - Click "Connect to Claude Code"
   - The extension shows a one-time pairing code
   - In Claude Code, run: `claude mcp connect chrome <pairing-code>` (or use the GUI's MCP-connect flow if your client has one)
   - Approve the connection in the browser

3. **Verify**: ask Claude "what tabs do I have open in Chrome?" — should call `mcp__Claude_in_Chrome__tabs_context_mcp` and return your tab list.

4. **Per-tool permissions**: same as Preview — first-use prompts, or pre-approve via settings.json.

5. **Common gotchas**:
   - Extension paired with the WRONG Claude Code instance (multiple machines): re-pair from this machine's extension popup
   - Chrome blocks the connection: check `chrome://extensions` → click "Details" on Claude for Chrome → ensure "Site access" is set to "On all sites"
   - The MCP responds but says "no tab selected": Chrome MCP creates its own tab group; call `tabs_context_mcp` first with `createIfEmpty: true` to bootstrap

### Preview MCP (`mcp__Claude_Preview__*`) — drives a headless preview of your localhost

After activation, per-tool permission cached after first grant.

| Tool | What Claude can autonomously do |
|---|---|
| `preview_start` | Start your dev server (`npm run dev` or `npm run start`) |
| `preview_stop` | Stop it |
| `preview_screenshot` | Capture page state — verify CSS changes, see what the user would see |
| `preview_console_logs` | Read browser console (filter by pattern, look for errors) — catches "this loads but throws JS errors" without manual inspection |
| `preview_network` | Network tab equivalent — verify requests + responses. Critical for confirming PostHog events flow through the reverse proxy (filter for `/<your-proxy-slug>/e/` or whatever your slug is) |
| `preview_inspect` | Inspect computed CSS values on any element |
| `preview_snapshot` | Accessibility tree dump — better than HTML for verifying semantic structure |
| `preview_eval` | Run arbitrary JS in page context. Use to check `posthog.has_opted_out_capturing()`, read localStorage, etc. |
| `preview_click` | Click any element — test Accept/Decline banner buttons, CTA flows |
| `preview_fill` | Fill form inputs — test newsletter / contact form end-to-end |
| `preview_resize` | Change viewport size — responsive checks at mobile / tablet / desktop |
| `preview_logs` | Read the dev server's stdout/stderr — catches server-side errors |

### Chrome MCP (`mcp__Claude_in_Chrome__*`) — drives the user's actual Chrome browser

**Setup**: requires the Chrome extension paired with the MCP. Install once via the Chrome Web Store ("Claude for Chrome").

Useful when verifying against PRODUCTION (not just localhost) — Claude sees what the user sees in their real browser session.

| Tool | What Claude can autonomously do |
|---|---|
| `read_network_requests` | Network tab on the live production site |
| `read_console_messages` | Console output |
| `javascript_tool` | Execute JS in the page context |
| `read_page` | Accessibility tree |
| `find` | Natural-language element search (e.g., "the newsletter signup button") |
| `navigate` | Drive page navigation |
| Browser tabs management | Switch tabs, open new ones |

### What Claude can autonomously verify (mapped to skill stages)

For each verification step the skill currently expresses as "open DevTools and check X", here's the autonomous equivalent:

**Stage 6 — PostHog event flow**:
- ✅ Verify `$pageview` events fire on initial load → `preview_network` filtering on `/<your-slug>/e/`
- ✅ Verify events DON'T hit `*.i.posthog.com` → `preview_network` with negative filter
- ✅ Verify deploy_variant + page_variant super-properties on every event → `preview_eval('posthog._send_request_queue || posthog.get_session_id()')`, then inspect the network request body
- ✅ Verify `recorder.js` loaded from local proxy → `preview_network` filter on `/<your-slug>/static/recorder.js`

**Stage 7 — Clarity + masking**:
- ✅ Verify Clarity snippet loaded → `preview_network` filter on `clarity.ms/tag/<id>`
- ✅ Verify Clarity NOT loaded on excluded routes → navigate via `preview_click`, check no clarity.ms requests appear
- ✅ Verify masking works → take a `preview_screenshot` of a form, then `preview_eval` to check CSS class is applied

**Stage 8 — Consent banner across locales**:
- ✅ Verify banner shows in GDPR → `preview_click` at `/?force_gdpr`, check banner element exists via `preview_inspect`
- ✅ Verify banner DOESN'T show in non-GDPR → `preview_click` at `/?force_country=US`, check element absence
- ✅ Verify Decline opts out → `preview_click` Decline button, then `preview_eval('posthog.has_opted_out_capturing()')` should return `true`
- ✅ Verify Accept opts in → same flow, expect `false`

**Stage 12 — pre-launch checks**:
- ✅ Lighthouse-style metrics → `preview_eval('JSON.stringify(performance.getEntriesByType("navigation")[0])')` for navigation timing; `preview_eval('JSON.stringify(performance.getEntriesByType("paint"))')` for FCP/LCP. Not a full Lighthouse score but covers the load-bearing metrics.
- ✅ Asset 404 check → `preview_network` filter status >= 400
- ✅ HTTPS redirect verification → `preview_network` look for 301 from http to https on the production URL (Chrome MCP path)
- ✅ Smoke test (full happy path: home → pricing → newsletter signup → form submit) → orchestrate via `preview_click` + `preview_fill` + `preview_screenshot` between steps for visual record

**Stage 11 — variant tag flow**:
- ✅ Verify `__PAGE_VARIANTS__` baked into bundle → `preview_eval('typeof __PAGE_VARIANTS__')` should return `"object"` not `"undefined"`
- ✅ Verify deploy_variant on event → `preview_eval('posthog.get_property("deploy_variant")')`

### Lighthouse via Preview MCP

Full Lighthouse requires Chrome's DevTools Protocol. Preview MCP doesn't expose CDP directly, but the load-bearing Lighthouse metrics (FCP, LCP, CLS, TTFB) are reachable via `performance.getEntriesByType()`:

```js
// preview_eval payload:
JSON.stringify({
  navigation: performance.getEntriesByType("navigation")[0],
  paint: performance.getEntriesByType("paint"),
  largestContentfulPaint: (() => {
    const entries = performance.getEntriesByType("largest-contentful-paint");
    return entries.length > 0 ? entries[entries.length - 1] : null;
  })(),
  cls: (() => {
    let cls = 0;
    new PerformanceObserver(list => {
      for (const e of list.getEntries()) if (!e.hadRecentInput) cls += e.value;
    }).observe({ type: "layout-shift", buffered: true });
    return cls;
  })()
})
```

For full Lighthouse scoring (accessibility, best-practices, SEO categories), fall back to running `lighthouse-cli` from a shell:

```bash
npx lighthouse http://localhost:3000 --output=json --output-path=./lighthouse-report.json
```

Claude can then `Read` the JSON file and summarize.

### Permissions / settings

Both MCPs are part of Claude Code's default tool surface (visible in your deferred-tool list). Per-tool permissions:

- First time Claude calls a tool, your client may prompt "Allow `mcp__Claude_Preview__preview_start`?" — approve once, cached per-session or per-conversation depending on your client settings
- Granting access to one Preview MCP tool doesn't auto-grant others — each gets its own first-use permission
- For Chrome MCP specifically: the extension must be installed in Chrome AND paired with your MCP server (see Chrome MCP docs)

**Settings to consider relaxing for autonomy**:

If you want Claude to verify autonomously WITHOUT per-tool approve prompts during a session, add a `permissions.allow` block to `~/.claude/settings.json` (user-level, applies to every project) OR `<project>/.claude/settings.json` (project-only). Pick one of two preset levels:

**Level 1 — Conservative (read-only autonomy)**: pre-approve only the tools that READ state. Anything that mutates state (clicks, form fills, JS execution) keeps prompting per-use. Safest baseline; Claude can verify without asking, but can't accidentally change something.

```json
{
  "permissions": {
    "allow": [
      "mcp__Claude_Preview__preview_start",
      "mcp__Claude_Preview__preview_screenshot",
      "mcp__Claude_Preview__preview_console_logs",
      "mcp__Claude_Preview__preview_network",
      "mcp__Claude_Preview__preview_inspect",
      "mcp__Claude_Preview__preview_snapshot",
      "mcp__Claude_Preview__preview_logs",
      "mcp__Claude_in_Chrome__read_network_requests",
      "mcp__Claude_in_Chrome__read_console_messages",
      "mcp__Claude_in_Chrome__find",
      "mcp__Claude_in_Chrome__read_page"
    ]
  }
}
```

**Level 2 — Full autonomy (read + write)**: pre-approve everything including UI-interaction tools. Claude can run an end-to-end smoke test (navigate to home, click a CTA, fill the newsletter form, submit, screenshot the success state) without asking. Faster but trades some safety — Claude could click "Decline" on a banner you wanted to test "Accept" on, or fill a form with placeholder data while you weren't watching.

```json
{
  "permissions": {
    "allow": [
      "mcp__Claude_Preview__preview_start",
      "mcp__Claude_Preview__preview_screenshot",
      "mcp__Claude_Preview__preview_console_logs",
      "mcp__Claude_Preview__preview_network",
      "mcp__Claude_Preview__preview_eval",
      "mcp__Claude_Preview__preview_click",
      "mcp__Claude_Preview__preview_fill",
      "mcp__Claude_Preview__preview_inspect",
      "mcp__Claude_Preview__preview_snapshot",
      "mcp__Claude_Preview__preview_logs",
      "mcp__Claude_in_Chrome__read_network_requests",
      "mcp__Claude_in_Chrome__read_console_messages",
      "mcp__Claude_in_Chrome__find",
      "mcp__Claude_in_Chrome__read_page"
    ]
  }
}
```

**Recommendation**: start with Level 1, upgrade to Level 2 once the user has worked with Claude long enough to trust the verification flow.

To remove a tool from the allowlist, edit settings.json and remove the line; to revert entirely to prompt-per-use, delete the `permissions` block.

### Manual fallback (when MCPs aren't available or you prefer hands-on)

Every automated step above has a manual equivalent — open your browser, hit `http://localhost:3000`, open DevTools, and check the same thing the MCP would have. The skill's verification checklists in each stage list both:

- "✅ Verify $pageview events fire — automated: `preview_network` filtering on `/<your-slug>/e/`, OR manual: DevTools Network tab, filter `posthog`"
- "✅ Verify masking works — automated: `preview_snapshot` + grep for `clarity-mask` class, OR manual: DevTools → Elements tab → inspect a form input"

So skipping MCP setup is a fully supported path. Tradeoff: you'll do more manual context-switching during verification phases, especially Stage 12's smoke test.

## Workflow 8: Multi-device + responsive testing

A surprising amount of bugs only manifest on real mobile devices. DevTools' device emulator catches most but not all (touch behavior, real iOS Safari quirks, font rendering on retina, etc.).

For local testing on a phone:

1. Find your computer's local IP: `ipconfig getifaddr en0` (Mac) or `ipconfig` (Windows)
2. Make sure phone + computer on same Wi-Fi
3. Update `vite.config.ts` server config:

```ts
server: {
  host: "0.0.0.0",   // ← required for LAN access (already set if you followed Stage 1)
  port: 5173,
}
```

4. On phone: open `http://<your-computer-ip>:5173` (e.g., `http://192.168.1.42:5173`)

For Express prod mode (`npm run start`), same — `0.0.0.0:3000` on phone reaches the local server.

**Common gotchas**:
- macOS firewall blocks incoming connections by default. Allow in System Preferences → Security → Firewall → Options
- Windows Defender Firewall: allow Node.exe inbound on private network
- Some routers isolate clients ("AP isolation"); disable if you see "connection refused" from phone but not laptop

## Workflow 9: Browser DevTools verification

Throughout the skill, "verify in DevTools" appears as a step. The key checks:

### Network tab — verify analytics flow

1. Open DevTools → Network tab
2. Filter by `posthog` (case-insensitive)
3. Reload page

Expected requests (Standard tier, accepted consent):
- `POST /<your-proxy-path>/e/?... → 200` (initial $pageview)
- `GET /<your-proxy-path>/array/<token>/config.js → 200` (PostHog config bootstrap)
- `GET /<your-proxy-path>/static/recorder.js → 200` (session replay loader)
- `GET /<your-proxy-path>/decide/?... → 200` (feature flag evaluation)

If you see requests to `*.i.posthog.com` instead, your `VITE_POSTHOG_HOST` didn't bake correctly.

### Console — verify no errors + custom events fire

1. DevTools → Console
2. Run `posthog` to see the loaded SDK is exposed (we assigned it in `loaded` callback)
3. Run `posthog.get_distinct_id()` to see your distinct_id
4. Run `posthog.capture("test_event", { from: "console" })` — appears in PostHog Live Events within seconds

### Application tab — verify cookies + localStorage

1. DevTools → Application → Cookies / Local Storage
2. Confirm:
   - PostHog distinct_id cookie present (key starts with `ph_`)
   - Consent decision in localStorage (`<your-prefix>_analytics_consent`)
   - Affiliate code in cookie + localStorage (if testing affiliate flow)

## Workflow 10: Lighthouse + accessibility audits

Run before launch (Stage 12) and ad-hoc throughout development:

```
DevTools → Lighthouse → Generate report
- Mode: Navigation (default)
- Device: Mobile + Desktop (run separately)
- Categories: All
```

Targets:
- Performance: ≥85 mobile, ≥95 desktop
- Accessibility: ≥95
- Best Practices: ≥95
- SEO: ≥95

For deeper accessibility: install [axe DevTools extension](https://chrome.google.com/webstore/detail/axe-devtools-web-accessib) — catches more issues than Lighthouse alone.

## Common gotchas

- **`.env.local` not picked up**: Vite reads it on dev server start, NOT on file change. Restart `npm run dev` after editing.
- **PostHog events fire locally but show wrong distinct_id in production**: localStorage carries over if you use the same browser. Clear `localStorage` after a `posthog.reset()` for clean session.
- **Reverse proxy 404 in dev**: see Workflow 3 — use direct ingest in `.env.local` (`VITE_POSTHOG_HOST=https://us.i.posthog.com`) for `npm run dev`. The proxy works in `npm run start`.
- **Cookie banner stuck open**: localStorage flag persists. Clear via DevTools → Application → Local Storage → delete the consent key, reload.
- **Form submissions fail locally with "RESEND_API_KEY not set"**: add to `.env.local`. If using a real key in dev, watch the Resend free-tier quota (100/day).
- **Webhook handler tests fail signature check**: real webhooks need real signing. Either use a tunnel (ngrok/cloudflared) so the vendor's webhook hits your local handler with valid signatures, or temporarily disable verification (env-gated, never committed).

## Cross-references

- [stage-1-scaffolding.md](../stage-1-scaffolding.md) — initial `npm run dev` setup
- [stage-3-express-server.md](../stage-3-express-server.md) — `npm run start` workflow
- [stage-6-posthog.md](../stage-6-posthog.md) — PostHog event verification in DevTools
- [stage-7-clarity.md](../stage-7-clarity.md) — Clarity recording verification
- [stage-8-privacy-consent.md](../stage-8-privacy-consent.md) — banner testing across locales
- [stage-12-launch-checks.md](../stage-12-launch-checks.md) — Lighthouse + smoke test
- [reference-cloudflare-dns.md](./reference-cloudflare-dns.md) — DNS-related local testing patterns (sibling file in `_internal/`)

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Documents `npm run dev` vs `npm run start`, `.env.local` for prod-parity, analytics + banner testing locally, webhook tunneling via ngrok/cloudflared, Claude Preview MCP integration, multi-device LAN access, DevTools verification patterns, Lighthouse audit setup. |
| 2026-05-08 | Workflow 5 reframed: the `?force_gdpr` / `?force_country=XX` URL override is now a permanent dev-only feature of `geo.ts` (added to Stage 8's standard implementation), not a "temporarily edit then remove" pattern. Added URL → return-value table for the override. Added first-time per-project verification step to confirm Vite drops the dev block from production. |
| 2026-05-08 | Workflow 7 expanded: full automated-verification surface via Preview MCP + Chrome MCP. Tool inventory tables for both. Stage-by-stage mapping of "open DevTools and check X" verification steps to autonomous Claude tools. Lighthouse-equivalent metrics via `preview_eval` and `performance.getEntriesByType()`. Permissions / settings.json template for relaxed-autonomy mode (pre-approving read-only browser tools while keeping write tools prompt-per-use). |
