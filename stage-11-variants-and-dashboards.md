# Stage 11: Variant System + Dashboards

**Last verified: 2026-05-08.**

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) write the variants JSON, the build-time injection in `vite.config.ts`, the runtime helpers, the per-page tagging, the Clarity-side mirror, and create 6+ PostHog dashboards via the PostHog MCP. The user's actions: (1) confirm which pages should have per-page variant tracking, (2) confirm the variant archive approach (local-only / live URLs / screenshot archive). Everything else autonomous.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 11 needs:
- `variants.enabled` — if `false`, this stage skips dashboard creation but I still ship the variant infrastructure (cheap to add later).
- `variants.tracked_pages` — array of paths. If unset, default to `["/", "/about", "/services", "/pricing", "/contact"]` (most common marketing pages) and confirm with the user.
- `variants.archive_approach` — `"local_only"` / `"live_urls"` / `"screenshot_archive"` / `"skip"`. If `"screenshot_archive"`, I cross-check `storage.provider` (must be set; if unset, I run the storage decision flow from `_internal/reference-object-storage.md`).
- `analytics.posthog.project_id` — required for dashboard creation via MCP. Set during Stage 6; if missing, that means Stage 6 didn't run and I block.
- `membership.enabled` and `affiliate.enabled` — drive whether Dashboards 4–5 (sales/revenue) and 7 (affiliate) get created.

## What I'm doing in this stage

Operationalizing the deploy identifiers from Stage 6 with:

1. A user-managed `current-variants.json` schema with global cohort marker AND per-page lineage tracking
2. Build-time injection so every page knows its `page_variant` at runtime
3. PostHog dashboards filtering by `deploy_variant` + `page_variant` for clean cohort comparison
4. A command vocabulary so future sessions have consistent verbs ("bump global", "bump page X", "lock in page X")

After this stage, the user can iterate page-by-page on CRO with clean lineage tracking AND have 6+ dashboards answering common questions.

**Time**: 30–45 minutes of automated work, plus user-confirmation of tracked pages and archive approach.

## The dual-variant model

This skill uses a sequential single-page CRO methodology — the practitioner optimizes pages one at a time, hitting target metrics on a focal page before moving on. The variant tagging supports this with TWO properties on every event:

| Property | What it means | When it bumps |
|---|---|---|
| `deploy_variant` (global, prefix `gvar-`) | Chronological cohort marker — "what iteration of the SITE was running when this event fired?" | When a new test cycle starts (focus shifts to a new page) |
| `page_variant` (per-page, prefix `pvar-`) | Page lineage — "what iteration of THIS specific page was running?" | Only when actively iterating on that specific page |

The `gvar-` / `pvar-` prefixes are conventions. Users who prefer different prefixes can override during onboarding — the structure stays the same.

Sequential-page methodology in practice: with single global tracking, the variant label encodes the SUM of all tests across all pages, making it impossible to ask "did pvar-y of opt-in convert better than pvar-x?" cleanly. The dual model fixes that.

## Prerequisites

- Stage 6 complete (PostHog wired, deploy identifiers as super-properties).
- Stage 7 complete (Clarity wired with deploy-tag forwarding).
- A planning sense of which pages need optimization tracking. If the user is unsure, I suggest a sensible default based on their site type (typically: home, pricing, the main sales page, contact).

## My execution sequence

### Step 1: I write the schema file

I create `src/_core/analytics/current-variants.json`:

```json
{
  "global": {
    "variant": "gvar-a",
    "label": "Initial baseline",
    "started_at": "2026-05-08"
  },
  "pages": {
    "/": {
      "variant": "pvar-a",
      "label": "Initial baseline",
      "status": "untested"
    },
    "/about": {
      "variant": "pvar-a",
      "label": "Initial baseline",
      "status": "untested"
    },
    "/pricing": {
      "variant": "pvar-a",
      "label": "Initial baseline",
      "status": "untested"
    }
  }
}
```

