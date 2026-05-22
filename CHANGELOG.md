# v2 Changelog — What's New vs v1

**v1 lives at [`../external-website-builder-to-production/`](../external-website-builder-to-production/) and remains a valid skill.** v2 is a sibling — install both and pick the right one per project. The two coexist as separate installed skills.

## When to pick v2

If the marketing site should be **discoverable by AI search engines** (ChatGPT Search, Perplexity, Google AI Overview, Claude with web search), v2 is the right choice. v2 ships the content-hub infrastructure (schema markup, content collections, sitemap+RSS, draft-preview, cornerstone workflow, Google Search Console automation) that makes AI assistants cite the brand.

For most marketing sites in 2026 and beyond, that's the default. Pure-Vite sites without schema infrastructure are leaving citation surface on the table.

## When v1 is still right

- Tightly-scoped landing pages with paid-only traffic (no organic ambitions)
- Internal admin tools that should never be indexed
- Sites where the user explicitly wants to skip the content-hub work

## The 10 deltas v2 owns

### 1. Astro 5 as default stack

v1 scaffolds Vite-only. v2 forks at onboarding: Astro 5 + React islands + Express wrapper when `aio.content_hub = true` (default), Vite when false. Astro brings native MDX, content collections, sitemap+RSS generation, schema components, and the draft-preview pattern.

**Files affected:** `onboarding.md`, `stage-1-scaffolding.md` (fork), `stage-3-express-server.md` (wraps Astro handler when forked).

### 2. AIO content-hub infrastructure (NEW stage 13)

Eight schema components (`Organization`, `WebSite`, `Person`, `Article`, `FAQPage`, `HowTo`, `Product`, `BreadcrumbList`) with TypeScript builders and Vitest validation tests. `BaseLayout.astro` with `head-extra` slot for per-page schema + OG + Twitter card injection. Content collection schema with `tags`, `readingTime`, `draft`, `bucket` (A/B/C/D query-bucket alignment), `schemas` enum array. Draft-aware sitemap filter — drafts get prerendered URLs for editorial preview but stay out of `sitemap-index.xml`, RSS, and the `/blog` index. RSS feed scaffold via `@astrojs/rss`.

**Files:** [`stage-13-content-hub.md`](stage-13-content-hub.md), [`_internal/reference-aio-schema-library.md`](_internal/reference-aio-schema-library.md).

### 3. Reoon Email Verifier integration (replaces TODO in v1's stage-5)

v1's `stage-5-email-resend.md` carries an HTML comment block flagging "🚧 PENDING — Reoon Email Verifier". v2 PORTS the actual `email-verifier.js` code into the skill:

