# Quota Monitoring — Threshold-Alert + Admin-Endpoint Pattern

> **🤖 This file is for Claude only.** It documents a reusable building block: in-memory counter + UTC midnight reset + 80%/100% Slack threshold alerts + admin spot-check endpoint. The Reoon integration in [`stage-5-email-resend.md`](../stage-5-email-resend.md) is the canonical implementation. The same pattern applies to any vendor with a per-day or per-month cap.

## When to read this file

I read this file:
- When wiring a third-party vendor that has a daily or monthly cap and a meaningful penalty for hitting it (rate limiting, soft suspension, hard charge for overage).
- When the user asks "can we add threshold alerts for [vendor]?" — the answer is yes, and this is the template.
- When refactoring the Reoon integration to share code with a second vendor.

## The pattern in 5 pieces

1. **Module-scope `usageState` object** with `date`, `count`, `alertedSoft`, `alertedHard` fields.
2. **`bumpUsage(credits, mode)` function** — increments the counter and triggers threshold checks. Auto-resets the counter when `usageState.date` rolls over to a new UTC day.
3. **`maybeFireThresholdAlert(mode)` function** — checks if usage crossed 80% or 100% and fires a once-per-day Slack alert to a configured webhook. Production-only (no-ops in dev).
4. **`getUsageSnapshot()` function** — returns a JSON-serializable object for the admin endpoint.
5. **Admin endpoint** — `GET /api/_admin/<vendor>-usage` with `Cache-Control: no-store` and no auth (relies on obscure URL path; Basic Auth staging gate handles staging).

## Generic template (copy + adapt per vendor)

```javascript
// <vendor>-usage.js

const DAILY_QUOTA = parseInt(process.env.<VENDOR>_DAILY_QUOTA || "<default>", 10);
const SOFT_THRESHOLD = 0.8;

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
    if (!r.ok) console.error(`[<vendor>/slack] returned ${r.status}`);
  } catch (err) {
    console.error("[<vendor>/slack] post failed:", err?.message ?? err);
  }
}

function maybeFireThresholdAlert(mode) {
  const pct = usageState.count / DAILY_QUOTA;
  const url = process.env.<VENDOR>_SLACK_WEBHOOK_URL;
  if (!url) return;
  if (!usageState.alertedSoft && pct >= SOFT_THRESHOLD) {
    usageState.alertedSoft = true;
    void postToSlack(url, {
      text: `:warning: <Vendor> usage at ${Math.round(pct * 100)}% of daily cap (${usageState.count}/${DAILY_QUOTA}, mode=${mode})`,
    });
  }
  if (!usageState.alertedHard && pct >= 1.0) {
    usageState.alertedHard = true;
    void postToSlack(url, {
      text: `:rotating_light: <Vendor> DAILY CAP HIT (${usageState.count}/${DAILY_QUOTA}).`,
    });
  }
}

export function getUsageSnapshot() {
  const today = todayUtc();
  const stale = usageState.date !== today;
  return {
    date: stale ? today : usageState.date,
    credits_used_today: stale ? 0 : usageState.count,
    daily_quota: DAILY_QUOTA,
    pct_used: stale ? 0 : Math.round((usageState.count / DAILY_QUOTA) * 1000) / 10,
    alerted_soft_today: stale ? false : usageState.alertedSoft,
    alerted_hard_today: stale ? false : usageState.alertedHard,
    note: "In-memory; resets at UTC midnight or container restart.",
  };
}

// In server.js:
//   app.get("/api/_admin/<vendor>-usage", (_req, res) => {
//     res.set("Cache-Control", "no-store, no-cache, must-revalidate");
//     res.json(getUsageSnapshot());
//   });

export { bumpUsage };
```

## Candidate applications