I populate the `pages` map from the user's tracked-pages list (confirmed during onboarding or asked here):

```
Which pages do you want to track per-page variants on? I'll auto-suggest
based on common patterns for your site type:

  Suggested:
    / (home)
    /about
    /pricing
    /contact

  Anything else? Common additions:
    - /<sales-page-slug> (if you have a long-form sales page)
    - /<lead-magnet> (if you offer a downloadable)
    - /<onboarding-step-X> (if you have a multi-step funnel)

Confirm or edit the list. Pages NOT in the map fall back to the global
variant; that's fine for catch-all routes (404, dynamic).
```

Field guide:
- `global.variant` — string matching `/^gvar-[a-z]+$/` (a → z → aa → ...)
- `global.label` — human-readable summary of what's currently being tested
- `global.started_at` — ISO date when this global variant was bumped
- `pages[path].variant` — `/^pvar-[a-z]+$/`
- `pages[path].label` — what's being tested on this page
- `pages[path].status` — `untested` / `active` / `locked_in`. Optional `locked_in_at` ISO date when status flipped.

### Step 2: I add build-time injection to `vite.config.ts`

I update Stage 6's `vite.config.ts` to read the JSON and inject the per-page map:

```ts
import fs from "node:fs";
import { resolve } from "node:path";

function readVariants(): { global: string; pages: Record<string, string> } {
  const fallback = { global: "gvar-unknown", pages: {} as Record<string, string> };
  try {
    const filePath = resolve(__dirname, "src", "_core", "analytics", "current-variants.json");
    const raw = fs.readFileSync(filePath, "utf8");
    const parsed = JSON.parse(raw);

    const globalVariant = typeof parsed.global?.variant === "string" && /^gvar-[a-z]+$/.test(parsed.global.variant)
      ? parsed.global.variant
      : "gvar-unknown";

    const pages: Record<string, string> = {};
    for (const [path, entry] of Object.entries(parsed.pages ?? {})) {
      const v = (entry as { variant?: unknown }).variant;
      if (typeof v === "string" && /^pvar-[a-z]+$/.test(v)) pages[path] = v;
    }

    return { global: globalVariant, pages };
  } catch {
    return fallback;
  }
}

const VARIANTS = readVariants();
const DEPLOY_VARIANT = VARIANTS.global;
const PAGE_VARIANTS = VARIANTS.pages;

// In the define block:
export default defineConfig({
  define: {
    "import.meta.env.VITE_DEPLOY_ID": JSON.stringify(deriveDeployId()),
    "import.meta.env.VITE_DEPLOY_VARIANT": JSON.stringify(DEPLOY_VARIANT),
    "import.meta.env.VITE_DEPLOY_VARIANT_SHA": JSON.stringify(deriveDeployVariantSha()),
    "__PAGE_VARIANTS__": JSON.stringify(PAGE_VARIANTS),  // injected as literal object
  },
  // ...
});
```

### Step 3: I add the runtime helper

I update `src/_core/analytics/track.ts`:

```ts
declare const __PAGE_VARIANTS__: Record<string, string>;

export function getPageVariantForPath(pathname?: string): string {
  const path = pathname ?? (typeof window !== "undefined" ? window.location.pathname : "/");
  if (path in __PAGE_VARIANTS__) return __PAGE_VARIANTS__[path];
  return (import.meta.env.VITE_DEPLOY_VARIANT as string) || "pvar-unknown";
}

export function track(event: string, props?: EventProps): void {
  try {
    if (posthog.__loaded) {
      posthog.capture(event, {
        page_variant: getPageVariantForPath(),
        ...props,
      });
    }
  } catch { /* silent */ }
}
```

### Step 4: I update pageview events to include `page_variant`

I update both `main.tsx` (initial pageview) and the router's `useEffect` (subsequent navigation) to include `page_variant`:

