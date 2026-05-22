# Stage 5: Email Provider + Forms Pipeline (with Reoon deliverability gate)

**Last verified: 2026-05-21.** Re-search `Resend MCP server <year>` and `Reoon Email Verifier API <year>` before this stage if older than 60 days.

> **Upgrading from v1?** See [CHANGELOG.md](CHANGELOG.md). v1 carried the Reoon Email Verifier as a 🚧 PENDING comment block; v2 has the full integration as actual code in Steps 2b and 4b. All other steps work the same as v1.


> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) wire up transactional email AND the full forms pipeline: I generate one `/api/<form-name>` endpoint per form discovered in Stage 0's `forms-manifest.md`, each with the four-layer model (validate → DB capture → notification email → ESP segment sync → CRM sync → PostHog event). The user's actions: (1) sign up at the email provider's dashboard once and paste the API key, (2) add DNS records (or grant Cloudflare API token), (3) review the per-form routing suggestions I generate and confirm the ESP segment / CRM stage mappings. Everything else I do autonomously, including LIVE researching the user's specific ESP and CRM at integration time so the generated code reflects each vendor's current API.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 5 needs:
- `email.provider` — default to `"resend"` if unset; I confirm with the user.
- `email.sending_domain` — must be a domain the user controls. If unset, I prompt; this is blocking because DKIM/SPF setup needs it.
- `email.from_address` — full RFC-5322 form like `"Brand Name <hello@<sending-domain>>"`. If unset, I propose `"<project.brand> <hello@<sending-domain>>"` and confirm.
- `email.notify_to` — where contact-form / signup notifications go (typically the user's own inbox or a team alias). If unset, default to `hello@<sending-domain>` and confirm.
- The user's API key, pasted into `<project>/.secrets/<provider>-api-key.txt` — I prompt for this when I need it. **Parallel work I can do while waiting**: write the transport abstraction module, the email templates, and the form-handler factory plumbing. The actual `railway variable set` of the key is the only step that blocks on the paste.
- **Stage 0's `forms-manifest.md`** must exist at repo root. Stage 0's "Forms — full structural extraction" section produces it. If missing, I run Stage 0's form-discovery sub-step inline before proceeding.
- `forms.esp_provider` and `forms.crm_provider` from onboarding (may be `null`; user can defer the decision to this stage's per-form review step).
- `forms.db_capture_enabled` (default `true`) — drives whether Layer A of the four-layer model fires.

## What I'm doing in this stage

Two coupled jobs:

**Part A — Transactional email**: install the email-provider SDK, write a swappable transport abstraction so future provider migrations are mechanical, verify the sending domain via DKIM/SPF/return-path DNS records.

**Part B — Forms pipeline**: read `forms-manifest.md` from Stage 0 and generate one handler per form using the four-layer model documented in [`_internal/reference-forms-and-persistence.md`](_internal/reference-forms-and-persistence.md):

```
validate → DB capture (default ON) → notification email (default ON if EMAIL_NOTIFY_TO set)
        → ESP segment sync (only if user mapped this form to a segment)
        → CRM sync (only if user mapped this form to a stage)
        → PostHog <form_name>_completed event
```

Per-form routing decisions are confirmed with the user via Steps F1–F6 in the reference doc. The user does NOT re-list fields or re-state form purposes — Stage 0 already discovered those by reading the code. The user only confirms routing (which segment, which CRM stage, opt-out of DB capture if applicable).

**At ESP/CRM integration time, I do live research** — WebFetch the vendor's current API docs, search for `<vendor> MCP server <year>`, search for an official Node SDK — and generate the integration code based on what I find that day. I do NOT use stale, pre-baked recipes. Vendor-specific procedures evolve faster than this skill can; the research-first pattern survives.

**Time**: 15–30 minutes of automated work for Part A + Part B combined. Plus DNS-verification wait time (5 minutes to a few hours, runs in parallel) and any user back-and-forth on per-form routing decisions.

## Default recommendation: Resend

Why I default to Resend:
- Strong deliverability + inbox placement
- Clean REST API, low conceptual surface
- Cheap free tier (3,000 emails/month, 100/day) — plenty for marketing signups
- Well-maintained TypeScript SDK

If the user (per onboarding) chose a different ESP, the structure of this stage stays the same — only Step 4 (the transport implementation) changes. Alternative implementations for AWS SES, Postmark, Mailgun, SendGrid are at the bottom of this file.

## Prerequisites

- Stage 4 complete (Railway service deployed).
- A sending domain the user controls (cannot send from `gmail.com`, `yahoo.com`, etc.).
- An email address on that domain that receives notifications (typically `hello@<user-domain>` or `support@<user-domain>`). If the user hasn't provisioned an inbox yet, I prompt them — Cloudflare Email Routing is the cheapest option (free, forwards to existing Gmail/etc.) for solo founders.

## My execution sequence

### Step 1: Resend account + initial API key

Resend has no MCP server as of 2026-05, and the first API key requires a dashboard click. I tell the user:

```
I need to set up your Resend account. This takes about 3 minutes:

  1. Please open https://resend.com/signup in your browser (I'll open it in a tab via Chrome MCP if it's connected; otherwise visit directly).
     just visit it directly).
  2. Sign up with your email + password (or use Google/GitHub OAuth).
  3. Verify your email when Resend sends the confirmation.
  4. Once you're in the dashboard, click "API Keys" → "Create API Key".
  5. Pick "Full Access" permission and click "Add".
  6. Resend shows the key ONCE. Copy it — it starts with "re_".
  7. Paste it here, and I'll save it to a project-local secrets file
     so it doesn't end up in chat history.
```

When the user pastes the key, I:
1. Create `<project>/.secrets/` directory if it doesn't exist
2. Add `.secrets/` to `.gitignore` if not already there
3. Write the key to `<project>/.secrets/resend-api-key.txt`
4. Confirm the file exists and is gitignored

After this initial bootstrap, Resend's REST API can mint additional sub-keys via `POST /api-keys` programmatically — I do that on the user's behalf if scoped keys per environment are needed later.

### Step 2: I verify the sending domain

This is the longest-pole step in Stage 5. DNS verification can take minutes to hours.

#### Path A: I do everything except DNS-record-paste (default for non_coder)

1. I call Resend's API to register the domain: `POST /domains` with the user's sending domain
2. Resend's response includes a `records` array — DKIM, SPF, optional MX + tracking CNAMEs
3. I tell the user the exact records to add at their DNS provider, with click paths:

```
Resend needs these DNS records added to mayasconsulting.com.

If your DNS is on Cloudflare:
  1. Open Cloudflare → Domains → mayasconsulting.com → DNS → Records
  2. Click "Add record" for each of the 4 records below
  3. (Easy mode: click "Bulk Edit" instead and paste this block)

The records:
  Type:  TXT
  Name:  resend._domainkey
  Value: p=MIGfMA0GCSqGSIb3DQEB... (long DKIM key Resend gave us)
  TTL:   Auto

  Type:  TXT
  Name:  send
  Value: v=spf1 include:amazonses.com ~all
  Note:  ⚠️ See "SPF merge" note below if you already have an SPF record.

  ... (and so on)

Tell me when you've added all of them. I'll click "Verify" in Resend.
```

4. Once the user confirms, I call Resend's `POST /domains/{id}/verify` endpoint
5. I poll status via `GET /domains/{id}` until status flips to `verified` (or timeout after 30 minutes — DNS can be slow)

#### Path B: Cloudflare API automation (faster if user grants me a token)

If the user has a Cloudflare API token (or grants me one for this project), I add the DNS records via Cloudflare's API automatically. Total time: 5 minutes for verification to settle, vs 30+ minutes for the human path.

I ask once during onboarding (or here if not yet asked):

```
Quick option: do you have a Cloudflare API token I can use to add these
DNS records automatically? (Saves you ~5 minutes of clicking.)

  (a) Yes — I'll paste it (will save to .secrets/cloudflare-api-token.txt)
  (b) No, I'll add the records manually
  (c) I don't have a Cloudflare API token; help me create one
```

If (a) or (c), I store the token in `.secrets/cloudflare-api-token.txt` and add records via Cloudflare's `POST /zones/{zone_id}/dns_records` endpoint, then verify via Resend's API.

#### SPF merge gotcha (I always check)

If the user already has an SPF record (common — Google Workspace, Microsoft 365, etc. add their own), they CANNOT have two TXT records starting with `v=spf1`. The records must be merged:

```
# Before (existing record from Google Workspace):
v=spf1 include:_spf.google.com ~all

# After (merged with Amazon SES that Resend uses):
v=spf1 include:_spf.google.com include:amazonses.com ~all
```

Before I tell the user to add the SPF record, I run `dig TXT <user-domain> +short` to check what's already there. If an existing `v=spf1` record exists, I produce the merged version instead of asking the user to add a duplicate.

#### Verification check

Once DNS propagates, I run `dig TXT resend._domainkey.<user-domain> +short` to confirm DKIM is visible. Then I call Resend's verify endpoint and confirm the status flips to `"verified"`.

### Step 2b: Reoon Email Verifier account + initial API key

If `email.reoon_enabled = true` (default for any site with forms), I collect the Reoon API key now in parallel with Step 2's DNS-verification wait.

```
I need to set up your Reoon Email Verifier account. This is the deliverability
gate that blocks fake/disposable/spamtrap emails from reaching your ESP — one
disposable signup can poison your Resend reputation for weeks.

Reoon's daily credits come from AppSumo as a one-time-purchase Lifetime Deal
(not a monthly SaaS). Here's the path:

  1. Please open https://appsumo.com/products/reoon-email-verifier/ in your
     browser. Buy at least Tier 1 ($59 one-time for 500 credits/day) — you
     can stack additional tiers later. Tier 2 ($118 total for 1,200/day) is
     the most common entry point.
  2. AppSumo emails you a redemption code. Click the link, sign in (or
     create the Reoon account), and redeem the code.
  3. Open https://emailverifier.reoon.com/dashboard → API → Create API Key.
  4. Copy the key. Paste it here, and I'll save it to a project-local
     secrets file so it doesn't end up in chat history.

If you're not ready to buy yet, you can skip this step — I'll leave the
verifier code in place but it'll no-op until REOON_API_KEY is set. Signups
just go through ungated until you wire the key.
```

When the user pastes the key, I:
1. Write the key to `<project>/.secrets/reoon-email-verifier-api-key.txt`
2. Confirm the file exists and is gitignored
3. Record the user's tier choice in `email.reoon_tier` (1–5) — this drives `REOON_DAILY_QUOTA`

The tier-to-quota mapping (verified 2026-05-21, double-check at AppSumo if older):

| Tier | Daily credits | Newsletter signups/day (2 credits each) | Contact submits/day (1 credit each) |
|---|---|---|---|
| 1 | 500 | ~250 | ~500 |
| 2 | 1,200 | ~600 | ~1,200 |
| 3 | 2,100 | ~1,050 | ~2,100 |
| 4 | 3,200 | ~1,600 | ~3,200 |
| 5 | 4,500 | ~2,250 | ~4,500 |

If the user buys more tiers later, they update `REOON_DAILY_QUOTA` on Railway (both prod and staging services) and update this skill's saved config.

### Step 3: I install the Resend SDK

```
npm install resend
```

The SDK is ESM + has TypeScript types. No additional config needed. Reoon has no SDK — the integration in Step 4b uses raw `fetch`.

### Step 4: I implement the transport abstraction

I create `server/email.js` with a transport interface. This is the swappable boundary — future SES / Postmark / Mailgun migrations only edit this file.

```js
/**
 * Email service — generic transport interface with Resend default.
 * To swap providers: implement EmailTransport, replace getTransport().
 * The rest of the codebase calls sendEmail() and doesn't know about the
 * underlying provider.
 */
import { Resend } from "resend";

// Fail-fast on missing env vars in production. The "example.com" fallbacks
// that AI-generated boilerplate often ships are silently broken — Resend
// rejects them with 422, but the bundle deploys, the server boots, and
// the first form submit fails in the visitor's face. Throwing at import
// time means the deploy fails LOUDLY, before any visitor is affected.
function requireEnv(name) {
  const v = process.env[name];
  if (v === undefined || v === null || v === "") {
    if (process.env.NODE_ENV === "production") {
      throw new Error(
        `[email] Required env var ${name} is not set. ` +
        `Set it on Railway: railway variable set ${name}="<value>"`
      );
    }
    // In development we tolerate missing vars so dev servers boot —
    // the console transport will be picked up below and emails just log.
    return null;
  }
  return v;
}

export const EMAIL_FROM = requireEnv("EMAIL_FROM");
export const EMAIL_NOTIFY_TO = requireEnv("EMAIL_NOTIFY_TO");
export const SITE_URL = requireEnv("SITE_URL");

// ─── Resend transport ─────────────────────────────────────────────
function createResendTransport() {
  if (!process.env.RESEND_API_KEY) {
    throw new Error("RESEND_API_KEY not set");
  }
  const resend = new Resend(process.env.RESEND_API_KEY);
  return {
    async send(msg) {
      try {
        const { data, error } = await resend.emails.send({
          from: msg.from || EMAIL_FROM,
          to: msg.to,
          replyTo: msg.replyTo,
          subject: msg.subject,
          html: msg.html,
          text: msg.text,
        });
        if (error) {
          console.error("[email/resend] send error:", error);
          return { success: false, error };
        }
        return { success: true, messageId: data?.id };
      } catch (err) {
        console.error("[email/resend] exception:", err);
        return { success: false, error: err };
      }
    },
  };
}

// ─── Console transport (development fallback) ─────────────────────
const consoleTransport = {
  async send(msg) {
    console.log(
      `\n📧 [Email DEV] ─────────────────────────────────\n` +
      `  To:      ${msg.to}\n` +
      `  Subject: ${msg.subject}\n` +
      `  Body:    ${msg.text ?? "(HTML only)"}\n` +
      `─────────────────────────────────────────────────\n`,
    );
    return { success: true, messageId: `dev-${Date.now()}` };
  },
};

// ─── Transport selection ──────────────────────────────────────────
let _transport = null;
function getTransport() {
  if (_transport) return _transport;
  if (process.env.RESEND_API_KEY) {
    _transport = createResendTransport();
  } else {
    if (process.env.NODE_ENV === "production") {
      console.warn("[email] RESEND_API_KEY unset in production — emails will log to console only");
    }
    _transport = consoleTransport;
  }
  return _transport;
}

// ─── Public API ───────────────────────────────────────────────────
export async function sendEmail(msg) {
  const t = getTransport();
  return t.send(msg);
}
```

**Key design points**:
- **Fail-fast on missing addresses, lazy on the transport.** Required addresses (`EMAIL_FROM`, `EMAIL_NOTIFY_TO`, `SITE_URL`) throw at import time in production, so the deploy fails loudly if the user forgot to set them on Railway. The Resend transport itself is lazy: if `RESEND_API_KEY` is unset, the module falls through to the console transport and the server boots — appropriate for staging or dev environments where transactional email isn't load-bearing. The reason for the asymmetry: the addresses ship as build-time invariants the user verified at Stage 1, so missing-at-deploy means broken-everywhere; the API key is more dynamic and might genuinely be unset on a non-prod environment.
- **`EMAIL_FROM` must be on a Resend-verified domain.** Stage 5's verification step confirms this. If the user later changes the sending domain, the deploy will silently send 422s — gotcha section below covers this.

### Step 4b: I write the Reoon email-verifier module

If `email.reoon_enabled = true`, I create `email-verifier.js` at the project root. This module exports `verifyQuick` (blocking gate), `verifyPowerAsync` (fire-and-forget), `userMessageForQuickReject` (user-facing error messages), and `getReoonUsageSnapshot` (admin endpoint data). It has no dependencies beyond Node 20's built-in `fetch`.

```javascript
// email-verifier.js
//
// Reoon Email Verifier integration.
//
// Two modes:
//   verifyQuick(email)        — blocking, ~250ms typical, 5s ceiling.
//                                Returns { ok: boolean, status, reason, raw }.
//                                Rejects: invalid | disposable | spamtrap.
//   verifyPowerAsync(email)   — fire-and-forget, seconds-to-minutes.
//                                Surfaces flags to Slack: catch_all, role_account,
//                                disabled, inbox_full, disposable, spamtrap, unknown.
//
// Fail-open posture: network errors, timeouts, non-200s, parse failures, and
// 429s all return { ok: true } so real signups continue if Reoon is down.
// REOON_API_KEY unset = skip entirely (returns { ok: true, status: "skipped" }).

const REOON_BASE = "https://emailverifier.reoon.com/api/v1/verify";
const QUICK_TIMEOUT_MS = 5000;
const POWER_TIMEOUT_MS = 90000;
const DAILY_QUOTA = parseInt(process.env.REOON_DAILY_QUOTA || "1200", 10);
const SOFT_THRESHOLD = 0.8;

const QUICK_REJECT_STATUSES = new Set(["invalid", "disposable", "spamtrap"]);
const POWER_FLAG_STATUSES = new Set([
  "invalid", "disabled", "disposable", "inbox_full",
  "catch_all", "role_account", "spamtrap", "unknown",
]);

// In-memory usage state. Resets at UTC midnight or container restart.
// Container restarts will under-count, but the goal is rough operational
// awareness, not exact accounting — Reoon's own dashboard is the
// authoritative source for billing.
let usageState = { date: null, count: 0, alertedSoft: false, alertedHard: false };

function todayUtc() {
  return new Date().toISOString().slice(0, 10);
}

function bumpUsage(credits, mode) {
  const today = todayUtc();
  if (usageState.date !== today) {
    usageState = { date: today, count: 0, alertedSoft: false, alertedHard: false };
  }
  usageState.count += credits;
  maybeFireThresholdAlert(mode);
}

async function postToSlack(webhookUrl, payload) {
  if (process.env.NODE_ENV !== "production") return;
  if (!webhookUrl) return;
  try {
    const r = await fetch(webhookUrl, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload),
    });
    if (!r.ok) console.error(`[reoon/slack] returned ${r.status}`);
  } catch (err) {
    console.error("[reoon/slack] post failed:", err?.message ?? err);
  }
}

function maybeFireThresholdAlert(mode) {
  const pct = usageState.count / DAILY_QUOTA;
  const url = process.env.SLACK_NEWSLETTER_WEBHOOK_URL;
  if (!url) return;
  if (!usageState.alertedSoft && pct >= SOFT_THRESHOLD) {
    usageState.alertedSoft = true;
    void postToSlack(url, {
      text: `:warning: Reoon usage at ${Math.round(pct * 100)}% of daily cap (${usageState.count}/${DAILY_QUOTA}, mode=${mode})`,
    });
  }
  if (!usageState.alertedHard && pct >= 1.0) {
    usageState.alertedHard = true;
    void postToSlack(url, {
      text: `:rotating_light: Reoon DAILY CAP HIT (${usageState.count}/${DAILY_QUOTA}). Signups continue with fail-open posture — no deliverability gate until UTC midnight or quota upgrade.`,
    });
  }
}

async function fetchWithTimeout(url, ms) {
  const ctrl = new AbortController();
  const t = setTimeout(() => ctrl.abort(), ms);
  try { return await fetch(url, { signal: ctrl.signal }); }
  finally { clearTimeout(t); }
}

export async function verifyQuick(email) {
  const key = process.env.REOON_API_KEY;
  if (!key) {
    return { ok: true, status: "skipped", reason: null, skipped: true, raw: null };
  }
  const url = `${REOON_BASE}?email=${encodeURIComponent(email)}&key=${encodeURIComponent(key)}&mode=quick`;
  let resp;
  try {
    resp = await fetchWithTimeout(url, QUICK_TIMEOUT_MS);
  } catch (err) {
    console.warn(`[reoon] quick fetch failed for ${email}:`, err?.message ?? err);
    return { ok: true, status: "fetch_error", reason: null, raw: null };
  }
  bumpUsage(1, "quick");
  if (!resp.ok) {
    console.warn(`[reoon] quick returned ${resp.status} for ${email}`);
    return { ok: true, status: `http_${resp.status}`, reason: null, raw: null };
  }
  let body;
  try { body = await resp.json(); }
  catch (_err) { return { ok: true, status: "parse_error", reason: null, raw: null }; }
  const status = body?.status ?? "unknown";
  if (QUICK_REJECT_STATUSES.has(status)) {
    return { ok: false, status, reason: status, raw: body };
  }
  return { ok: true, status, reason: null, raw: body };
}

async function runPower(email, key, context) {
  const url = `${REOON_BASE}?email=${encodeURIComponent(email)}&key=${encodeURIComponent(key)}&mode=power`;
  let resp;
  try { resp = await fetchWithTimeout(url, POWER_TIMEOUT_MS); }
  catch (err) {
    console.warn(`[reoon] power fetch failed for ${email}:`, err?.message ?? err);
    return;
  }
  bumpUsage(1, "power");
  if (!resp.ok) {
    console.warn(`[reoon] power returned ${resp.status} for ${email}`);
    return;
  }
  let body;
  try { body = await resp.json(); } catch { return; }
  const status = body?.status ?? "unknown";
  if (status === "safe") return;
  if (POWER_FLAG_STATUSES.has(status)) {
    const slackUrl = context?.slackWebhookUrl;
    if (slackUrl) {
      void postToSlack(slackUrl, {
        text: `:mag: Reoon power flagged ${email} as "${status}" (source: ${context?.source ?? "unknown"}). Action: ${recommendedAction(status)}`,
      });
    }
  }
}

function recommendedAction(status) {
  switch (status) {
    case "catch_all":    return "domain accepts all addresses — moderate spam risk; monitor engagement.";
    case "role_account": return "shared inbox (admin@, info@) — engagement usually low; flag for review.";
    case "inbox_full":   return "mailbox is full — bounces likely; suppress after 1-2 attempts.";
    case "disabled":     return "mailbox is disabled — suppress immediately.";
    case "disposable":   return "disposable provider slipped past quick check — suppress.";
    case "spamtrap":     return "SPAMTRAP — suppress immediately; investigate signup source.";
    case "unknown":      return "Reoon couldn't determine — leave in, monitor.";
    default:             return "review manually.";
  }
}

export function verifyPowerAsync(email, context) {
  const key = process.env.REOON_API_KEY;
  if (!key) return;
  void runPower(email, key, context).catch((err) => {
    console.error("[reoon] power task crashed:", err);
  });
}

export function userMessageForQuickReject(status) {
  switch (status) {
    case "invalid":    return "That email address doesn't look reachable. Can you double-check it?";
    case "disposable": return "Please use your regular email instead of a temporary/disposable address.";
    case "spamtrap":   return "Invalid email address.";  // intentionally generic — don't tell an attacker we detected them
    default:           return "Invalid email address.";
  }
}

export function getReoonUsageSnapshot() {
  const today = todayUtc();
  const stale = usageState.date !== today;
  return {
    date: stale ? today : usageState.date,
    credits_used_today: stale ? 0 : usageState.count,
    daily_quota: DAILY_QUOTA,
    pct_used: stale ? 0 : Math.round((usageState.count / DAILY_QUOTA) * 1000) / 10,
    alerted_soft_today: stale ? false : usageState.alertedSoft,
    alerted_hard_today: stale ? false : usageState.alertedHard,
    note: "In-memory; resets at UTC midnight or container restart. Real ceiling = Reoon 429.",
  };
}
```

I also write a one-off smoke test at `scripts/test-email-verifier.mjs` so the user can spot-check Reoon post-wiring without going through the full form pipeline:

```javascript
// scripts/test-email-verifier.mjs
// Run: node scripts/test-email-verifier.mjs
// Consumes Reoon credits — designed for one-off verification, not CI.

import { verifyQuick, userMessageForQuickReject } from "../email-verifier.js";

const TESTS = [
  { email: "real@gmail.com",                       expect: "accept (valid Gmail)" },
  { email: "totallyfake1234567@gmail.com",         expect: "quick says valid; power should flag" },
  { email: "qwerty123@notarealdomain12345.com",    expect: "reject (no MX)" },
  { email: "test@mailinator.com",                  expect: "reject (disposable)" },
  { email: "admin@example.com",                    expect: "depends on role detection" },
];

async function run() {
  for (const t of TESTS) {
    const start = Date.now();
    const result = await verifyQuick(t.email);
    const ms = Date.now() - start;
    const verdict = result.ok ? "ACCEPT" : `REJECT (${result.reason})`;
    const userMsg = result.ok ? "" : ` -> "${userMessageForQuickReject(result.reason)}"`;
    console.log(`[${ms}ms] ${t.email}\n  expected: ${t.expect}\n  verdict: ${verdict} (status="${result.status}")${userMsg}\n`);
  }
}
run().catch((e) => { console.error("FAILED:", e); process.exit(1); });
```

The threshold-alert + admin-endpoint pattern in this module is the canonical implementation of [`_internal/reference-quota-monitoring.md`](_internal/reference-quota-monitoring.md), which generalizes it to other vendors with per-day or per-month caps (Resend monthly quota, PostHog volume, B2 storage).

### Step 5: I wire form endpoints from `forms-manifest.md`

I read `forms-manifest.md` (produced by Stage 0) and generate ONE handler per form using the form-handler factory pattern from [`_internal/reference-forms-and-persistence.md`](_internal/reference-forms-and-persistence.md). The endpoint paths come from each form's `name` (e.g., `name: "newsletter_signup"` → `/api/newsletter-signup`); they are NOT hardcoded to `/api/newsletter` and `/api/contact`.

**Reoon gating in the validate layer**: every form handler follows this shape after the syntax check:

```javascript
// In server.js (assuming email-verifier.js is at project root, alongside server.js)
import { verifyQuick, verifyPowerAsync, userMessageForQuickReject } from "./email-verifier.js";

app.post("/api/<form-name>", async (req, res) => {
  // Layer 0a: syntax
  const email = (req.body?.email || "").toString().trim().toLowerCase();
  if (!email || !EMAIL_RE.test(email) || email.length > 254) {
    return res.status(400).json({ ok: false, error: "Invalid email address." });
  }

  // Layer 0b: Reoon quick gate (blocking, ~250ms; fail-open if Reoon down)
  const quick = await verifyQuick(email);
  if (!quick.ok) {
    return res.status(400).json({ ok: false, error: userMessageForQuickReject(quick.reason) });
  }

  // Layer 1: DB capture
  // Layer 2: notification email
  // Layer 3: ESP segment sync
  // Layer 4: CRM sync
  // Layer 5: PostHog event
  // ... (per the form-handler factory)

  // Newsletter handlers only — fire-and-forget power verification AFTER the 200 OK.
  // Contact handlers skip this; contact volume is low and the message body is the
  // spam tell, not the sender's email status.
  if (FORM_TYPE === "newsletter") {
    verifyPowerAsync(email, {
      source: "<form-name>",
      slackWebhookUrl: process.env.SLACK_NEWSLETTER_WEBHOOK_URL,
    });
  }

  return res.json({ ok: true });
});
```

I also add the admin spot-check endpoint to the server:

```javascript
import { getReoonUsageSnapshot } from "./email-verifier.js";

app.get("/api/_admin/reoon-usage", (_req, res) => {
  res.set("Cache-Control", "no-store, no-cache, must-revalidate");
  res.json(getReoonUsageSnapshot());
});
```

This endpoint is intentionally not behind auth — it's behind an obscure URL path. The threat model: an attacker who guesses the URL learns the credit count, which is not sensitive. The Basic Auth staging middleware from Stage 3 also gates this path on staging; in production the obscure path is the only barrier.

The four-layer model runs in every handler:
1. **Validate** against the form's discovered field schema
2. **DB capture** (Layer A; default ON) — `INSERT INTO form_submissions`
3. **Notification email** (Layer B; default ON if `EMAIL_NOTIFY_TO` set) — to user's inbox
4. **ESP segment sync** (Layer C; only if user mapped this form to a segment)
5. **CRM sync** (Layer D; only if user mapped this form to a stage)
6. **PostHog event** — `<form_name>_completed`

Per-form routing decisions live in `forms.per_form_routing` in skill config — populated during Steps F1–F6 in the reference doc.

**The illustrative example below shows what a generated handler looks like for two common form shapes** (newsletter + contact). The actual handlers I generate match whatever forms `forms-manifest.md` describes — the names, fields, and routing differ per project. I do NOT hardcode `/api/newsletter` and `/api/contact`.

**Example — newsletter (single-email form, generated from a manifest entry):**

```js
import { sendEmail, EMAIL_FROM, EMAIL_NOTIFY_TO, SITE_URL } from "./server/email.js";

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

app.post("/api/newsletter", async (req, res) => {
  const email = (req.body?.email || "").toString().trim().toLowerCase();
  if (!email || !EMAIL_RE.test(email) || email.length > 254) {
    return res.status(400).json({ ok: false, error: "Invalid email address." });
  }
  try {
    await Promise.all([
      sendEmail({
        to: email,
        subject: "Welcome",
        text: newsletterWelcomeText(),
        html: newsletterWelcomeHtml(),
      }),
      sendEmail({
        to: EMAIL_NOTIFY_TO,
        subject: `New newsletter signup: ${email}`,
        text: `${email} subscribed at ${new Date().toISOString()}`,
      }),
    ]);
    return res.json({ ok: true });
  } catch (err) {
    console.error("[/api/newsletter] failed:", err);
    return res.status(500).json({ ok: false, error: "Send failed." });
  }
});

app.post("/api/contact", async (req, res) => {
  const { full_name, email, subject, message, inquiry_type, organization } = req.body || {};
  if (
    !full_name || !email || !subject || !message ||
    !EMAIL_RE.test(email) ||
    full_name.length > 200 || email.length > 254 ||
    subject.length > 300 || message.length > 5000
  ) {
    return res.status(400).json({ ok: false, error: "Invalid submission." });
  }
  try {
    await Promise.all([
      sendEmail({
        to: email,
        subject: "We got your message",
        text: contactAutoresponderText(full_name),
        html: contactAutoresponderHtml(full_name),
      }),
      sendEmail({
        to: EMAIL_NOTIFY_TO,
        replyTo: email,
        subject: `[Contact] ${subject}`,
        text: contactNotificationText({ full_name, email, subject, message, inquiry_type, organization }),
      }),
    ]);
    return res.json({ ok: true });
  } catch (err) {
    console.error("[/api/contact] failed:", err);
    return res.status(500).json({ ok: false, error: "Send failed." });
  }
});
```

I add similar handlers for any other forms Stage 0 catalogued (support, beta-signup, lead-magnet, request-demo, get-a-quote, event-RSVP, custom forms — whatever `forms-manifest.md` describes). The structure stays the same: validate against discovered schema → DB capture → notification email → optional ESP/CRM sync → PostHog event → return success.

**The example handlers above are simplified for illustration** (they show only Layers B + email notifications). The factory-generated handlers in production include all four layers per the reference doc, with the DB / ESP / CRM steps gated on each form's routing config.

### Step 5b: I orchestrate the per-form routing review with the user

After generating the handler skeletons, I walk the user through Steps F1–F6 from [`_internal/reference-forms-and-persistence.md`](_internal/reference-forms-and-persistence.md):

- **F1**: Confirm DB capture default (recommended ON; opt-out gated on CRM coverage)
- **F2**: Per-form review of inferred routing (with the user accepting or adjusting per form)
- **F3**: ESP credential paste + live research of the user's ESP API → segment pick-list
- **F4**: CRM credential paste + live research of the user's CRM API → stage pick-list
- **F5**: I generate the production handlers from manifest + routing config + research
- **F6**: End-to-end synthetic-submission test per form

The reference doc has the full flow + the form-handler factory code template + the `form_submissions` table schema. This stage's job is to execute that flow against the specific user's project.

**While I'm waiting for the user to paste ESP / CRM credentials**, I do parallel work that doesn't depend on them: generate the handler skeletons with Layer A + B already wired, write the validation schemas, write the email templates, write the synthetic-test scripts. When the user pastes credentials, I add the Layer C / D wiring and run the F6 verification.

### Step 6: I write email templates

I keep templates simple — heavy styling hurts deliverability (spam filters score against image-heavy / CSS-heavy mail). Inline styles only, no `<style>` blocks.

Example shell + welcome template. **Color values shown are generic neutrals** — at write time I extract brand colors from the user's existing site (CSS variables, Tailwind config, brand kit if provided) and substitute. If no brand colors are detectable, I keep the neutrals below and ask the user to provide brand hex codes.

```js
// I substitute brand colors at write time. Variables I look for:
const BRAND_BG = "<brand-background>";        // e.g., "#ffffff" or off-white
const BRAND_ACCENT = "<brand-accent>";        // e.g., logo color
const BRAND_TEXT_PRIMARY = "<text-primary>";  // typically near-black
const BRAND_TEXT_MUTED = "<text-muted>";      // typically gray
const BRAND_BORDER = "<border-color>";        // light gray for dividers

function emailShell(bodyHtml) {
  return `<!doctype html>
<html><body style="margin:0;padding:0;background:${BRAND_BG};font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;color:${BRAND_TEXT_PRIMARY};">
  <table role="presentation" cellpadding="0" cellspacing="0" border="0" width="100%" style="background:${BRAND_BG};padding:40px 20px;">
    <tr><td align="center">
      <table role="presentation" cellpadding="0" cellspacing="0" border="0" width="560" style="max-width:560px;background:#ffffff;border-radius:12px;padding:40px;text-align:left;">
        <tr><td style="padding-bottom:28px;border-bottom:1px solid ${BRAND_BORDER};">
          <div style="font-size:14px;letter-spacing:2px;text-transform:uppercase;color:${BRAND_ACCENT};font-weight:600;">Brand Name</div>
        </td></tr>
        <tr><td style="padding-top:28px;font-size:15px;line-height:1.6;color:${BRAND_TEXT_PRIMARY};">${bodyHtml}</td></tr>
        <tr><td style="padding-top:32px;border-top:1px solid ${BRAND_BORDER};font-size:12px;color:${BRAND_TEXT_MUTED};line-height:1.5;">
          You're receiving this because you signed up at <a href="${SITE_URL}" style="color:${BRAND_ACCENT};text-decoration:none;">${SITE_URL.replace(/^https?:\/\//, '')}</a>.
        </td></tr>
      </table>
    </td></tr>
  </table>
</body></html>`;
}

function newsletterWelcomeHtml() {
  return emailShell(`
    <h1 style="font-size:24px;font-weight:300;margin:0 0 16px 0;">Welcome.</h1>
    <p style="margin:0 0 16px 0;">You're in. Thanks for joining the list.</p>
    <p style="margin:0 0 16px 0;">A few times a month you'll hear from us with insights and early-access updates. Unsubscribing is one click whenever you've had enough.</p>
  `);
}

function newsletterWelcomeText() {
  return `Welcome.

You're in. Thanks for joining the list.

A few times a month you'll hear from us with insights and early-access updates. Unsubscribing is one click whenever you've had enough.

—
You're receiving this because you signed up at ${SITE_URL}.`;
}
```

I always write both HTML and text variants. Some email clients render text only; some flag HTML-only mail as suspicious.

**For user-provided strings** (full name, message body) interpolated into HTML, I use an `escapeHtml` helper to prevent injection:

```js
function escapeHtml(s) {
  return String(s).replace(/[&<>"']/g, (c) => ({
    "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;",
  }[c]));
}
```

Used inside template functions: `<h1>Got it, ${escapeHtml(firstName)}.</h1>`.

For the actual brand voice / copy in each template, I draft based on the user's brand description from onboarding, then ask them to review before going live.

### Step 7: I set Resend + Reoon env vars on Railway

I run (from the canonical pattern in Stage 4 Step 3):

```
# Resend (always)
railway variable set RESEND_API_KEY="$(cat .secrets/resend-api-key.txt)"
railway variable set EMAIL_FROM="Brand Name <hello@<user-domain>>"
railway variable set EMAIL_NOTIFY_TO="hello@<user-domain>"