| Vendor | Cap | Penalty | Worth wiring? |
|---|---|---|---|
| **Reoon Email Verifier** | Daily credit cap from AppSumo LTD tier (500–4500/day) | 429 + fail-open posture means silent deliverability gap | ✅ Done (Stage 5 reference impl) |
| **Resend** | Monthly cap (3,000/mo free, paid tiers higher) | Hard throttle + 429 | ✅ Worth adding when site exceeds ~50% of monthly cap consistently |
| **PostHog** | Monthly event cap (1M events/mo free, paid higher) | Soft throttle + paid-plan upsell | ✅ Worth adding for high-traffic sites |
| **Backblaze B2** | Daily download cap (~3x storage size free) | Charges on overage | ⚠️ Only if site serves heavy media. Caps are usually generous. |
| **Cloudflare Workers** | Daily request cap (100k/day free) | 1027 error response | ⚠️ Most marketing sites don't hit this. |
| **Switchy** | Monthly link-clicks (varies by AppSumo tier) | Throttled redirects | ⚠️ Niche — for affiliate-heavy sites. |
| **OpenAI / Anthropic API** | Tier-based rate limits | Per-request 429 | Different pattern needed (per-request, not per-day — see [vendor SDK retry guidance]) |

## Adapting the pattern

When wiring this for a vendor other than Reoon:

1. Copy the template above; substitute `<vendor>` and `<VENDOR>` consistently.
2. Adjust `DAILY_QUOTA` env var name and default value to match the vendor's pricing.
3. Adjust the Slack message text to mention the vendor's name and the consequence of hitting the cap (different vendors have different penalties).
4. Choose where `bumpUsage()` is called — typically inside the wrapper function that makes the vendor API call. If a single request uses multiple credits (e.g., Reoon power mode = 1 credit per call but 1 call ≠ 1 user action), the cost should be calculated at the right granularity.
5. Add the admin endpoint to `server.js` with an obscure path.
6. Set the env vars on Railway: `<VENDOR>_DAILY_QUOTA`, `<VENDOR>_SLACK_WEBHOOK_URL`.

## Why not a shared module?

Each vendor's cap has different semantics:
- Reoon = credits (1 quick = 1 credit, 1 power = 1 credit; 1 newsletter signup = 2 credits)
- Resend = email sends (1 send = 1 cap unit, regardless of recipient count up to provider limits)
- PostHog = events (1 `$pageview` = 1 event; 1 `$identify` = 1 event)
- B2 = bytes (highly variable per request)

A shared module would either need to be agnostic to credit semantics (which is what this template essentially is) or would need to know each vendor's semantics (defeating the purpose). The pragmatic answer is: copy the template per vendor, customize the bump granularity.

If the project ends up with >3 vendors using this pattern, I'd extract a `usage-counter-base.js` that exposes `createUsageCounter({ envQuotaName, defaults, slackWebhookEnvName })` and returns a `{ bumpUsage, getSnapshot }` pair. But until then, copy-paste is fine.

## Common gotchas

- **Container restarts under-count.** Railway redeploys (or any container restart) reset `usageState.count` to 0 mid-day. The counter shows lower-than-real numbers until UTC midnight rolls over and the count starts fresh. The vendor's own dashboard is the authoritative source for billing — this counter is for "should we alert?" purposes, not for accounting.
- **Multiple Railway replicas under-count too.** If the site scales horizontally, each replica has its own in-memory counter. Threshold alerts fire when ANY replica crosses 80%/100%, but the real aggregate usage is the sum across replicas. For caps that matter financially, the vendor's dashboard remains authoritative.
- **Once-per-day dedup is per-container, not per-day-globally.** A container restart resets `alertedSoft` and `alertedHard` to `false`. After restart, the next crossing fires a new alert. This is a feature (you want to know if usage is still climbing after the previous alert) not a bug, but worth understanding.
- **The Slack channel can get noisy.** If a site routinely hits 80% (e.g., quota is too low for traffic), the daily alert becomes wallpaper. Either bump the quota (buy another tier) or raise the soft threshold to 0.9. Don't silence alerts entirely — they're the cheap insurance against a silent rate limit blocking real users.

## Cross-references

- [`stage-5-email-resend.md`](../stage-5-email-resend.md) — Reoon canonical implementation (Step 4b)
- [`_internal/claude-invariants.md`](claude-invariants.md) — Invariant #8: fail-open posture for capped external dependencies