```ts
// main.tsx loaded callback:
ph.capture("$pageview", {
  $current_url: window.location.href,
  page_variant: getPageVariantForPath(),
});

// router useEffect:
posthog.capture("$pageview", {
  $current_url: window.location.href,
  page_variant: getPageVariantForPath(pathname),
});
```

### Step 5: I add the Clarity-side per-page tag

I extend `src/_core/analytics/deploy-tag.ts`:

```ts
declare const __PAGE_VARIANTS__: Record<string, string>;

export function setPageVariantOnClarity(pathname?: string): void {
  if (typeof window === "undefined") return;
  const clarity = window.clarity;
  if (typeof clarity !== "function") return;
  const path = pathname ?? window.location.pathname;
  const pageVariant =
    __PAGE_VARIANTS__[path] ??
    (import.meta.env.VITE_DEPLOY_VARIANT as string) ??
    "pvar-unknown";
  clarity("set", "page_variant", pageVariant);
}
```

I call it from `initAnalytics()` (initial mount) AND from the router on every route change.

### Step 6: I document the user's command vocabulary

I update the saved skill config + a small `<project>/docs/variants-runbook.md` documenting the verbs the user can use in future sessions:

| User says | I do |
|---|---|
| "Show variant state" | Read `current-variants.json`, display global + per-page table with status |
| "Bump global to gvar-X with label 'Y'" | Edit global section, increment letter, set label + started_at, commit + push, verify deploy |
| "Bump page X" | Increment that page's variant letter, ask for label, set status: `active`, commit + push, verify deploy |
| "Lock in page X" | Set page X's status: `locked_in`, locked_in_at: today, commit + push, verify deploy |
| "Re-test page X" | Bump that page's variant letter (continuing lineage), status: `active`, commit + push |
| "Compare gvar-a vs gvar-b" | Pull all metrics filtered by both variants side-by-side via PostHog MCP |

### Step 7: I create the dashboards via PostHog MCP

I create 6+ saved dashboards using the PostHog MCP tools. The relevant tools (verified 2026-05):