# Reoon (if email.reoon_enabled = true)
railway variable set REOON_API_KEY="$(cat .secrets/reoon-email-verifier-api-key.txt)"
railway variable set REOON_DAILY_QUOTA="1200"  # ← from email.reoon_tier (1=500, 2=1200, 3=2100, 4=3200, 5=4500)

# Slack webhook for Reoon threshold alerts (production-only; safe to leave unset on staging)
railway variable set SLACK_NEWSLETTER_WEBHOOK_URL="<your-slack-webhook-url>"
```

I substitute the real brand name and domain at write time. **All API keys are server-side only** — I never prefix with `VITE_` or `PUBLIC_` (those would bake into the client bundle and expose the key to anyone viewing source).

**For the staging service**: I set the SAME `REOON_API_KEY` (staging shares the credit pool with prod — there's no separate staging account at Reoon). I deliberately leave `SLACK_NEWSLETTER_WEBHOOK_URL` UNSET on staging so threshold alerts only fire from production traffic.

Then I trigger a redeploy: `railway up`.

### Step 8: I run end-to-end test

I submit a test newsletter signup via curl to the deployed site:

```
curl -X POST https://<user-domain>/api/newsletter \
  -H "Content-Type: application/json" \
  -d '{"email":"<test-email>"}'
```

Expected response: `{"ok":true}`.

Then I verify:
- The user's test email address received the welcome email (I ask the user to confirm this — it's the only thing I can't autonomously verify since I can't read their inbox)
- The notification email landed at `EMAIL_NOTIFY_TO` (same — user confirms)
- Resend dashboard shows both sends with status "delivered" — I check this via Resend's API: `GET /emails` filtered by recent timestamp

If sends fail:
- I check `railway logs` for `[email/resend] send error:` entries (current CLI streams by default)
- I query Resend's API for the failed message and report the error code
- Common causes: domain not yet verified, `from` address not on a verified domain, key typo

### Step 9: I update privacy policy

Once Resend is in the data path, I add it to the privacy policy as a sub-processor. Stage 8 owns the privacy policy file; I add a draft entry now that Stage 8 picks up:

```markdown
## Sub-processors
- Resend (email delivery) — receives form submissions for newsletter and
  contact processing. Stores email metadata for 30 days. Privacy policy:
  https://resend.com/legal/privacy-policy
