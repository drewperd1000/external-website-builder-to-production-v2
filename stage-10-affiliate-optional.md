# Stage 10: Affiliate Tracking (Conditional)

**Last verified: 2026-05-08.**

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) write the multi-layer attribution: marketing-site capture, DB attribution table + helper, webhook handler join, Switchy short-link helper script, daily sync task, affiliate registry. The user's actions: (1) sign up for Switchy and configure the branded short-URL domain (~10 min in dashboard), (2) generate Switchy API token and paste it. Everything else autonomous.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 10 needs:
- `affiliate.enabled` — if `false`, this stage is skipped entirely.
- `affiliate.commission_rate` — informational (used in registry doc); default 30% if unset.
- `project.brand_prefix` — embedded in the affiliate cookie + localStorage key names (`<prefix>_affiliate_code`). If missing, I prompt per the same fallback as Stage 8.
- `short_urls.enabled` and `short_urls.domain` — if affiliate uses a short-URL indirection layer (recommended; protects against platform username renames). If `enabled` is true but `domain` is unset, I prompt; the user provides the domain they registered for short URLs.
- `hosting.cloudflare_token` — drives Path A (autonomous CNAME via CF API) vs Path B (paste-ready DNS instructions) for Switchy domain setup.
- The user's Switchy API token, pasted into `<project>/.secrets/switchy-api-key.txt`. **Parallel work**: I can write the capture module, DB attribution helper, webhook-handler join, short-link helper script, and daily-sync task while the user does Switchy signup.

## What I'm doing in this stage

If the site runs an affiliate program: wiring multi-layer attribution that survives cookie deletion, Safari ITP, AND ad-platform username changes.

After this stage:
- Marketing site captures `?ref=<handle>` or `?a=<handle>` on landing → cookie + localStorage + PostHog super-property
- App / signup endpoint persists the affiliate code to a DB attribution table keyed by user
- Membership webhook (Stage 9) joins on the attribution table to attach `affiliate_code` to subscription events
- A URL shortener (Switchy or equivalent) sits between affiliate-shared URLs and the actual destination, providing stable indirection that survives platform username renames
- OG metadata on the marketing site enables affiliate URL unfurling in iMessage / Slack / WhatsApp

**Time**: 30–60 minutes. Plus DNS propagation for the branded short-URL domain (5 min – several hours, runs in parallel).

**Skip this stage** if onboarding said no affiliate program planned.

## Why multi-layer attribution

The verified gotcha (2026-04-29) that drives this stage's complexity: **when an affiliate's username is renamed in Whop, OLD `?a=<old-username>` URLs DO NOT credit attribution**. Whop validates against current username at purchase time. Old usernames silently lose crediting.

This means the **URL shortener indirection layer is ESSENTIAL, NOT OPTIONAL**. Without it, every Whop username rename silently breaks every affiliate URL in the wild — social posts, print materials, podcast show notes, email signatures all stop crediting until re-shared. With the shortener, short URLs are stable forever; I update the destination invisibly.

I always recommend a shortener, even if onboarding flagged "no shortener." If the user explicitly accepted the risk, I document it in `<project>/.skill-config.json` and proceed without one.

## Prerequisites

- Stage 9 complete (membership webhook live).
- Affiliate program enabled on the membership platform (Whop / Trackdesk / etc.). For Whop specifically, the user enables this in the Whop dashboard's Affiliate Program section.
- A URL shortener choice from onboarding. If deferred, I prompt now.

## My execution sequence

### Step 1: I write the marketing-site capture layer

I implement `src/_core/analytics/affiliate-capture.ts` (Stage 6 left a stub):