| Tool | What it does |
|---|---|
| `mcp__posthog__dashboard-create` | Creates an empty dashboard, returns dashboard ID |
| `mcp__posthog__query-validate` | Validates a HogQL query against the project's schema before creating an insight (catches typos that would silently produce empty insights) |
| `mcp__posthog__insight-create` | Creates an insight (Trend / Funnel / HogQL / Retention / etc.) and optionally attaches it to a dashboard |
| `mcp__posthog__query-run` | Runs a HogQL query directly and returns results (used during validation to confirm I'm getting the data shape I expect before saving as an insight) |
| `mcp__posthog__dashboard-reorder-tiles` | Sorts insights on a dashboard into the order I want (cards default to creation order, which is rarely the right order) |

#### The per-insight flow

For every insight in every dashboard below, I follow this pattern:

```
1. Write the query body (HogQL string for HogQL insights, or trends/funnel
   filter object for those types — the MCP accepts both).
2. Call query-validate with the query body.
   → If invalid: I fix the query and re-validate. I do NOT create an insight
     against an unvalidated query — silent empty-insights waste user time.
3. Call query-run with the validated query.
   → If results are zero rows AND the user hasn't generated test traffic yet:
     I note this and proceed (the insight will populate once traffic arrives).
   → If results look wrong (missing breakdown, wrong dimensions): I revise and
     restart at step 1.
4. Call insight-create with the validated query body + name + description +
   dashboardId. I save the returned insightId for later reference.
```

**Worked example — "Per-affiliate scoreboard" HogQL insight on Dashboard 7:**

```js
// Step 1: write the query
const query = {
  kind: "HogQLQuery",
  query: `
    SELECT
      properties.affiliate_code AS affiliate,
      countIf(event = '$pageview') AS pageviews,
      countIf(event = 'newsletter_optin_completed') AS optins,
      countIf(event = 'subscription_started') AS subscriptions,
      sumIf(toFloat(properties.revenue), event = 'subscription_started') AS revenue
    FROM events
    WHERE timestamp >= now() - INTERVAL 90 DAY
      AND properties.affiliate_code IS NOT NULL
    GROUP BY affiliate
    ORDER BY revenue DESC
    LIMIT 50
  `,
};

// Step 2: validate
const validation = await mcp.posthog.query_validate({ query });
if (!validation.valid) throw new Error(`HogQL invalid: ${validation.errors.join("; ")}`);

// Step 3: smoke-test
const sample = await mcp.posthog.query_run({ query });
console.log(`Sample returned ${sample.results.length} rows`);

// Step 4: create the insight, attached to the affiliate dashboard
const insight = await mcp.posthog.insight_create({
  name: "Per-affiliate scoreboard (last 90 days)",
  description: "Top affiliates by tracked revenue. Updates daily.",
  query,
  dashboards: [affiliateDashboardId],
});
```

I follow this exact flow for all ~30+ insights across the 6–7 dashboards. The pattern is mechanical once I know the dashboard ID and the query body; every dashboard description below becomes one such loop.

#### Dashboard 1: Variant Performance Overview (daily-use)

```
Active variant — Big Number — current `deploy_variant` for the last 24h
Pageviews per day — Trend — $pageview, breakdown by deploy_variant
Newsletter opt-in funnel — Funnel — $pageview → newsletter_optin_completed, breakdown by variant
Subscription events — Trend — subscription_started + _renewed + _canceled stacked
Top CTAs — Trend — cta_click ordered by count
Custom events today — Table — top 20 events from last 24h
```

#### Dashboard 2: Opt-In Funnel

```
Newsletter opt-in funnel by variant — Funnel — $pageview → newsletter_optin_started → newsletter_optin_completed, breakdown by variant
Newsletter opt-ins by page — Trend — newsletter_optin_completed grouped by $pathname
Newsletter started → opted-in drop-off — Funnel — _started → _completed
Contact form submissions by topic — Trend — contact_form_completed grouped by topic
```

#### Dashboard 3: CTA Performance

```
CTA clicks ranked by label — Trend — cta_click ordered by count
CTA clicks per variant — Table — cta_click grouped by deploy_variant
Pricing card view → click conversion — Funnel — pricing_card_viewed → pricing_card_clicked
CTA effectiveness by position — Trend — cta_click grouped by cta_position
```

#### Dashboard 4: Sales Funnel (if Stage 9 is in scope)

```
Full sales funnel — Funnel — $pageview → pricing_card_viewed → pricing_card_clicked → app_signup_redirect → signup_completed → subscription_started, breakdown by deploy_variant
Pricing-tier conversion rate — Funnel — pricing_card_viewed → pricing_card_clicked, broken down by tier
Subscriptions started by tier — Trend — subscription_started grouped by tier
Drop-off pages before purchase — HogQL — find users who saw pricing but didn't subscribe within 1 hour, return their last pageview path
Pricing → subscribe conversion rate by variant — Funnel — pricing_card_clicked → subscription_started, breakdown by deploy_variant
```

#### Dashboard 5: Revenue by Variant (if Stage 9 is in scope)

```
Total revenue by variant — Trend — sum(properties.revenue) on subscription_started, breakdown by deploy_variant
ARPU by variant — HogQL — sum(revenue) / unique paying users per variant
Tier mix by variant — Trend — subscription_started grouped by tier × deploy_variant (heatmap)
Renewal vs new revenue mix — Trend — subscription_started vs subscription_renewed area chart
Cancel/refund/dispute rates by variant — Trend — count of subscription_canceled + _refunded + _disputed per variant
```

#### Dashboard 6: UTM × Variant Cross-Tab

```
UTM source × variant — Trend — $pageview grouped by utm_source × deploy_variant
UTM campaign × variant — HogQL — conversion_rate per (utm_campaign, deploy_variant)
UTM medium breakdown — Trend — $pageview grouped by utm_medium
UTM source × newsletter conversion — HogQL — newsletter_optin_completed / $pageview per utm_source
Sub-platform sessions (utm_placement rollup) — HogQL — count grouped by utm_placement prefix
```

#### Dashboard 7 (only if Stage 10 is in scope): Affiliate Program

```
Affiliate channel summary — Trend — affiliate-attributed pageviews/opt-ins/subscriptions/revenue per month
Per-affiliate scoreboard — HogQL — top affiliates by tracked revenue last 90 days
Affiliate funnel — Funnel — pageview → opt-in → subscribed, restricted to is_affiliate_attributed=true
Affiliate × variant cross-tab — HogQL — per-(affiliate, deploy_variant) cell
Affiliate gross revenue + estimated commission — HogQL
Affiliates seen for first time (last 30 days) — HogQL — first-touch by affiliate_code
```

I create each dashboard + insight via MCP calls. The user doesn't need to click through PostHog manually.

### Step 8: I write the runbook

I create `<project>/docs/dashboards-runbook.md` covering:

- Which dashboard answers which question (linked URLs once dashboards are created)
- How to read each card (what's "good")
- When to bump the variant (signal patterns)
- How to investigate a sudden conversion drop
- How to add a new event to the taxonomy
- How to add a new dashboard

Sample threshold I include: a variant beats another when sample size ≥ 100 visitors per variant, relative difference ≥ 20% on the headline metric, gap holds ≥ 3 days, gap visible on ≥ 2 dashboards, and channel-controlled (UTM × Variant cross-tab confirms it's not a channel mix shift).

### Step 9: Variant archive — preserving past iterations

Bumping a variant overwrites the live experience. If the user wants to compare gvar-a vs gvar-b SIDE BY SIDE later, they need an archive. Three approaches; the user picked during onboarding (or I prompt now if deferred).

#### Approach A: Local-only via git checkout (default — free)

Every variant bump is a git commit. To view the old variant later, the user checks out the parent commit and runs locally. I tag every variant bump:

```
git tag -a "variant-gvar-a-final" <commit-sha> -m "Final state of gvar-a"
git push origin --tags
```

Future: `git checkout variant-gvar-a-final` + `npm run dev` shows the old UI.

**Cost**: zero infrastructure.
**Use**: solo dev review, "what did the old hero look like?", design archeology.
**Limitation**: no shareable URL, no analytics on old variant.

#### Approach B: Multiple Railway services per variant (live URLs)

Each significant variant gets its own Railway service connected to its git tag. Subdomain per variant: `gvar-a.<your-domain>`, `gvar-b.<your-domain>`, etc.

I write the deployment script that creates a new Railway service from a git tag when the user invokes "archive gvar-X." The user signs the new Railway service's OAuth approve once; subsequent variant archives autonomous.

**Cost**: Railway resources per archived service (~$5/mo per service). At 6 archived variants, $30/mo ongoing.
**Use**: driving paid traffic to old variants, sharing frozen variants with stakeholders.
**Pruning**: every 3 months, I review and tear down anything older than the last 3 milestones.

#### Approach C: Static screenshot archive (Playwright + cloud storage)

Post-deploy I capture screenshots of every key page at every variant bump and upload to Backblaze B2 (or S3). I write a Playwright script + an `/admin/variants` viewer page that reads the manifest.

I write the script with the user's tracked-pages list and their B2/S3 credentials. The capture runs on every Railway deploy via a webhook hook.

**Cost**: B2 storage (10 GB free tier covers most projects) + CI minutes.
**Use**: visual evolution timeline, comparison without paying for live deploys.
**Limitation**: static — can't interact with archived variants.

#### My default recommendation by user type

- **non_coder marketer**: Approach A (zero cost, simple)
- **developer with active CRO program**: Approach A + Approach C (visual archive without monthly cost)
- **expert with paid traffic to test against old variants**: A + B + C (full coverage)

I default to A + offer C as upgrade once first variant ships.

### Step 10: I cross-check multi-site sync (if applicable)

If the user has multiple sites sharing one PostHog project (e.g., marketing + app), I check that the variant tagging is consistent. Each site's `current-variants.json` is independent; they don't need to match, but I flag if the global variants drift in a way that would confuse cross-site dashboards.

## Verification (autonomous)

I run these myself before declaring Stage 11 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| `current-variants.json` exists | Read file | Valid JSON with global + at least 1 page entry |
| Build picks up `__PAGE_VARIANTS__` | `npm run build`, grep bundle | Literal object, not the placeholder string |
| Pageviews include `deploy_variant` AND `page_variant` | MCP query for recent `$pageview` | Both properties non-null |
| Custom events include `page_variant` | MCP query for any custom event | `page_variant` property present |
| Clarity recordings tagged | Check Clarity dashboard via MCP (or manual if MCP not installed) | `deploy_variant` + `page_variant` as custom tags |
| Dashboards created | `mcp__posthog__dashboards-get-all` | 6 (or 7 if Stage 10) dashboards present |
| Runbook exists | Read file | Present with all sections populated |

I report:

```
✅ Stage 11 complete:
   • current-variants.json scaffolded (global: gvar-a, 5 pages: all pvar-a)
   • Build-time injection wired in vite.config.ts
   • Pageview + custom events include both deploy_variant + page_variant
   • Clarity tags mirror PostHog properties
   • 6 PostHog dashboards created:
     - Variant Performance Overview (daily-use)
     - Opt-In Funnel
     - CTA Performance
     - Sales Funnel (Stage 9 in scope)
     - Revenue by Variant (Stage 9 in scope)
     - UTM × Variant Cross-Tab
   • Variant archive: local-only (Approach A) — I tag every bump
   • Runbook at docs/dashboards-runbook.md
   • Command vocabulary saved to .skill-config.json

Ready for Stage 12 (pre-launch checks)?
```

## Common gotchas I handle automatically

- **JSON typo prevents build-time injection**: `vite.config.ts` falls back to `gvar-unknown` if parse fails. I watch for this in Live Events; bump immediately if seen.
- **Page entry path doesn't match React Router path**: keys must match exactly what `useLocation().pathname` returns. I always normalize trailing slashes.
- **Async events fired after navigation**: an event captured 200ms after a click might run after the user navigated. I document this in the runbook.
- **Server-side events get `page_variant: undefined`**: server has no pathname context. Expected; documented.
- **Old events** (pre-this-deploy) don't have `page_variant`: dashboards filtering on it see null. Documented.

## Outputs

After Stage 11:
- Dual-variant model wired (global + per-page)
- User-managed `current-variants.json` with build-time injection
- 6+ PostHog dashboards live (number depends on which stages are in scope)
- Runbook for interpretation + maintenance
- Command vocabulary documented for future sessions
- Variant archive approach selected and (if A or C) implemented

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Captures sequential single-page CRO variant pattern: global cohort marker (gvar-X) + per-page lineage (pvar-X), build-time injection via Vite define{}, PostHog super-property + Clarity custom tag dual-registration, 6–7-dashboard set with HogQL recipes. |
| 2026-05-08 | Reframed for non-coder audience: replaced "Run X" / "Edit Y" with first-person Claude voice. Replaced trademarked methodology references with generic "sequential single-page CRO methodology" terminology. The `gvar-` / `pvar-` prefix convention is now described as a default users can override during onboarding rather than a fixed standard. Dashboard creation flow reframed from "create these in PostHog UI" to "I create them via the PostHog MCP." Variant archive section reorganized with explicit default by user-expertise mode. |