- Quick mode (blocking gate; rejects `invalid` / `disposable` / `spamtrap`)
- Power mode (fire-and-forget async; flags `safe` / `disabled` / `inbox_full` / `catch_all` / `role_account` to Slack)
- User-facing rejection messages per quick-reject status (spamtrap intentionally generic — don't tell an attacker we detected them)
- In-memory daily-quota counter with UTC midnight reset
- 80% + 100% Slack threshold alerts (once-per-day dedup)
- `/api/_admin/reoon-usage` admin spot-check endpoint
- `REOON_DAILY_QUOTA` env var with AppSumo LTD tier ladder (Tier 1 = 500/day → Tier 5 = 4500/day)
- Fail-open posture: network errors, timeouts, non-200s, 429-credit-exhausted on quick → `{ ok: true }`

**Files:** [`stage-5-email-resend.md`](stage-5-email-resend.md) (full code, not TODO), [`_internal/reference-quota-monitoring.md`](_internal/reference-quota-monitoring.md) (the pattern generalized for any vendor with a per-day or per-month cap).

### 4. Google Search Console sitemap automation (NEW stage 15)

A `gsc_sitemap_submit.py` CLI helper authenticated via the project's `oauth_helper.py` (Google OAuth with `webmasters` + `webmasters.readonly` scopes). Subcommands: `sites` / `list` / `submit` / `remove` / `replace`. Documents the additional-owner pattern for the edge case where the GSC property is owned by a different Google account than the OAuth helper authenticates as.

**Files:** [`stage-15-gsc-sitemap.md`](stage-15-gsc-sitemap.md), and a reference to `.shared/scripts/gsc_sitemap_submit.py` (lives outside any individual project repo; reusable across projects).

### 5. Two-stage deploy + HTTP Basic Auth staging service

v1 deploys a single Railway service. v2 documents the pattern of creating a SECOND service (`<name>-staging`) pointing at the same repo branch with `BASIC_AUTH_USER` + `BASIC_AUTH_PASS` env vars set ONLY on staging. The Express middleware enforces the gate when the vars are set and no-ops when they're not. No Cloudflare DNS/Access needed — the default Railway URL plus Basic Auth is the entire staging surface.

**Files:** [`stage-3-express-server.md`](stage-3-express-server.md) (middleware), [`stage-4-railway.md`](stage-4-railway.md) (staging service setup).

### 6. `client:only="react"` Astro island pattern wrapping legacy React

When importing a React Router site into Astro, per-page island wrapper components provide `<BrowserRouter>` context. Existing `<Link>` calls work because `useLocation()` has its context provider. Astro is the actual router; BrowserRouter is effectively a context shim. The follow-up refactor (replace `<Link>` with `<a>`, then drop the BrowserRouter wrapper) is documented but optional.

**Decision criterion:** wrap if existing site is <2 weeks old or has >10 pages; refactor to pure Astro otherwise.

**Files:** [`stage-13-content-hub.md`](stage-13-content-hub.md).

### 7. Prerendered-directory + Windows-worktree gotcha

Express middleware that explicitly resolves `<path>/index.html` for prerendered blog routes (Astro's Node adapter doesn't reliably handle trailing-slash dirs through `express.static`'s `index` option on Windows). Uses `dotfiles: "allow"` because dev paths can include `.worktrees/` — any path with a dot-segment would otherwise 403.

**Files:** [`_internal/reference-prerendered-windows.md`](_internal/reference-prerendered-windows.md).

### 8. Threshold-alert + admin-endpoint generalized pattern

The Reoon usage-tracking machinery is a reusable building block for any vendor with a daily or monthly cap: counter + UTC reset + 80%/100% Slack alert + admin spot-check endpoint. Reference doc lists candidate applications: Reoon (already wired), Resend monthly quota, PostHog volume cap, Backblaze B2 storage, any per-day or per-month API budget.

**Files:** [`_internal/reference-quota-monitoring.md`](_internal/reference-quota-monitoring.md).

### 9. Railway deploy verification gotcha (commitHash, not id)

`latestDeployment.id` and `createdAt` DO NOT change between deploys in Railway's GraphQL response. The deployment record is re-used in place. The only reliable change-signal is `meta.commitHash` matching the pushed commit. v2 documents the correct polling pattern: `railway status --json` → parse JSON → check `commitHash`, not `id`.

**Files:** [`_internal/reference-railway-automation.md`](_internal/reference-railway-automation.md) (updated section).

### 10. Cornerstone-article content workflow (NEW stage 14)

The article-session-vs-integration-session protocol: cornerstone drafting happens in a separate `aio/articles` branch and worktree to keep MDX work out of the implementation thread. Integration sessions merge in periodically. The draft-to-publish flip protocol (edit frontmatter `draft: true → false`, commit, push — sitemap auto-regenerates, GSC re-pings). The pattern for cornerstones (4-cornerstone seed strategy, query-bucket alignment, 2K–4K word target, inline FAQPage + HowTo schema, internal-link discipline). Does NOT include cornerstone drafts verbatim — those are brand-specific.

**Files:** [`stage-14-cornerstones.md`](stage-14-cornerstones.md).

## 🚧 Learnings-pending callouts

Four areas in v2 are flagged with `🚧 LEARNINGS-PENDING` HTML comment blocks because the cornerstone-article ranking data from the reference implementation isn't in yet:

1. **Cornerstone topic selection criteria** — current heuristic is query-bucket alignment; lacks post-publish ranking confirmation
2. **Optimal schema combinations per content type** — does Article+FAQPage+HowTo consistently outperform Article+FAQPage alone? Unknown until 30+ days of citation data
3. **Internal-link patterns** — density, directionality, cornerstone-to-cornerstone vs. cornerstone-to-product
4. **Cornerstone-to-conversion measurement** — which PostHog events meaningfully tie AI-search-driven traffic to signup/purchase
5. **Refresh cadence** — how often to update each cornerstone (this is what the pending `aio-content-maintenance` skill will own)

Each flag includes: (a) what the open question is, (b) where the data will come from (PostHog dashboard / GSC performance pull / Citation tracker), (c) when to revisit (typically "30 days after cornerstone 1 publishes, 60 days after cornerstone 2 publishes").

## Stage-by-stage map: v1 → v2

| v1 stage | v2 stage | Change type |
|---|---|---|
| SKILL.md | SKILL.md | Rewritten — 16 stages, content-hub fork, expanded automation contract |
| README.md | README.md | Rewritten — references v2 deltas |
| onboarding.md | onboarding.md | Extended — adds content-hub question, named-author identity, cornerstone seed Qs |
| stage-0-discovery.md | stage-0-discovery.md | Extended — detect existing blog/CMS content, identify cornerstone candidates from source |
| stage-1-scaffolding.md | stage-1-scaffolding.md | FORKED — Astro 5 if content-hub, Vite if not |
| stage-2-platform-migration.md | stage-2-platform-migration.md | Unchanged from v1 |
| stage-3-express-server.md | stage-3-express-server.md | Extended — Basic Auth middleware, prerendered-dir resolver, dotfiles handling |
| stage-4-railway.md | stage-4-railway.md | Extended — staging service creation pattern, commitHash deploy-poll fix |
| stage-5-email-resend.md | stage-5-email-resend.md | PORTED — Reoon integration replaces TODO comment with full code |
| stage-6-posthog.md | stage-6-posthog.md | Unchanged from v1 |
| stage-7-clarity.md | stage-7-clarity.md | Unchanged from v1 |
| stage-8-privacy-consent.md | stage-8-privacy-consent.md | Unchanged from v1 |
| stage-9-membership-optional.md | stage-9-membership-optional.md | Unchanged from v1 |
| stage-10-affiliate-optional.md | stage-10-affiliate-optional.md | Unchanged from v1 |
| stage-11-variants-and-dashboards.md | stage-11-variants-and-dashboards.md | Unchanged from v1 |
| stage-12-launch-checks.md | stage-12-launch-checks.md | Extended — GSC sitemap submit verification, schema validator, JSON-LD count audit |
| — | **stage-13-content-hub.md** | NEW — AIO infrastructure |
| — | **stage-14-cornerstones.md** | NEW — cornerstone-content workflow |
| — | **stage-15-gsc-sitemap.md** | NEW — GSC property setup + sitemap automation |
| _internal/claude-invariants.md | _internal/claude-invariants.md | Extended — adds Astro middleware order invariant + Reoon fail-open invariant |
| _internal/local-preview.md | _internal/local-preview.md | Unchanged from v1 |
| _internal/reference-railway-automation.md | _internal/reference-railway-automation.md | Updated — commitHash gotcha section |
| _internal/reference-cloudflare-dns.md | _internal/reference-cloudflare-dns.md | Unchanged from v1 |
| _internal/reference-object-storage.md | _internal/reference-object-storage.md | Unchanged from v1 |
| _internal/reference-forms-and-persistence.md | _internal/reference-forms-and-persistence.md | Unchanged from v1 |
| — | **_internal/reference-quota-monitoring.md** | NEW — generalizes the Reoon threshold-alert pattern |
| — | **_internal/reference-aio-schema-library.md** | NEW — 8 schema component reference |
| — | **_internal/reference-prerendered-windows.md** | NEW — prerendered-dir Express middleware + dotfiles gotcha |