```ts
/**
 * Affiliate-code capture — captures ?ref=<handle> / ?a=<handle> on
 * marketing-site arrival, persists across the attribution window, and
 * registers as a PostHog super-property.
 *
 * Persistence layers (multi-channel for ITP-resilience):
 *   1. Cookie on parent domain (.<your-domain>) so it flows to subdomains
 *   2. localStorage backup
 *   3. PostHog super-property (every event in the session)
 *
 * Once user signs up, the app reads from cookie/localStorage and writes a
 * row to affiliate_attributions keyed to user_id. From that point our DB
 * is the durable source of truth.
 *
 * Must be called BEFORE first posthog.capture() — super-properties only
 * attach to events fired after register().
 */
import posthog from "posthog-js";

const REF_PARAM_KEYS = ["ref", "a"] as const;
const COOKIE_NAME = "<brand-prefix>_affiliate_code";  // I substitute brand prefix
const STORAGE_KEY = "<brand-prefix>_affiliate_code";
const COOKIE_MAX_AGE_SECONDS = 30 * 24 * 60 * 60;  // 30 days

function readAffiliateCodeFromUrl(): string | null {
  if (typeof window === "undefined") return null;
  const params = new URLSearchParams(window.location.search);
  for (const key of REF_PARAM_KEYS) {
    const raw = params.get(key);
    if (raw !== null && raw !== "") {
      const trimmed = raw.trim().toLowerCase();
      if (trimmed.length > 0 && trimmed.length <= 64) return trimmed;
    }
  }
  return null;
}

function setAffiliateCookie(value: string): void {
  if (typeof document === "undefined") return;
  // Set on parent domain so it flows to subdomains.
  // I substitute the user's actual domain at write time.
  const isProd = window.location.hostname.endsWith("<user-domain>");
  const domainAttr = isProd ? "; domain=.<user-domain>" : "";
  const secureAttr = window.location.protocol === "https:" ? "; secure" : "";
  document.cookie = `${COOKIE_NAME}=${encodeURIComponent(value)}; max-age=${COOKIE_MAX_AGE_SECONDS}; path=/${domainAttr}; samesite=lax${secureAttr}`;
}

function setAffiliateLocalStorage(value: string): void {
  if (typeof window === "undefined") return;
  try {
    window.localStorage.setItem(STORAGE_KEY, value);
  } catch { /* private mode */ }
}

export function registerAffiliateCodeOnPostHog(): string | null {
  const code = readAffiliateCodeFromUrl();
  if (code === null) return null;
  setAffiliateCookie(code);
  setAffiliateLocalStorage(code);
  if (posthog.__loaded) posthog.register({ affiliate_code: code });
  return code;
}
```

I substitute `<brand-prefix>` and `<user-domain>` with the user's actual values at write time.

The function is already wired into Stage 6's `main.tsx` `loaded` callback.

### Step 2: I write the marketing-site checkout-CTA layer

Affiliate code from the cookie must flow to the membership checkout URL. I create `src/_core/affiliate-link.ts`:

```ts
export type PlanKey = "plus_monthly" | "plus_annual" | "pro_monthly" | "pro_annual" | "lifetime";

// I populate this map from the user's Whop plan IDs (captured via Whop API
// in Stage 9, or pasted by the user during this stage).
const PLAN_IDS: Record<PlanKey, string> = {
  plus_monthly: "plan_xxxxx",
  plus_annual:  "plan_xxxxx",
  pro_monthly:  "plan_xxxxx",
  pro_annual:   "plan_xxxxx",
  lifetime:     "plan_xxxxx",
};

const COOKIE_NAME = "<brand-prefix>_affiliate_code";

function readAffiliateCookie(): string | null {
  if (typeof document === "undefined") return null;
  const m = document.cookie.match(new RegExp(`(?:^|; )${COOKIE_NAME}=([^;]+)`));
  return m ? decodeURIComponent(m[1]) : null;
}

export function buildCheckoutUrl(planKey: PlanKey, options?: { linkType?: "checkout" | "store" }): string {
  const planId = PLAN_IDS[planKey];
  const affiliateCode = readAffiliateCookie();
  const linkType = options?.linkType ?? "checkout";

  const base = linkType === "store"
    ? `https://whop.com/<your-company-slug>/<product-slug>/?directPlanId=${planId}`
    : `https://whop.com/checkout/${planId}`;

  return affiliateCode ? `${base}?a=${affiliateCode}` : base;
}
```

I update the user's pricing components to call `buildCheckoutUrl` instead of using hardcoded URLs.

### Step 3: I create the PWA-side attribution table + helper

If the site has an authenticated app, I add the durable attribution layer.

Migration:

```sql
CREATE TABLE affiliate_attributions (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  affiliate_code VARCHAR(64) NOT NULL,
  captured_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  captured_via ENUM('url_click','manual_assign','whop_webhook') NOT NULL,
  session_id VARCHAR(64),
  utm_source VARCHAR(64),
  utm_medium VARCHAR(64),
  utm_campaign VARCHAR(128),
  utm_content VARCHAR(128),
  utm_placement VARCHAR(128),
  referrer TEXT,
  INDEX (user_id),
  INDEX (affiliate_code)
);
```

This is a pure event-fact table — immutable after insert. Aggregates derive from Whop dashboard + PostHog cross-tab queries on demand.

Helper module at `server/affiliate-attribution.js`:

```js
import { parse } from "cookie";

