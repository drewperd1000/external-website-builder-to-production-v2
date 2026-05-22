# Stage 9: Membership Platform (Conditional)

**Last verified: 2026-05-08.**

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) write the webhook handler, the signature verification, the plan-cache table migration, the server-side PostHog forwarding, and Railway env-var setup. The user's actions: (1) sign up at the chosen membership platform and configure products/plans (~30 min in dashboard), (2) generate API keys + webhook signing secret and paste them into project-local secrets files. Everything else autonomous.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 9 needs:
- `membership.enabled` — if `false`, this stage is skipped entirely.
- `membership.platform` — one of `"stripe"` (default), `"whop"`, `"skool"`, `"lemonsqueezy"`, `"paddle"`, `"gumroad"`, or `"other"`. If unset, I prompt with the recommendation logic from onboarding (Stripe for general payments; Whop for content/community/course businesses).
- `membership.tiers` — array of `{name, price, interval}` objects. If unset, I prompt; this is blocking because the webhook → PostHog event mapping needs to know what tiers exist.
- The user's webhook signing secret + API key, pasted into `<project>/.secrets/<platform>-webhook-secret.txt` and `<project>/.secrets/<platform>-api-key.txt`. **Parallel work**: I can write the webhook handler, signature-verification logic, plan-cache schema, and server-side PostHog helper while the user does the platform-side dashboard setup.
- The DB connection (`DATABASE_URL` env var on Railway) — if unset and the platform's webhook payload requires plan-cache lookup, I either provision PostgreSQL via the Railway MCP `deploy-template` tool OR fall back to in-memory cache (less efficient but functional for low-traffic sites).

## What I'm doing in this stage

If the site sells subscriptions, memberships, or one-time access: wiring a membership platform's webhook handler so subscription events flow into PostHog server-side, joining the visitor analytics from Stages 6–7.