```

For users with a DPA (Data Processing Agreement) requirement (GDPR scope), I note that Resend supports DPAs — request via their support if processing EU data at scale.

## Verification (autonomous)

I run all of these myself before declaring Stage 5 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Domain verified | `GET /domains/{id}` via Resend API | status: "verified" |
| Railway env vars set | `railway variable list` filtered on `RESEND_*`, `EMAIL_*` | All three present |
| Form endpoints respond | `curl POST /api/newsletter` and `/api/contact` against deployed URL | HTTP 200, `{ok:true}` |
| Resend logs sends | `GET /emails` via Resend API (filter recent) | Both test sends visible with status "delivered" |
| Privacy policy mentions Resend | Read draft policy file | Resend listed as sub-processor |

I report results to the user:

```
✅ Stage 5 complete:
   • Resend account active, sending domain mayasconsulting.com verified
   • Transport abstraction at server/email.js (Resend default; swappable)
   • /api/newsletter and /api/contact wired in server.js
   • Welcome + autoresponder + notification templates created
   • Test sends: 2 delivered (1 welcome to your test email, 1 notification
     to hello@mayasconsulting.com)
   • Privacy policy draft updated with Resend as sub-processor

Ready for Stage 6 (PostHog)?
```

## Alternative ESP implementations

The transport abstraction (Step 4) is the swap point. Each alternative ESP implements the same `{ send: async (msg) => { success, messageId | error } }` contract.

**AWS SES** (`@aws-sdk/client-sesv2`):

```js
import { SESv2Client, SendEmailCommand } from "@aws-sdk/client-sesv2";
const ses = new SESv2Client({ region: process.env.AWS_REGION });
return {
  async send(msg) {
    const cmd = new SendEmailCommand({
      FromEmailAddress: msg.from || EMAIL_FROM,
      Destination: { ToAddresses: [msg.to] },
      Content: { Simple: {
        Subject: { Data: msg.subject },
        Body: { Text: { Data: msg.text || "" }, Html: { Data: msg.html || "" } },
      }},
    });
    try {
      const out = await ses.send(cmd);
      return { success: true, messageId: out.MessageId };
    } catch (err) {
      return { success: false, error: err };
    }
  },
};
```

**Postmark** (`postmark`):

```js
import postmark from "postmark";
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);
return {
  async send(msg) {
    try {
      const out = await client.sendEmail({
        From: msg.from || EMAIL_FROM,
        To: msg.to,
        Subject: msg.subject,
        TextBody: msg.text,
        HtmlBody: msg.html,
      });
      return { success: true, messageId: out.MessageID };
    } catch (err) {
      return { success: false, error: err };
    }
  },
};
```

**Mailgun, SendGrid, Sendinblue**: same pattern. Each SDK has its own `send` shape; I adapt the `{ success, messageId | error }` return contract.

## Common gotchas I handle automatically

- **`from` address must be on a verified domain**: sending `From: hello@unverified.com` returns 422 from Resend. I always check `EMAIL_FROM` matches a verified domain before saving.
- **Test email lands in spam**: usually a DKIM/SPF DNS misconfiguration. I run `dig TXT <user-domain> +short` to verify both records present + valid. Resend dashboard's domain page also lints this.
- **Hitting daily quota in development**: Resend free tier is 100/day. If a user runs `npm run dev` repeatedly with form submits, they hit the cap fast. I default the dev environment to use the console transport (don't set `RESEND_API_KEY` locally).
- **Replying to autoresponder bounces back to user**: I set `replyTo: EMAIL_NOTIFY_TO` on the autoresponder so user replies go to the team inbox, not a no-reply address.
- **Email mailbox provisioning lag**: I provision the inbox and verify it's receiving BEFORE deploying the live form endpoints. Otherwise users sign up and the user never sees the notification.

## Self-research instruction

Before running this stage, I web-search:
- `Resend MCP server <year>` — none as of 2026-05; if one exists now, I use it instead of REST
- `Resend API rate limits <year>` — quotas may have shifted
- `<chosen-ESP> Cloudflare DNS auto-provision <year>` — if a DNS-auto-add integration shipped, Step 2 Path A becomes one-click

## Outputs

After Stage 5:
- Verified sending domain in Resend (or alternative ESP)
- `server/email.js` with swappable transport
- Form endpoints wired in `server.js`
- Email templates with brand-appropriate copy + styling
- Real emails flowing from real form submits
- Privacy policy draft updated to list ESP as sub-processor

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. No Resend MCP exists in 2026-05; first API key requires dashboard click. Sub-keys + domains programmatic via REST. DNS records always require user to add to DNS provider (or Cloudflare API side-channel automation). |
| 2026-05-08 | Reframed for non-coder audience: every "Run X" / "Edit Y" reframed to first-person Claude voice for actions Claude actually executes. Step 1 user instructions made honest: signup is a user action ("Please open https://resend.com/signup; I open it via Chrome MCP if connected") rather than a fictional Claude-drives-browser claim. Step 2 split into Path A (user adds DNS records, fastest for one-domain) vs Path B (Cloudflare API automation if user grants me a token, fastest if they have one). Replaced literal author-domain examples with `<user-domain>` substitution markers. Stripped author-specific knowledge-base references and replaced author-home-directory secret paths with project-local equivalents. |
| 2026-05-21 | **v2 delta**: ported Reoon Email Verifier as a default integration (v1 had this as a TODO comment block). Added Step 2b (Reoon account + API key collection + AppSumo tier ladder), Step 4b (full `email-verifier.js` implementation with quick + power + threshold alerts + admin endpoint), Step 5 form-handler update (Reoon gating in the validate layer), Step 7 env-var additions (`REOON_API_KEY`, `REOON_DAILY_QUOTA`, `SLACK_NEWSLETTER_WEBHOOK_URL`). Documented fail-open posture (network errors / timeouts / non-200s / 429s / unset API key all return `ok: true`). The threshold-alert + admin-endpoint pattern is documented as canonical implementation of the generalized `_internal/reference-quota-monitoring.md` reference. |