const COOKIE_NAME = "<brand-prefix>_affiliate_code";

export function getAffiliateCodeFromRequest(req) {
  const cookieHeader = req.headers.cookie ?? "";
  const cookies = parse(cookieHeader);
  return cookies[COOKIE_NAME] ?? null;
}

export async function recordAttributionForNewUser(req, userId, db) {
  // Best-effort write; never blocks signup.
  try {
    const code = getAffiliateCodeFromRequest(req);
    if (!code) return;
    const url = new URL(req.url);
    await db.insert("affiliate_attributions", {
      user_id: userId,
      affiliate_code: code,
      captured_via: "url_click",
      utm_source: url.searchParams.get("utm_source"),
      utm_medium: url.searchParams.get("utm_medium"),
      utm_campaign: url.searchParams.get("utm_campaign"),
      utm_content: url.searchParams.get("utm_content"),
      utm_placement: url.searchParams.get("utm_placement"),
      referrer: req.headers.referer ?? null,
    });
  } catch (err) {
    console.warn("[affiliate] recordAttributionForNewUser error:", err);
  }
}

export async function getAffiliateCodeForUser(userId, db) {
  try {
    const rows = await db.query(
      "SELECT affiliate_code FROM affiliate_attributions WHERE user_id = ? ORDER BY captured_at DESC LIMIT 1",
      [userId],
    );
    return rows[0]?.affiliate_code ?? null;
  } catch {
    return null;
  }
}
```

I wire `recordAttributionForNewUser` into the user's signup handler (best-effort write; never blocks signup).

### Step 4: I extend Stage 9's webhook handler

I update `handlePaymentSucceeded` (and the other Whop event handlers) to look up the affiliate code:

```js
import { getAffiliateCodeForUser } from "./affiliate-attribution.js";

async function handlePaymentSucceeded(payload, db, capturePostHog) {
  // ... existing logic
  const userId = parseInt(distinctId);
  const affiliateCode = userId ? await getAffiliateCodeForUser(userId, db) : null;

  await capturePostHog({
    event: eventName,
    distinctId,
    properties: {
      // ... existing properties
      affiliate_code: affiliateCode,
      is_affiliate_attributed: affiliateCode !== null,
    },
  });
}
```

Same pattern for refund / dispute / cancellation handlers — every subscription event carries the affiliate context for cohort analysis.

### Step 5: URL shortener indirection layer

This is the load-bearing piece protecting affiliate URLs from breaking on platform renames.

#### User chooses a shortener service

Onboarding asked. If deferred, I ask now:

```
Pick a URL shortener:

  (a) Switchy (recommended) — branded short URLs, REST API for destination
      updates without slug changes, OG metadata support, pixel attachment
      for retargeting. ~$10/mo on the relevant plan.
  (b) Rebrandly — similar feature set, more mature
  (c) Short.io — branded short URLs, REST API
  (d) Bitly Enterprise — most mainstream, less DX-friendly
  (e) DIY (Cloudflare Workers / Express middleware) — free, more setup