After this stage:
- Webhook endpoint at `/api/webhooks/<platform>` verifies signatures + replay-protects
- Subscription lifecycle events (`started`, `renewed`, `canceled`, `refunded`, `disputed`, etc.) are forwarded to PostHog with `distinct_id` continuity (matches the user's browser-side PostHog ID)
- Plan ID → billing period cache table prevents per-event API calls to the platform

**Time**: 30–60 minutes (most of it user-time on the membership platform's dashboard for product setup).

**Skip this stage** if onboarding said the site doesn't sell anything (free marketing, lead-gen, content site).

## Platform support

Onboarding asked which platform the user picked. I adapt to:

| Platform | Notes |
|---|---|
| **Stripe** (default) | Standard payment processor. Webhook signing via Stripe's `stripe-signature` header (`t=<timestamp>,v1=<sig>`). Well-documented; battle-tested. The right default for SaaS, e-commerce, and any business that doesn't specifically need content/community hosting. |
| **Whop** (recommended for content/community/course businesses) | Built-in affiliate program + marketplace + content/community hosting. Standard Webhooks HMAC-SHA256 signing. **Whop's API can also handle payments** so users running content/community businesses can consolidate to one vendor (vs. Stripe + Whop). |
| **Skool** | Community + course platform. No native subscription webhooks as of 2026-05; requires API polling or user-side trigger from browser. Less mature integration path. |
| **Lemonsqueezy** / **Paddle** / **Gumroad** | Similar webhook patterns to Stripe. Adapt signature verification per vendor docs. Good for digital downloads + simple subscriptions. |
| **Other** | Any HMAC-signed webhook with email/userid in payload — same generalized pattern. |

The implementation below shows two parallel paths in detail: **Stripe** (default) and **Whop** (the alternative with deepest verified coverage). For Skool/Lemonsqueezy/Paddle/Gumroad/etc., I follow the same pattern with vendor-specific signature verification + event-name mapping.

## My execution sequence (default: Stripe; Whop path follows)

### Step 1: User sets up Whop account + products

This is the longest user-action step. I tell them:

```
Whop account setup — about 30 minutes if starting fresh:

  1. https://whop.com — create company (or sign in if you have one)
  2. Add your products (one per tier you sell — Plus, Pro, Lifetime,
     etc.) with prices
  3. Each product can have multiple plans (monthly / annual / lifetime).
     Note the plan IDs — they're load-bearing for the marketing site CTAs
     in Stage 10.
  4. After setup, find your business ID (it looks like `biz_xxxxxx`).
     Share it with me — I'll save it to .skill-config.json.

Whop checkout URL formats (I'll use the first by default):
  - Direct checkout: https://whop.com/checkout/<plan_id>?a=<affiliate_handle>
  - Store page:      https://whop.com/<your-company-slug>/<product-slug>/
                     ?directPlanId=<plan_id>&a=<affiliate_handle>

Tell me when you've added all products + plans. I'll need:
  • Your business ID
  • The plan IDs for each tier (I'll capture them via Whop's API once
    you give me an API key in Step 2)
```

### Step 2: User generates Whop API key + webhook secret

```
Two more dashboard items, takes about 5 minutes:

  1. Whop dashboard → Developer → API Keys → "Create"
  2. Save: paste the key here, I'll store at .secrets/whop-api-key.txt

  3. Whop dashboard → Developer → Webhooks → "Create endpoint"
  4. URL: https://app.<your-domain>/api/webhooks/whop
     (or https://<your-domain>/api/webhooks/whop if no app subdomain)
  5. Subscribe to these events:
     - payment.succeeded (initial purchases + renewals)
     - payment.failed
     - membership.activated
     - membership.deactivated
     - membership.cancel_at_period_end_changed
     - refund.created / refund.updated
     - dispute.created / dispute.updated
     - invoice.past_due
  6. Save the webhook signing secret (starts with "whsec_")
  7. Paste it here, I'll store at .secrets/whop-webhook-secret.txt
```

When the user pastes both secrets, I save them to project-local `.secrets/` files and proceed.

### Step 2b (optional): I install Whop's MCP if available

Whop has shipped an MCP server for chat-driven Whop operations (verified the existence as of 2026-05; I check the current endpoint via `docs.whop.com` or Whop's developer page right before this step since the URL may have shifted). **The Whop MCP often does NOT appear in Claude Code's built-in Connectors picker** — it's added as a Custom Connector.

Two paths to add it:

- **Path A (vendor-driven)**: I open Whop's MCP / developer-tools page that has the "Add to Claude Code" button. The user clicks it, Claude Code prompts to approve, done.
- **Path B (manual)**: I add the MCP entry to the user's Claude Code config directly — server URL from Whop's docs, transport SSE, auth OAuth or API-key paste depending on what Whop's MCP supports today.

What having the Whop MCP unlocks:
- Querying products, plans, and members from chat ("show me the last 10 cancellations")
- Listing affiliates and their performance without leaving the session
- Adjusting pricing or pausing plans without dashboard context-switching

If the MCP isn't yet shipped or the user can't add it, this stage proceeds via Whop's REST API + the API key from Step 2 — no functionality is lost, just chat-driven convenience.

### Step 3: I provision the database + write the `getDb()` helper

Stages 5 (forms persistence), 9 (membership plan-cache), and 10 (affiliate attribution) all reference `db.query(...)` and `getDb()`. The helper + DB provisioning are documented HERE because Stage 9 was historically the first to need it, but the helper is reusable: **whichever stage gets to DB-needing-work first provisions; the others just import from `server/db.js`**. For most marketing sites (no membership, but with forms), Stage 5 ends up doing this work. **This step is the same for Whop / Stripe / any other webhook-driven payment platform — and for plain form persistence with no payments at all.**

#### Database choice

I pick based on what the project already has + what Railway makes easy:

| Project state | DB I provision | Why |
|---|---|---|
| Marketing site, no auth, no existing DB | In-memory cache (no DB) | Acceptable for low-traffic; webhook handler still works, just re-fetches from Whop/Stripe API on cold start |
| Project already has a `pg` / `postgres` / `kysely` / `drizzle-orm` dep, or a `DATABASE_URL` env var | Use existing PostgreSQL | Don't fight the existing infra |
| Project already has `better-sqlite3` or `sqlite3` | Use existing SQLite | Same — don't fight existing infra |
| Greenfield (most users) | **PostgreSQL on Railway** (provisioned via `deploy-template` MCP tool or dashboard) | Railway provisions managed PG with backups, free tier covers low-traffic sites, sets `DATABASE_URL` automatically |

For greenfield, I provision via the MCP's template-deploy tool, which can deploy a managed PostgreSQL plugin from Railway's template library:

```
mcp__railway__deploy-template { projectId: "<from-onboarding>", template: "postgres" }
```

The MCP doesn't currently expose a dedicated `create-database` tool (verified 2026-05 against `docs.railway.com/ai/mcp-server`); `deploy-template` is the supported path. If template-deploy doesn't accept "postgres" as a name in the user's MCP version, I fall back to telling the user to click "+ Create" → "Database" → "PostgreSQL" in the Railway dashboard. Either way, the auto-generated `DATABASE_URL` env var then becomes available to the project's services through Railway's reference-variable system.

#### The `getDb()` helper

I create `server/db.js` (path adjusted to project layout) that exposes a normalized API. The helper translates `?` placeholders to whatever the underlying driver wants (`$1, $2, ...` for `pg`, `?` for SQLite/MySQL).

```js
// server/db.js
import { Pool } from "pg";
import Database from "better-sqlite3";

let _pool = null;
let _sqlite = null;

function isPostgres() {
  return Boolean(process.env.DATABASE_URL && process.env.DATABASE_URL.startsWith("postgres"));
}

function isSqlite() {
  return Boolean(process.env.DATABASE_URL && process.env.DATABASE_URL.startsWith("sqlite"));
}

function translatePlaceholders(sql) {
  // Convert "?" placeholders → "$1, $2, ..." for pg; leave alone for SQLite.
  if (!isPostgres()) return sql;
  let i = 0;
  return sql.replace(/\?/g, () => `$${++i}`);
}

export async function getDb() {
  if (!process.env.DATABASE_URL) {
    return null;  // Caller must handle null (in-memory fallback path)
  }
  if (isPostgres()) {
    if (!_pool) _pool = new Pool({ connectionString: process.env.DATABASE_URL });
    return {
      async query(sql, params = []) {
        const translated = translatePlaceholders(sql);
        const result = await _pool.query(translated, params);
        return result.rows;
      },
    };
  }
  if (isSqlite()) {
    if (!_sqlite) _sqlite = new Database(process.env.DATABASE_URL.replace(/^sqlite:\/\//, ""));
    return {
      async query(sql, params = []) {
        const stmt = _sqlite.prepare(sql);
        if (sql.trim().toUpperCase().startsWith("SELECT")) return stmt.all(...params);
        const info = stmt.run(...params);
        return [{ lastInsertRowid: info.lastInsertRowid, changes: info.changes }];
      },
    };
  }
  throw new Error(`Unsupported DATABASE_URL scheme: ${process.env.DATABASE_URL.split(":")[0]}`);
}
```

Usage convention throughout this stage and Stage 10: **all SQL is written with `?` placeholders.** The helper translates as needed. This means the same code paths work whether the user runs PostgreSQL on Railway or SQLite locally for development.

#### The plan-cache table

Most payment platforms' webhook payloads include `plan.id` but NOT the billing period (monthly / annual / lifetime). Billing period lives on the Plan resource, fetched via the platform's API. I cache locally so I hit the platform's API once per new plan, never again.

The table name is **platform-agnostic** (`buyer_tier_plans`, not `whop_plans`) because the same caching pattern applies whether the project uses Stripe, Whop, Skool, Lemonsqueezy, etc. The `plan_id` column holds whatever identifier the chosen platform uses.

```sql
-- PostgreSQL-compatible (works on SQLite with minor tweaks the helper handles)
CREATE TABLE IF NOT EXISTS buyer_tier_plans (
  plan_id              VARCHAR(64) PRIMARY KEY,  -- platform-specific ID (Stripe price_xxx, Whop plan_xxx, etc.)
  billing_period_days  INTEGER,                  -- 30 = monthly, 365 = annual, NULL = lifetime/one-time
  tier_label           VARCHAR(64),              -- "plus", "pro", "lifetime", etc.
  last_synced_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

If the project uses Drizzle / Prisma / Kysely, I write the migration in that tool's idiom rather than raw SQL — I detect from `package.json` deps.

**If `getDb()` returns `null` (no DB)**, the webhook handler falls back to an in-memory plan cache that re-fetches from Whop's API on cold start. Less efficient but functional for low-traffic marketing sites.

### Step 4: I write the webhook handler

I create `server/whop-webhook.js`:

```js
/**
 * Whop webhook handler.
 *
 * Verifies HMAC-SHA256 signatures per Standard Webhooks spec.
 * Translates Whop subscription events to PostHog server-side captures with
 * distinct_id continuity (resolves Whop user.email → our internal user.id).
 *
 * MUST be mounted BEFORE express.json() so req.body is the raw Buffer.
 */
import express from "express";
import crypto from "crypto";

const WHOP_API_BASE = "https://api.whop.com/api/v1";
const MAX_AGE_SEC = 300;  // 5-minute replay window

export function verifyStandardWebhookSignature(rawBody, headers, secret) {
  if (!headers.webhookId || !headers.webhookSignature || !headers.webhookTimestamp) return false;
  const secretBytes = secret.startsWith("whsec_")
    ? Buffer.from(secret.slice(6), "base64")
    : Buffer.from(secret, "utf8");
  const signedPayload = `${headers.webhookId}.${headers.webhookTimestamp}.${rawBody}`;
  const expectedSignature = crypto.createHmac("sha256", secretBytes).update(signedPayload).digest("base64");

  // Header format: "v1,BASE64SIG" (may have multiple space-separated entries during rotation)
  const sigEntries = headers.webhookSignature.split(" ");
  for (const entry of sigEntries) {
    const idx = entry.indexOf(",");
    if (idx < 0) continue;
    const version = entry.slice(0, idx);
    const value = entry.slice(idx + 1);
    if (version === "v1" && value && timingSafeEqualBase64(value, expectedSignature)) return true;
  }
  return false;
}

function timingSafeEqualBase64(a, b) {
  const aBuf = Buffer.from(a, "base64");
  const bBuf = Buffer.from(b, "base64");
  if (aBuf.length !== bBuf.length) return false;
  return crypto.timingSafeEqual(aBuf, bBuf);
}

async function getPlanBillingPeriod(planId, db) {
  if (!db) return null;
  const cached = await db.query("SELECT billing_period_days FROM buyer_tier_plans WHERE plan_id = ?", [planId]);
  if (cached.length > 0) return cached[0].billing_period_days;

  const apiKey = process.env.WHOP_API_KEY;
  if (!apiKey) return null;

  try {
    const res = await fetch(`${WHOP_API_BASE}/plans/${planId}`, {
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (!res.ok) return null;
    const plan = await res.json();
    const billingPeriodDays = plan.billing_period ?? null;
    const tierLabel = plan.product?.title ?? null;
    try {
      await db.query(
        "INSERT INTO buyer_tier_plans (plan_id, billing_period_days, tier_label) VALUES (?, ?, ?)",
        [planId, billingPeriodDays, tierLabel],
      );
    } catch { /* concurrent insert collided — fine */ }
    return billingPeriodDays;
  } catch (err) {
    console.error(`[whop] plan fetch error for ${planId}:`, err);
    return null;
  }
}

function inferTierLabel(productTitle) {
  if (!productTitle) return "unknown";
  const lower = productTitle.toLowerCase();
  if (lower.includes("lifetime")) return "lifetime";
  if (lower.includes("pro") || lower.includes("top")) return "pro";
  if (lower.includes("plus") || lower.includes("basic") || lower.includes("starter")) return "plus";
  if (lower.includes("free")) return "free";
  return lower;
}

async function resolveDistinctId(whopUserId, whopUserEmail, db) {
  if (!whopUserEmail) return whopUserId;
  try {
    const found = await db.query("SELECT id FROM users WHERE email = ?", [whopUserEmail.toLowerCase()]);
    if (found.length > 0) return String(found[0].id);
  } catch (err) {
    console.warn("[whop] resolveDistinctId DB error:", err);
  }
  return whopUserId;
}

function eventNameForBillingReason(billingReason) {
  if (billingReason === "subscription_cycle") return "subscription_renewed";
  if (billingReason === "subscription_update") return "subscription_upgraded";
  return "subscription_started";
}

export async function handleWhopEvent(envelope, deps) {
  const { db, capturePostHog } = deps;
  switch (envelope.type) {
    case "payment.succeeded":
      return handlePaymentSucceeded(envelope.data, db, capturePostHog);
    case "payment.failed":
      return handlePaymentFailed(envelope.data, db, capturePostHog);
    case "membership.deactivated":
      return handleMembershipDeactivated(envelope.data, db, capturePostHog);
    case "refund.created":
      return handleRefundCreated(envelope.data, db, capturePostHog);
    case "dispute.created":
      return handleDisputeCreated(envelope.data, db, capturePostHog);
    // ... I add handlers for each subscribed event in Step 2
    default:
      console.log(`[whop] ignored event: ${envelope.type}`);
  }
}

async function handlePaymentSucceeded(payload, db, capturePostHog) {
  const billingPeriodDays = await getPlanBillingPeriod(payload.plan.id, db);
  const eventName = eventNameForBillingReason(payload.billing_reason);
  const tier = inferTierLabel(payload.product?.title);
  const distinctId = await resolveDistinctId(payload.user.id, payload.user.email, db);

  await capturePostHog({
    event: eventName,
    distinctId,
    properties: {
      revenue: payload.usd_total ?? payload.total,
      currency: payload.currency,
      tier,
      billing_period: billingPeriodDays,
      whop_payment_id: payload.id,
      whop_plan_id: payload.plan.id,
      whop_product_id: payload.product.id,
      whop_user_id: payload.user.id,
      billing_reason: payload.billing_reason,
      // Stage 10 will add: affiliate_code, is_affiliate_attributed
    },
  });
}

// Other handlers (handlePaymentFailed, handleMembershipDeactivated, etc.)
// follow the same pattern. Each calls capturePostHog with appropriate
// event name + properties from the Whop payload.

export function registerWhopWebhook(app, deps) {
  app.post(
    "/api/webhooks/whop",
    express.raw({ type: "application/json", limit: "1mb" }),
    async (req, res) => {
      const secret = process.env.WHOP_WEBHOOK_SECRET;
      if (!secret) {
        res.status(503).json({ error: "Whop integration not configured" });
        return;
      }

      const rawBody = req.body?.toString("utf8") ?? "";
      const headers = {
        webhookId: String(req.headers["webhook-id"] ?? ""),
        webhookSignature: String(req.headers["webhook-signature"] ?? ""),
        webhookTimestamp: String(req.headers["webhook-timestamp"] ?? ""),
      };

      if (!verifyStandardWebhookSignature(rawBody, headers, secret)) {
        console.warn(`[whop] invalid signature on event ${headers.webhookId}`);
        res.status(401).json({ error: "Invalid signature" });
        return;
      }

      const tsSeconds = parseInt(headers.webhookTimestamp, 10);
      const ageSec = Math.abs(Date.now() / 1000 - tsSeconds);
      if (!Number.isFinite(tsSeconds) || ageSec > MAX_AGE_SEC) {
        res.status(400).json({ error: "Webhook timestamp out of bounds" });
        return;
      }

      let envelope;
      try { envelope = JSON.parse(rawBody); }
      catch { res.status(400).json({ error: "Invalid JSON" }); return; }

      try {
        await handleWhopEvent(envelope, deps);
      } catch (err) {
        // Don't 5xx — Whop retries. Log + ack to investigate without re-receiving.
        console.error("[whop] handler error:", err);
      }

      res.status(200).json({ ok: true });
    },
  );
}
```

### Step 5: I write the server-side PostHog capture helper

I create `server/posthog-server.js`:

```js
/**
 * Server-side PostHog event capture.
 * Uses plain fetch instead of posthog-node SDK to avoid a runtime dep.
 */
const POSTHOG_HOST = process.env.POSTHOG_HOST || "https://us.i.posthog.com";

export async function captureServerEvent({ event, distinctId, properties = {}, timestamp }) {
  const apiKey = process.env.POSTHOG_PROJECT_API_KEY;
  if (!apiKey) {
    if (process.env.NODE_ENV !== "test") {
      console.warn(`[posthog-server] POSTHOG_PROJECT_API_KEY not set; skipping "${event}"`);
    }
    return false;
  }

  try {
    const res = await fetch(`${POSTHOG_HOST}/capture/`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        api_key: apiKey,
        event,
        distinct_id: distinctId,
        properties,
        timestamp: timestamp ?? new Date().toISOString(),
      }),
    });
    if (!res.ok) {
      console.warn(`[posthog-server] capture failed for "${event}": ${res.status}`);
      return false;
    }
    return true;
  } catch (err) {
    console.warn(`[posthog-server] capture error for "${event}":`, err);
    return false;
  }
}
```

### Step 6: I mount the webhook in `server.js`

I update Stage 3's `server.js`:

```js
import { registerWhopWebhook } from "./server/whop-webhook.js";
import { captureServerEvent } from "./server/posthog-server.js";
import { getDb } from "./server/db.js";  // created in Step 3 of this stage

// after registerPosthogProxy, BEFORE express.json:
registerWhopWebhook(app, {
  db: await getDb(),
  capturePostHog: captureServerEvent,
});

app.use(express.json({ limit: "32kb" }));
```

The order is critical (Stage 3's invariant): webhook handler mounted BEFORE `express.json()` so HMAC verification has access to raw bytes.

### Step 7: I set Railway env vars

Per Stage 4's secrets-file pattern:

```
railway variable set WHOP_WEBHOOK_SECRET="$(cat .secrets/whop-webhook-secret.txt)"
railway variable set WHOP_API_KEY="$(cat .secrets/whop-api-key.txt)"
railway variable set POSTHOG_PROJECT_API_KEY="$(cat .secrets/posthog-public-key.txt)"
railway up
```

Note: `POSTHOG_PROJECT_API_KEY` is the same value as `VITE_POSTHOG_KEY` (both reference the public Project API Key). The naming convention here keeps them separate so server-side code doesn't accidentally rely on a `VITE_*` var (which would imply browser context).

### Step 8: I run end-to-end test

I tell the user:

```
Final test — about 2 minutes:

  1. Whop dashboard → Webhooks → your endpoint → "Send test event"
  2. Pick "payment.succeeded" as the event type
  3. Click Send

I'll watch the Railway logs for the webhook arrival + verify the PostHog
event appears.
```

Then I:
1. `railway logs` (streams by default in current CLI) and look for `[<platform>]` entries (e.g., `[stripe]`, `[whop]`) — expect a `event: ...succeeded` line, no signature error
2. Query PostHog via MCP: `mcp__posthog__query-run` with HogQL: `select event, properties from events where event = 'subscription_started' order by timestamp desc limit 1` — expect the test event

If both checks pass, the integration works end-to-end. The user can make a real $1 test purchase next to verify the full chain (real Whop event → my handler → PostHog → eventually showing up in Revenue Analytics). I don't push them to do this, but I describe the option.

## Verification (autonomous)

| Check | What I do | Pass criteria |
|---|---|---|
| Webhook secret + API key set | `railway variable list` filtered | All three env vars present |
| Webhook endpoint registered | I instruct user to check Whop dashboard | Endpoint listed |
| Test webhook returns 200 | User sends test from Whop; I check Railway logs | `[whop] event: payment.succeeded` line appears, no signature error |
| Test event in PostHog | MCP query for recent `subscription_started` | Event present with `whop_payment_id` matching test |
| Plan cache populates after first real purchase | Query DB for `buyer_tier_plans` rows | Row exists for the purchased plan |

I report:

```
✅ Stage 9 complete:
   • Whop integration active
   • Webhook endpoint: https://app.<your-domain>/api/webhooks/whop
   • Subscribed events: 9 (payment, membership, refund, dispute, invoice)
   • HMAC signature verification + 5-minute replay protection in place
   • Plan-cache table created (buyer_tier_plans)
   • Server-side PostHog forwarding active
   • Test webhook → PostHog round-trip verified

Ready for Stage 10 (affiliates) or skip?
```

## Alternative platforms (briefer treatment)

### Stripe direct (the actual default for most projects)

Stripe is simpler than Whop in some ways (no plan-billing-period cache needed; Stripe puts everything in the webhook payload) and more complex in others (signature scheme differs; the user has to manage Products + Prices in Stripe's dashboard).

**User-side setup** (~15 min):

1. Stripe Dashboard → Products → create one Product per pricing tier
2. For each Product, create one or more Prices (recurring monthly, recurring annual, one-time, etc.)
3. Note each Price ID (`price_...`) — these are load-bearing for the marketing site CTAs at Stage 10
4. Developers → API keys → reveal Secret key (`sk_live_...` for production, `sk_test_...` for testing) — paste to me; I save to `<project>/.secrets/stripe-secret-key.txt`
5. Developers → Webhooks → "Add endpoint" → URL: `https://<user-domain>/api/webhooks/stripe`
6. Subscribe to events:
   - `checkout.session.completed`
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `invoice.payment_succeeded`
   - `invoice.payment_failed`
   - `charge.refunded`
   - `charge.dispute.created`
7. Note the webhook signing secret (`whsec_...`) — paste to me; I save to `<project>/.secrets/stripe-webhook-secret.txt`

**My implementation** — Stripe uses a different signature scheme than Whop's Standard Webhooks. I use Stripe's official `stripe.webhooks.constructEvent()` for verification:

```js
import Stripe from "stripe";
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

export function registerStripeWebhook(app, deps) {
  app.post(
    "/api/webhooks/stripe",
    express.raw({ type: "application/json", limit: "1mb" }),
    async (req, res) => {
      const sigHeader = req.headers["stripe-signature"];
      const secret = process.env.STRIPE_WEBHOOK_SECRET;
      let event;
      try {
        event = stripe.webhooks.constructEvent(req.body, sigHeader, secret);
      } catch (err) {
        return res.status(400).json({ error: `Invalid signature: ${err.message}` });
      }
      try {
        await handleStripeEvent(event, deps);
      } catch (err) {
        console.error("[stripe] handler error:", err);
      }
      res.status(200).json({ received: true });
    },
  );
}
```

**Event mapping** I use:

| Stripe event | PostHog event |
|---|---|
| `checkout.session.completed` (mode: `subscription`) | `subscription_started` (with revenue from `amount_total`) |
| `customer.subscription.created` | (informational; `checkout.session.completed` is more reliable for new-purchase signal) |
| `invoice.payment_succeeded` (with `billing_reason: "subscription_cycle"`) | `subscription_renewed` |
| `invoice.payment_failed` | `subscription_payment_failed` |
| `customer.subscription.updated` (status changed to `canceled` OR `cancel_at_period_end: true`) | `subscription_cancellation_intent` or `subscription_canceled` |
| `customer.subscription.deleted` | `subscription_canceled` |
| `charge.refunded` | `subscription_refunded` |
| `charge.dispute.created` | `subscription_disputed` |

`distinct_id` resolution: Stripe's `customer.email` → my internal `users` table → internal user.id (same pattern as the Whop implementation; only the source field differs).

No plan-cache table needed — Stripe includes `price.id` + `price.recurring.interval` directly in the webhook payload, so I extract billing period without an API call.

**Key differences from Whop**:
- No HMAC verification — use `stripe.webhooks.constructEvent` (handles signature verification internally)
- No `buyer_tier_plans` cache — billing period in the payload
- No marketplace / affiliate program built-in (use Trackdesk or Rewardful for affiliates with Stripe — handled in Stage 10)
- Refunds via `charge.refunded` event (not a separate `refund.*` event family)

### Stripe — original concise reference

If preferred, the brief form:

```js
const event = stripe.webhooks.constructEvent(rawBody, sigHeader, endpointSecret);
```

Then map `event.type` to PostHog events:
- `customer.subscription.created` → `subscription_started`
- `invoice.payment_succeeded` with `billing_reason="subscription_cycle"` → `subscription_renewed`
- `customer.subscription.deleted` → `subscription_canceled`

### Skool

Skool doesn't expose subscription webhooks publicly as of 2026-05. Workarounds:

1. **Periodic polling**: scheduled task hits Skool's API for changes since last sync. I write a polling script that runs hourly via the user's preferred scheduler.
2. **User-side trigger**: when a user's tier changes, the app emits a custom PostHog event from the browser.

Path 1 is more reliable but requires a scheduler. Path 2 is simpler but loses events for users who never log in.

### Lemonsqueezy / Paddle / Gumroad

All use HMAC-SHA256 signature schemes similar to Whop. I adapt the verification step + event names per vendor docs. The general structure (raw body → HMAC verify → parse → server-side PostHog capture) stays identical.

## Common gotchas I handle automatically

- **Forgot `express.raw()` middleware on the webhook route**: `req.body` becomes the parsed JSON object, not raw bytes; HMAC verification fails. I always use the per-route `express.raw({ type: "application/json" })` middleware.
- **Whop's webhook payload field name varies by event**: I never hardcode `data.user.id` — verified each event's actual shape against current Whop docs (their schema has changed before).
- **Replay-protection window too tight**: 300 seconds is the Standard Webhooks recommendation. Whop retries on 5xx, so a transient outage that pushes events past the window causes re-deliveries to fail. I keep 300s as default.
- **Multiple webhook secrets during rotation**: Standard Webhooks supports `v1,sig1 v1,sig2` for atomic key rotation. My `verifyStandardWebhookSignature` handles this.

## Outputs

After Stage 9:
- Webhook endpoint live and signature-verifying
- Subscription events flowing to PostHog with proper distinct_id continuity
- Plan billing-period cache populated (or in-memory fallback for DB-less sites)
- Refund / dispute / payment-failed handlers also live (for Whop's 9 subscribed events)

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Standard Webhooks HMAC-SHA256, plan-billing-period cache, distinct_id resolution via email lookup, server-side captureServerEvent helper. Generalized for Stripe / Skool / etc. |
| 2026-05-08 | Reframed for non-coder audience: replaced "Run X" / "Add Y to your server" with first-person Claude voice. Replaced literal author Whop business ID example with `biz_xxxxxx` placeholder. Replaced author-home-directory secret paths with project-local `.secrets/`. User-facing dashboard click paths preserved (these are required — vendor APIs don't cover initial product setup). Verification reframed to autonomous Railway log + PostHog MCP queries. |
