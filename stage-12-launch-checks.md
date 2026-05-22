# Stage 12: Pre-Launch Checks (with AIO + GSC + schema audit)

**Last verified: 2026-05-21.**

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) run as many checks as possible autonomously via my Bash tool, Preview MCP, Chrome MCP, and Lighthouse-equivalent metrics via `preview_eval`. The user's actions are limited to: (1) reviewing items I can't autonomously verify (subjective brand voice, legal review of policy text, real testimonials authenticity), (2) approving the launch go-ahead. I categorize findings by severity so the user can scan a short summary instead of a 175-item checklist.

> **Upgrading from v1?** See [CHANGELOG.md](CHANGELOG.md). This stage adds three AIO-specific check categories (schema validation, GSC sitemap reachability, content readiness) when `aio.content_hub = true`. All v1 checks are preserved.

## AIO check category: schema validation

For every prerendered HTML file under `dist/client/`, I:

1. Count `<script type="application/ld+json">` blocks.
2. JSON.parse each block; flag any that don't parse.
3. For each parsed schema, verify `@context = "https://schema.org"` and `@type` is one of the 8 expected types (`Organization`, `WebSite`, `Person`, `Article`, `FAQPage`, `HowTo`, `Product`, `BreadcrumbList`).
4. For `Product`: verify `aggregateRating` and `review` are absent (FTC trap — see [`_internal/reference-aio-schema-library.md`](_internal/reference-aio-schema-library.md)).
5. For `Article`: verify `author`, `datePublished`, `publisher`, `mainEntityOfPage` all present.
6. For `BreadcrumbList`: verify `itemListElement` is an array with `position: 1, 2, 3...` in sequence.

Shell-level smoke check:

```bash
for f in dist/client/**/index.html; do
  count=$(grep -c '<script type="application/ld+json">' "$f" 2>/dev/null || echo 0)
  echo "$f: $count JSON-LD blocks"
done
```

Pass criteria: every page ≥1 JSON-LD block; every cornerstone ≥3 JSON-LD blocks.

Failure → 🟡 (pre-launch fix). Missing schemas don't block the launch but reduce AI-search citation surface significantly.

## AIO check category: GSC sitemap reachability

| Check | What I do | Pass criteria |
|---|---|---|
| Sitemap reachable | `curl -I https://<your-domain>/sitemap-index.xml` | HTTP 200 |
| Sitemap valid XML | `curl -s https://<your-domain>/sitemap-index.xml \| xmllint --noout -` | No errors |
| Sitemap excludes drafts | Iterate `aio.cornerstones[].slug` where `draft=true`; `grep -c` for each | 0 matches per draft |
| Sitemap includes published cornerstones | Iterate `aio.cornerstones[].slug` where `draft=false`; `grep -c` | ≥1 match per published |
| GSC sitemap submitted | `python gsc_sitemap_submit.py list <site>` | Sitemap appears with `last submitted` ≤ 24 hours old |
| GSC sitemap zero errors | Same call, check `errors:` field | 0 |
| Sitemap reachable from outside auth | If staging Basic Auth was accidentally set on prod, sitemap returns 401. `curl -I <sitemap-url>` with no auth header | HTTP 200 (not 401) |

Failure on any → 🔴 (launch blocker). Search engines can't index unreachable sitemaps. Stage 15's GSC submit flow should have caught these, but Stage 12 re-verifies.

## AIO check category: AIO content readiness

For each published cornerstone (i.e., `aio.cornerstones[]` where `draft=false`):

| Check | What I do | Pass criteria |
|---|---|---|
| Article schema present | `grep -c '"@type":"Article"' dist/client/blog/<slug>/index.html` | ≥1 |
| At least one of FAQPage/HowTo | `grep -E '"@type":"(FAQPage\|HowTo)"' dist/client/blog/<slug>/index.html` | ≥1 match |
| BreadcrumbList present | `grep -c '"@type":"BreadcrumbList"' dist/client/blog/<slug>/index.html` | ≥1 |
| AuthorBio rendered | `grep -c 'class="author-bio"' dist/client/blog/<slug>/index.html` (or whatever class the AuthorBio uses) | ≥1 |
| At least 2 internal links to other cornerstones | Parse anchor tags, intersect against `aio.cornerstones[].slug` list | ≥2 |
| RSS feed parses | `curl https://<your-domain>/blog/rss.xml \| xmllint --noout -` | No errors |
| RSS feed includes published cornerstones | `curl https://<your-domain>/blog/rss.xml \| grep -c '<title>'` | ≥ (published cornerstone count + index entry) |