```

Switchy is the proven path; I default to it unless the user picks otherwise.

#### Switchy setup (Switchy default path)

I read the saved config to determine the DNS path:

- If `hosting.dns_provider === "cloudflare"` AND `hosting.cloudflare_token === "granted"` → I add the CNAME autonomously via the Cloudflare API after the user adds the domain in Switchy. The user only does the Switchy-side steps.
- Otherwise → I generate a paste-ready DNS record for the user to add manually in their DNS provider's dashboard.

**Path A — Cloudflare with token granted (autonomous DNS):**

I tell the user:

```
Switchy setup — about 7 minutes (I'll handle the DNS edit):

  1. https://switchy.io — create account
  2. Add your branded short-URL domain (e.g., mayas.link)
  3. Switchy will show you the CNAME target (typically links.switchy.io
     or similar). Just say "domain added" — I'll fetch the target via
     the Switchy API and write the CNAME to Cloudflare automatically.
  4. Settings → API Tokens → Create — paste the token here, I'll save
     it to .secrets/switchy-api-key.txt
  5. (Optional) Settings → Pixels → add Meta/Google/TikTok pixel IDs you
     want short links to fire (for retargeting clickers)
```

After the user confirms domain-added + pastes the API token, I:

```js
// Read CNAME target from Switchy API
const target = await fetch("https://api.switchy.io/v1/domains", { headers }).then(r => r.json());
// Add the CNAME record on Cloudflare via API token
await fetch(`https://api.cloudflare.com/client/v4/zones/${zoneId}/dns_records`, {
  method: "POST",
  headers: { Authorization: `Bearer ${cfToken}`, "Content-Type": "application/json" },
  body: JSON.stringify({
    type: "CNAME",
    name: "<short-domain>",
    content: target.cname,
    proxied: false,  // DNS-only (gray cloud) — Switchy provisions its own SSL
    ttl: 1,          // auto
  }),
});
```

I confirm the record was created via a follow-up `GET /dns_records` call and report success.

**Path B — non-Cloudflare or no token (paste-ready instructions):**

I tell the user:

```
Switchy setup — about 10 minutes:

  1. https://switchy.io — create account
  2. Add your branded short-URL domain (e.g., mayas.link)
  3. After adding the domain, Switchy will show you a CNAME target
     (typically links.switchy.io). Copy that value.
  4. In your DNS provider (<detected-provider> from onboarding), add:
     - Type:   CNAME
     - Name:   @  (or the apex/root of your short domain)
     - Target: <the value from step 3>
     - Proxy:  OFF (DNS-only) — Switchy provisions its own SSL
     - TTL:    Auto / 5 min
  5. Switchy provisions SSL after DNS propagates (5 min – 1 hour, async)
  6. Settings → API Tokens → Create — paste the token here, I'll save
     it to .secrets/switchy-api-key.txt
  7. (Optional) Settings → Pixels → add Meta/Google/TikTok pixel IDs you
     want short links to fire (for retargeting clickers)
```

When the user pastes the token, I save to `<project>/.secrets/switchy-api-key.txt`.

#### I write the Switchy helper script

I create `scripts/switchy-shortlink.py` (or `.js`/`.ts` if the user prefers):

```python
"""
Switchy short-link helper.

Usage:
  create:   create a new short link
  update:   update destination of existing slug
  offboard: flip status:terminated tag + redirect to sunset
"""
import os
import sys
import argparse
import requests

API_KEY_PATH = ".secrets/switchy-api-key.txt"
API_BASE = "https://api.switchy.io/v1"

def get_api_key():
    if os.path.exists(API_KEY_PATH):
        return open(API_KEY_PATH).read().strip()
    return os.environ.get("SWITCHY_API_KEY")

def auth_headers():
    return {"Api-Authorization": get_api_key(), "Content-Type": "application/json"}

# (full implementation: create/update/offboard subcommands;
# I substitute the user's branded domain + folder IDs + pixel
# objects at write time based on what they've configured.)
```

I write the create/update/offboard subcommands with the user's specific configuration (their branded domain, any folder IDs they've created in Switchy, any pixels they want attached by default).

#### Switchy operations I'll use

Verified working endpoints (per 2026-04 Switchy API):

| Operation | REST endpoint | Use case |
|---|---|---|
| Create link | `POST /links/create` | New affiliate added → create short URL |
| Update destination | `PUT /links/by-domain/<domain>/<slug>` | Affiliate renamed → update destination invisibly |
| Update OG metadata | Same `PUT` with title/description/image fields | Marketing OG changes → push new OG to all affiliate links |
| Tag-based filtering | GraphQL endpoint at `graphql.switchy.io/v1/graphql` | List all `status:active` affiliate links |

I document the user's Switchy folder IDs / pixel objects / default tags in `<project>/.skill-config.json` so future stages and operational scripts find them.

### Step 6: I create the affiliate registry

I create `<project>/marketing/affiliate-registry.json` to track affiliate metadata Whop doesn't expose (audience description, contact info, custom commission rates):

```json
{
  "schema_version": "1",
  "last_updated": "2026-05-08",
  "affiliates": []
}
```

Empty initially. As the user adds affiliates, I populate via a "Add affiliate" command flow:

```
User: "Add affiliate joe@example.com as joeswebsite"

I (Claude):
  1. Call Whop's POST /api/v1/affiliates with the user identifier
  2. Capture the stable Whop affiliate_id (aff_xxxxxxxxxxxxxx)
  3. Generate the canonical UTM-tagged URL with their handle
  4. Create the Switchy short link via the helper script
  5. Append to affiliate-registry.json
  6. Commit to git: "marketing(affiliate): add joeswebsite (aff_...)"
```

Each affiliate entry tracks:
- Whop affiliate_id (immutable — primary key)
- Whop user_id (immutable — defensive secondary)
- Current Whop username (mutable — what's in `?a=` today)
- Username history (audit trail of all renames)
- Commission rate
- Audience description
- Tracking URLs (long + short)
- Status (pending / approved / active / paused / terminated)

### Step 7: I set up the daily sync task

Whop username renames silently break old `?a=` URLs. Without active monitoring, attribution decays. I write `scripts/sync-affiliates.py`:

```python
"""
Daily sync: for each registry entry, GET /api/v1/affiliates/<aff_id> on Whop.
If Whop's user.username has changed since last sync:
  1. Update affiliate-registry.json: append rename to whop_username_history
  2. Update the Switchy destination URL via switchy-shortlink.py
"""
# (I write the full implementation based on the user's registry schema.)
```

I schedule the script via the user's available scheduler. Options in rough order of effort:

- **GitHub Actions cron**: I commit `.github/workflows/affiliate-sync.yml` with a daily cron trigger that runs the Python script; works for any repo
- **Railway cron jobs**: native Railway feature; runs the script on a separate service on the cron schedule
- **A local scheduled-tasks MCP** (if available in the user's Claude Code environment): runs the script on the user's machine on a cron expression — only works if the user's machine is on
- **Manual run**: I document the command (`python scripts/sync-affiliates.py`) and the user runs it weekly or whenever they recall an affiliate may have renamed

I pick the option that matches what the user has set up. For most projects, GitHub Actions cron is the right default — it's free, doesn't require the user's machine to be running, and version-controls the schedule alongside the code.

### Step 8: I update the privacy + CHD policy drafts

Affiliate tracking attaches an `affiliate_code` property to user events. This is NOT user PII (it belongs to the affiliate). No additional masking needed.

For the Stage 8 privacy policy: I add a sentence noting that affiliate-attribution data is collected when users arrive via affiliate links. For Consumer Health Data policy (if applicable): same note in plain language.

## Verification (autonomous)

I run these myself before declaring Stage 10 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Affiliate capture works | `preview_click` to `/?a=test`, then `preview_eval` to read cookie + localStorage + PostHog super-property | All three contain "test" |
| PostHog super-property attached | MCP query for events in session | `affiliate_code: "test"` on `$pageview` |
| DB attribution row exists after signup | I sign up via the dev path, query DB | Row in `affiliate_attributions` |
| Webhook event has affiliate context | Send Whop test event for that user, check PostHog | `affiliate_code` + `is_affiliate_attributed: true` on the event |
| Switchy short URL resolves | Create a test link, curl it | 302 redirect to long URL with `?a=` preserved |
| OG metadata fetched + baked | curl the short URL HTML response | OG tags match marketing site's OG |
| Daily sync scheduled | Verify the scheduled task is registered | Task ID `affiliate-sync-daily` exists |

I report:

```
✅ Stage 10 complete:
   • Multi-layer attribution wired (cookie + localStorage + PostHog +
     DB attribution table)
   • Affiliate code joins onto every subscription event server-side
   • Switchy custom domain mayas.link active (DNS-only, SSL provisioned)
   • Switchy helper script at scripts/switchy-shortlink.py
   • Daily sync task scheduled (catches Whop username renames)
   • Affiliate registry initialized at marketing/affiliate-registry.json
   • Privacy policy draft updated to mention affiliate attribution

Ready for Stage 11 (variants + dashboards)?
```

## Common gotchas I handle automatically

- **Cookie domain mismatch**: setting on `domain=.<your-domain>` doesn't apply if the user lands on `app.<your-domain>` first. Marketing site sets cookie → user clicks "Get Pro" → cookie flows to app subdomain → app reads it. I verify the order works on the staged URL.
- **Safari ITP shortens cookies**: `SameSite=Lax` cookies have effectively 7-day lifetimes in Safari ITP for cross-site contexts. localStorage backup is critical — I always include it.
- **Switchy slug typo on creation**: slugs are stable forever; can't rename without losing attribution. I validate slug == affiliate handle before creating.
- **Whop username rename undetected**: daily sync MUST run. Without it, old Switchy short URLs keep redirecting to the long URL with the OLD username, which Whop no longer credits → silent attribution loss. I verify the scheduled task is active before declaring Stage 10 complete.
- **Multiple affiliate codes captured** (user clicks Affiliate A, then Affiliate B before signing up): last-touch wins (cookie overwrites). If the user prefers first-touch, I change `setAffiliateCookie` to skip when already set.

## Outputs

After Stage 10:
- Multi-layer attribution stack live
- Affiliate code joins onto every subscription event
- Branded short URLs with stable indirection (Switchy or alternative)
- Daily sync detects + repairs platform username changes
- Affiliate registry tracks metadata the platform doesn't expose
- OG metadata enables unfurl in messaging apps

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Captures the multi-layer affiliate pattern: marketing-site capture + DB attribution + webhook handler join + Switchy indirection + daily username sync. Critical: Whop username renames silently break old `?a=<old>` URLs → Switchy is essential, not optional. |
| 2026-05-08 | Reframed for non-coder audience: replaced "Run X" / "Edit Y" with first-person Claude voice. Replaced literal author short-domain examples with `mayas.link` placeholder. Replaced literal Switchy folder IDs / pixel object examples with "I substitute the user's actual values at write time" framing. Replaced author-home-directory secret paths with project-local `.secrets/`. User-facing dashboard click paths preserved (Switchy initial setup requires it). |
| 2026-05-08 | Switchy DNS edit now conditional on Cloudflare API token from onboarding. Path A (CF + token granted): I add the CNAME autonomously via Cloudflare API. Path B (non-CF or no token): paste-ready DNS-record instructions for the user's DNS provider. Saves ~5 minutes of clicking through Cloudflare's UI for users who granted the token. |