Failure → 🟡 (pre-launch fix recommended). Missing internal links don't block launch but reduce topical-authority signal. Missing AuthorBio on a cornerstone is harder to fix later (the cornerstone gets cited without author attribution), so it's worth fixing now.

## Pre-launch step: Google Rich Results Test

I run a quick external-validator check on each cornerstone URL via Google's Rich Results Test:

```bash
# Test the Article schema rendering as Google would see it
curl -s -X POST "https://searchconsole.googleapis.com/v1/urlTestingTools/richResults:run" \
  -H "Authorization: Bearer $(python -c '
import sys
sys.path.insert(0, ".shared/scripts")
from oauth_helper import get_credentials
print(get_credentials().token)
')" \
  -H "Content-Type: application/json" \
  --data '{"url":"https://<your-domain>/blog/<cornerstone-slug>/","userAgent":"GOOGLEBOT"}'
```

Pass criteria: `richResults[]` includes the expected types (Article, FAQPage if present, HowTo if present) with status `VALID`.

Note: the Rich Results Test API requires the same OAuth scopes as `gsc_sitemap_submit.py` (`webmasters.readonly` is sufficient for read-only testing). If the OAuth helper isn't configured with those scopes, I fall back to a manual prompt: "Open https://search.google.com/test/rich-results and paste the URL https://<your-domain>/blog/<cornerstone-slug>/ — report what you see back."

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 12 doesn't introduce new required values; it verifies that everything Stages 0–11 set up is actually working in production. Specifically I cross-check:
- All Railway env vars from earlier stages are present on the deployed service (`railway variable list`).
- The deployed bundle contains the literal `phc_*` PostHog key, the Clarity project ID, the Resend domain — confirming Vite-bake-at-build-time worked (Invariant #1).
- The reverse proxy is reachable at `https://<your-domain>/<your-proxy-slug>/...`.
- The custom domain (if applicable) resolves and serves with valid SSL.

If any required-values gap surfaces here that should have been caught earlier, I report it and offer to backfill — Stage 12 acts as the safety net.

## What I'm doing in this stage

Running the final pre-launch checklist. After this stage, the site is genuinely ready for paid traffic / public announcement / press / etc.

**Time**: 15–30 minutes of automated checks + however long the user takes to review the legal/content drafts.

## When this stage runs

After all earlier stages complete + before any of:
- First paid ad campaign goes live
- Public press release / Product Hunt / etc.
- Switching DNS from staging to production domain (if not yet)
- Inviting affiliates to share URLs

## Categorized findings (how I report results)

I split findings into four severity buckets so the user knows what blocks launch and what doesn't:

| Bucket | Meaning | User must |
|---|---|---|
| 🔴 **Launch blocker** | Legal exposure, broken core flow, data-loss risk | Resolve before launch — I won't declare Stage 12 complete |
| 🟡 **Pre-launch fix** | Will affect early users / SEO / deliverability if shipped as-is | Strongly recommend resolving; user can accept-with-known-risk |
| 🟢 **Post-launch polish** | Won't hurt at launch but worth fixing in week 1 | Optional |
| ⚪ **Informational** | Notes for the user (e.g., "your DPA from PostHog is queued; arrives in ~10 days") | No action |

## My execution sequence

I run the checks below via my available tools. For each, I report status + bucket. At the end, I produce a single consolidated report.

### Content authenticity (FTC critical)

| Check | What I do | Bucket if fails |
|---|---|---|
| No AI-generated faces presented as real users | Re-scan source for `<img>` URLs from AI image generators; cross-check against Stage 0 punchlist resolutions | 🔴 |
| No fabricated press mentions / partnership claims | Grep for "Featured in" / "As seen in" patterns; user confirms each citation is real | 🔴 |
| No fabricated milestones | Grep for specific numeric claims; user confirms accuracy | 🔴 |
| All testimonials are real with written permission | User confirms (I can't verify this autonomously) | 🔴 if any are fabricated |
| Solo founder bio uses real photo (or styled placeholder) | Read `FounderStorySection` (or equivalent); check image URL | 🟡 if placeholder still in place |

### SEO basics

| Check | What I do | Bucket |
|---|---|---|
| Real `<title>` set on all pages | Curl each page, parse `<title>` tag | 🟡 if generic / empty |
| `<meta name="description">` 120–160 chars, accurate | Curl each page, measure description length | 🟡 |
| Real favicon | Curl `/favicon.ico`, check it's not Vite's placeholder | 🟡 |
| OG metadata (`og:title`, `og:description`, `og:image`, `og:url`, `og:type`) | Curl pages, parse OG tags | 🟡 |
| Twitter card metadata | Same | 🟡 |
| OG image 1200x630, hosted on user's domain | Fetch the OG image URL, check dimensions + host | 🟡 |
| `sitemap.xml` at root with `<lastmod>` dates | Curl `/sitemap.xml` | 🟢 |
| `robots.txt` at root with `Sitemap:` directive | Curl `/robots.txt` | 🟢 |
| All page-specific `<title>` and meta descriptions set | Crawl each route | 🟡 if any missing |
| No `noindex` on production pages | Grep build output for `noindex` | 🔴 if accidentally present |

### Analytics health

| Check | What I do | Bucket |
|---|---|---|
| PostHog Live Events shows real events | MCP query for events from last 24h | 🔴 if no events |
| All custom events fire correctly | MCP query for each event name from Stage 6's taxonomy | 🟡 if any missing |
| `deploy_variant` + `page_variant` non-null on every event | MCP query, filter for nulls | 🟡 if any nulls |
| Session recordings work, form inputs masked | Open a test session via MCP `session-recording-get` | 🟡 if masking broken |
| Clarity dashboard shows real-visit recordings | User confirms (Clarity API rate-limit makes autonomous check expensive) | 🟡 |
| All Stage 11 dashboards populate | MCP `dashboards-get-all`, then for each, query a sample insight | 🟡 if any blank |

### Privacy / compliance

| Check | What I do | Bucket |
|---|---|---|
| Banner appears in expected locales | `preview_click` to `?force_gdpr` and `?force_country=US`, check banner DOM in each | 🔴 |
| Decline blocks both PostHog AND Clarity | `preview_click` Decline, then `preview_eval('posthog.has_opted_out_capturing()')` | 🔴 |
| Accept enables both | Same with Accept | 🔴 |
| `/privacy` policy lists all sub-processors | Curl + parse | 🟡 |
| `/terms` route lives | Curl | 🟡 |
| `/consumer-health-data` (if applicable) lives + has prominent homepage link | Curl + check homepage footer for link | 🔴 if health-adjacent and missing |
| DPAs queued or signed | Read `<project>/docs/compliance-artifacts.md` | ⚪ informational |
| `?force_gdpr` dev override absent from production bundle | Build production, grep `dist/` for `force_gdpr` | 🟡 if present |

### Email deliverability

| Check | What I do | Bucket |
|---|---|---|
| Sending domain verified (DKIM + SPF) | Resend API + `dig TXT <your-domain> +short` | 🔴 if not verified |
| Test newsletter signup → real welcome email arrives | I send via curl; user confirms inbox arrival | 🔴 if no email arrives |
| Email arrives in INBOX (not spam) | User confirms | 🟡 if spam |
| Email templates render correctly in Gmail / Outlook / Apple Mail | User confirms | 🟢 |
| Email links use `${SITE_URL}` / `${APP_URL}` (not hardcoded) | Grep template files | 🟡 if hardcoded |
| Unsubscribe link present in newsletter sends | Grep template | 🔴 if absent (CAN-SPAM) |

### Form security

| Check | What I do | Bucket |
|---|---|---|
| Form endpoints validate input | Send malformed POST to each endpoint via curl | 🔴 if any accepts garbage |
| `escapeHtml` applied to user input before HTML interpolation | Read template functions | 🔴 if any uses raw input |
| Form size limits enforced (`limit: "32kb"` on `express.json()`) | Read `server.js` | 🟡 |
| Rate-limiting on form endpoints (sustained POST flood) | Send 10 rapid POSTs | 🟡 if all succeed |

### Membership / payment (if Stage 9)

| Check | What I do | Bucket |
|---|---|---|
| Webhook endpoint signature-verifies | Send unsigned POST to `/api/webhooks/<platform>` (e.g., `/stripe`, `/whop`); confirm 401/400 response | 🔴 if accepts |
| Test purchase flows: checkout → webhook → PostHog event → DB | User makes a $1 test purchase via the chosen payment platform; I verify via MCP + DB query | 🟡 |
| Refund / cancellation paths emit correct events | User triggers refund via the platform's dashboard; I verify the inbound webhook fires the matching PostHog event | 🟢 |
| Pricing on site matches platform source-of-truth | Compare pricing component values vs the platform's API plan data (Stripe Price object / Whop Plan / etc.) | 🟡 if drift |
| Plan IDs in `affiliate-link.ts` match actual platform IDs | Same | 🔴 if mismatch (broken checkout) |

### Affiliate (if Stage 10)

| Check | What I do | Bucket |
|---|---|---|
| `?a=test` URL captures correctly | `preview_click` to `/?a=test`, `preview_eval` reads cookie + localStorage + super-property | 🔴 if any layer fails |
| Test affiliate signup flow | I sign up via dev path with `?a=test`; verify DB attribution row | 🟡 |
| Switchy short URL resolves correctly | Curl test short URL, check 302 + query string preserved | 🔴 if broken |
| OG metadata baked into Switchy short URL | curl short URL HTML, parse OG tags | 🟡 |
| Daily sync scheduled task active | Check scheduler | 🟡 if missing |

### Server / Railway

| Check | What I do | Bucket |
|---|---|---|
| Service running on `0.0.0.0:$PORT` | `curl -I https://<your-domain>/` | 🔴 if 502 |
| Health checks pass | Curl + check `x-railway-edge` header | 🟡 if missing |
| All env vars set | `railway variable list` | 🟡 if any expected vars unset |
| Build picks up `RAILWAY_GIT_COMMIT_SHA` for deploy_id | View deployed bundle source, search for SHA | 🟡 if literal "devlocal" |
| `RAILWAY_CHECKOUT_DEPTH=0` set | `railway variable get RAILWAY_CHECKOUT_DEPTH` | 🟡 if missing (variant labels break) |
| Logs show no startup warnings | `railway logs` for 30 sec (current CLI streams by default; older `--tail` flag was removed) | 🟡 |
| Auto-deploy from GitHub on `main` push | Make a trivial commit + push, watch logs | 🟢 |

### DNS / domain

| Check | What I do | Bucket |
|---|---|---|
| Custom domain resolves to Railway | `dig +short <your-domain>` | 🔴 if not |
| HTTPS works | Curl `https://<your-domain>` | 🔴 |
| HTTP → HTTPS redirect | Curl `http://<your-domain>`, check 301 | 🔴 |
| Service worker tombstone (if migrating) tested | I open old URL via Chrome MCP if available, verify console message | 🟡 |
| 301 redirects work for every old PWA path | Curl each `PWA_PATH_PREFIXES` entry on the old domain | 🔴 if any returns 200 |
| External dashboard updates done | User confirms (Google Cloud Console OAuth, payment-platform webhook URL — Stripe / Whop / etc., ad platform pixels, etc.) | 🔴 if migrating and any missed |

### Performance baseline

I run Lighthouse-equivalent metrics via `preview_eval`:

```js
// preview_eval payload — I run this and parse the result:
JSON.stringify({
  navigation: performance.getEntriesByType("navigation")[0],
  paint: performance.getEntriesByType("paint"),
  largestContentfulPaint: (() => {
    const entries = performance.getEntriesByType("largest-contentful-paint");
    return entries.length > 0 ? entries[entries.length - 1] : null;
  })(),
})
```

| Check | What I do | Bucket |
|---|---|---|
| FCP < 1.5s on mobile | Run on simulated 3G via `preview_resize` to mobile + measure | 🟡 if > 2s |
| LCP < 2.5s on mobile | Same | 🟡 if > 4s |
| CLS < 0.1 | Same | 🟡 if > 0.25 |
| Bundle gzipped JS reasonable | Read `dist/` size | 🟢 |
| No render-blocking third-party scripts before main content | Inspect `<head>` of deployed HTML | 🟡 |
| Images optimized | Check image dimensions vs displayed size | 🟢 |

For full Lighthouse scoring (accessibility, best-practices, SEO categories), I fall back to running `npx lighthouse <url> --output=json --output-path=./lighthouse-report.json` and reading the JSON.

### Mobile + cross-browser

| Check | What I do | Bucket |
|---|---|---|
| Mobile layout works at 375x667 | `preview_resize` + screenshot | 🟡 if broken |
| Touch targets ≥ 44x44 px (iOS HIG) | `preview_inspect` on CTA buttons | 🟡 |
| Horizontal scroll absent | `preview_eval('document.documentElement.scrollWidth - document.documentElement.clientWidth')` should be ≤ 0 | 🟡 |
| Viewport meta tag correct | Curl + parse | 🔴 if missing |

### Accessibility (Lighthouse + manual where needed)

| Check | What I do | Bucket |
|---|---|---|
| Lighthouse accessibility score | `npx lighthouse --only=accessibility` | 🟡 if < 90 |
| All `<img>` have `alt` text | Grep build output | 🟡 |
| Form fields have labels | Same | 🟡 |
| Color contrast ≥ 4.5:1 for body text | Lighthouse | 🟡 |
| Keyboard navigation works | Manual user check (I can't autonomously verify focus indicators) | 🟢 |

### Documentation

| Check | What I do | Bucket |
|---|---|---|
| `migration-punchlist.md` 🔴 + 🟡 items resolved | Read file | 🔴 if any 🔴 unresolved |
| `<project>/docs/archived-platform-deps.md` lists migrations | Read | 🟢 |
| `README.md` exists with project overview | Read | 🟢 |
| Saved skill config at `<project>/.skill-config.json` | Read | ⚪ informational |
| Operator runbook for data-subject requests | Read | 🟡 (Stage 8 should have created) |

## End-to-end smoke test

I run a final happy-path test:

1. Fresh incognito browser session via Preview MCP
2. Navigate to home → click around: home → pricing → about → contact
3. Submit newsletter form with test email
4. (If applicable) Click pricing CTA → verify the payment platform's checkout (Stripe Checkout / Whop / etc.) opens with the correct plan + affiliate code preserved in the URL or session
5. Take screenshots at each step for the user's review
6. Confirm via PostHog MCP that all events fired with correct super-properties

If any step fails, that's a 🔴 launch blocker.

## My consolidated report

After all checks, I produce a single summary the user can scan in a minute:

```
🚀 Stage 12 PRE-LAUNCH STATUS

🔴 Launch blockers (must resolve before launch): 0
   ✓ All clear

🟡 Pre-launch fixes (strongly recommend): 3
   • OG image dimensions are 800x400 (should be 1200x630). Affects social
     unfurl quality. Fix before first paid ad campaign.
   • Email template hardcodes "https://yourdomain.com" in one place
     (newsletter footer). Should source from SITE_URL env var.
   • Lighthouse accessibility score is 88. Mostly missing alt text on
     3 hero images. ~15 min fix.

🟢 Post-launch polish (nice to have): 4
   • sitemap.xml + robots.txt
   • Header `<title>` could be more SEO-targeted
   • Bundle could shed ~30 KB by lazy-loading the contact form module
   • README.md is sparse

⚪ Informational:
   • DPA from PostHog: requested, expected to arrive in 7–10 days
   • Payment-platform webhook test purchase (Stripe / Whop / etc.): not
     yet performed (recommend doing this once before driving paid traffic)

E2E smoke test: ✅ all 6 steps passed (screenshots saved)

Want me to fix the 🟡 items now (estimated 30 min), or are you ready
to launch as-is and circle back?
```

## Post-launch monitoring

I queue these alerts/monitors for the first week post-launch (these go into the data-subject-runbook + a new monitoring section):

- **PostHog**: alert on `subscription_canceled` rate >5% in any 24h window (early churn signal)
- **Railway**: confirm health checks pass; alert on >1% error rate
- **Resend**: dashboard → Logs → confirm send rate matches signup rate
- **Cloudflare**: monitor for unusual traffic patterns
- **Clarity**: install the Live browser extension for ongoing spot-checking

## Pre-launch risk register

Even after this checklist, I document residual risks for the user:

- **First-purchase paths untested at scale**: a subtle bug in checkout / webhook can hide until 10+ purchases land
- **DNS propagation gotchas**: some users may land on the OLD service for 24h post-cutover (if migrating)
- **Email deliverability cliff**: hitting a major provider's spam filter mid-launch is hard to recover from
- **Ad platform attribution lag**: Meta/Google show data with 6–24h delay; first-hour conversion data is noisy

I document the rollback procedure inline (Railway: detach domain, reattach to staging — DNS is already low-TTL'd).

## Outputs

After Stage 12:
- A genuinely launch-ready site
- Categorized findings report (🔴/🟡/🟢/⚪)
- All 🔴 items resolved or explicitly accepted-with-known-risk
- Smoke test screenshots saved
- Post-launch monitoring queued
- Risk register documented for the user

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Pre-launch checklist across content, SEO, analytics, privacy, email, payments, affiliates, performance, accessibility. |
| 2026-05-08 | Reframed for non-coder audience: replaced 175+ undifferentiated checklist items with severity-bucketed report (🔴 launch blocker / 🟡 pre-launch fix / 🟢 polish / ⚪ informational). All checks reframed as "I run X autonomously" with the user only confirming subjective items I can't verify (real testimonials authenticity, brand voice, inbox spam-folder placement, brand-appropriate copy). Final consolidated report shows scannable summary instead of a 175-item list. |
