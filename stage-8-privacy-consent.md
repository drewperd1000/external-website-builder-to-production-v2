# Stage 8: Privacy + Consent

**Last verified: 2026-05-08.**

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) write the consent state machine, the geo-detection helper, the consent banner component, the privacy policy + terms pages, and (for health-adjacent sites) the Consumer Health Data policy page. I adapt PostHog/Clarity init code from Stages 6 + 7 to the chosen privacy posture. The user's actions: (1) confirm the privacy posture I derived from their onboarding answers, (2) review draft policy text I produce and provide brand-specific edits, (3) sign DPAs with each vendor when they arrive. Everything else autonomous.

## Required-values check (run before any code is written in this stage)

Before I write the consent module, I read `<project>/.skill-config.json` and check for these required values:

| Field | Used for | If missing |
|---|---|---|
| `project.brand_prefix` | Cookie + localStorage key namespacing (`<prefix>_analytics_consent`, `<prefix>_consent_version`) | I prompt the user now — see fallback below |
| `privacy.posture` | Decides banner-show logic | I derive from `privacy.eu_visitors` + `privacy.health_adjacent` if posture itself is null |
| `privacy.consent_version` | Sentinel that lets me re-prompt all users when policy changes | I auto-generate as today's date + `-v1` if null |
| `analytics.proxy_path_slug` | Embedded in scripts the consent banner gates | Stage 6 sets this; if missing, that means Stage 6 hasn't run — I block and direct user back to Stage 6 |

**If `brand_prefix` is missing** (older saved configs may not have this field), I prompt:

```
Quick one — I need a "brand prefix" for the cookie + localStorage keys
this stage will set on visitors' browsers. This is a 2–6 character
lowercase token that namespaces the data so it doesn't collide with
other sites the visitor has open.

Based on your project name, I propose: "<auto-derived-from-slug>_"

  (a) Use that — looks fine
  (b) Pick my own — I'll type a 2–6 char lowercase string
  (c) Just decide for me with no input — I'll go with the proposal

(For full context on what this is and why, see Section A in onboarding.md.)
```

I save the answer to `project.brand_prefix` in skill config so future stages and future runs don't ask again. If the user picks (b), I sanitize their input (strip spaces, lowercase, cap at 6 chars + `_`).

**While the user answers the brand-prefix question**, I can do work that doesn't depend on it: drafting the privacy policy text, writing the geo-detection helper (`detectCountry`, `isGdprLocale`), and scaffolding the policy page templates. I block on writing the actual ConsentBanner component until I have the prefix.

## What I'm doing in this stage

Wiring a privacy posture appropriate to the site's audience, content sensitivity, and risk tolerance. After this stage:

- A consent banner appears in the configured locales (or everywhere, depending on posture)
- Analytics tools (PostHog, Clarity) respect consent state
- Server-side consent persistence works for authenticated users (if applicable)
- Privacy policy + Terms pages are wired
- (If health-adjacent) Consumer Health Data policy page is wired with prominent homepage link
- DPA signing process is queued
- Right-to-delete operator runbook exists

**Time**: 20–40 minutes of automated work + user review of draft policy text.

## Privacy posture (auto-derived from onboarding answers)

Onboarding asked two yes/no questions: "Will EU/UK/EEA visitors see this site?" and "Is the content health-adjacent?" From those, I derive a posture using this table:

| EU visitors? | Health-adjacent? | Posture I apply |
|---|---|---|
| Yes | Yes | **Strict**: banner everywhere, opt-in everywhere. Analytics dormant until Accept. |
| Yes | No | **Standard**: banner in EU/UK/EEA locales (Cloudflare-detected), opt-out elsewhere |
| No | Yes | **Standard**: banner shown per US state laws like Washington MHMD; analytics opt-out elsewhere |
| No | No | **Light**: no banner, respect Do-Not-Track header only |

I confirm the derived posture with the user before applying:

```
Based on your onboarding answers (EU visitors: yes, health-adjacent: no),
I'm applying the "Standard" posture:

  • Cookie banner appears for EU/UK/EEA visitors (auto-detected via Cloudflare)
  • Analytics opt-in for those visitors; opt-out elsewhere
  • Analytics defaults active for non-EU visitors with a footer link
    they can click to opt out
  • PostHog session recording defaults on for accepted users; off for declined

This is what most consumer-product marketing sites use. Want to override?
  (a) Keep Standard (recommended)
  (b) Strict — banner everywhere, opt-in everywhere
  (c) Light — no banner, respect Do-Not-Track only (only with explicit
      legal advice)
  (d) Custom — describe your matrix; I'll encode it
```

Default to (a). If the user picks (b), (c), or (d), I adapt the code accordingly.

## Prerequisites

- Stages 6 + 7 complete (PostHog + Clarity wired).
- Privacy posture decision recorded in the saved skill config (from onboarding or this stage's confirmation).
- DNS on Cloudflare if using Standard posture (geo detection uses Cloudflare's `cdn-cgi/trace` endpoint). If not on Cloudflare, I fall back to a third-party geo service (`ipapi.co/json/` is the typical replacement) and document the trade-off.

## My execution sequence

### Step 1: I write the geo-detection helper

I create `src/_core/geo.ts`:

```ts
/**
 * Visitor country detection via Cloudflare's free /cdn-cgi/trace endpoint.
 * Available on every CF-fronted domain; returns plain text including loc=XX.
 *
 * Result cached in sessionStorage. Conservative failure mode: unknown country
 * is treated as GDPR (over-show banner > under-show banner).
 */
const COUNTRY_CACHE_KEY = "<brand-prefix>_geo_country";  // I substitute brand prefix

const GDPR_LOCALES: ReadonlySet<string> = new Set([
  // EU 27
  "AT","BE","BG","HR","CY","CZ","DK","EE","FI","FR","DE","GR","HU","IE",
  "IT","LV","LT","LU","MT","NL","PL","PT","RO","SK","SI","ES","SE",
  // EEA non-EU
  "IS","LI","NO",
  // UK + Crown Dependencies
  "GB","JE","GG","IM",
]);

export function isGdprLocale(country: string | null | undefined): boolean {
  if (!country) return true;  // unknown → conservative: show banner
  return GDPR_LOCALES.has(country.toUpperCase());
}

export async function detectCountry(): Promise<string | null> {
  if (typeof window === "undefined") return null;

  // ─────────────────────────────────────────────────────────────────────
  // Dev-only override: ?force_gdpr (any value) returns "FR" so the GDPR
  // path is testable from localhost without a VPN. ?force_country=XX
  // returns the literal country code for testing arbitrary locales.
  //
  // Guarded by import.meta.env.DEV — Vite eliminates this whole block as
  // dead code during production minification. Safe to commit.
  // ─────────────────────────────────────────────────────────────────────
  if (import.meta.env.DEV) {
    const params = new URLSearchParams(window.location.search);
    const forced = params.get("force_country") ?? params.get("force_gdpr");
    if (forced !== null) {
      return forced === "" ? "FR" : forced.toUpperCase();
    }
  }

  // Normal path
  try {
    const cached = window.sessionStorage.getItem(COUNTRY_CACHE_KEY);
    if (cached) return cached;
  } catch { /* sessionStorage blocked */ }

  try {
    const r = await fetch("/cdn-cgi/trace", { credentials: "omit" });
    if (!r.ok) return null;
    const text = await r.text();
    const m = /^loc=([A-Z]{2})$/m.exec(text);
    if (!m) return null;
    const country = m[1];
    try {
      window.sessionStorage.setItem(COUNTRY_CACHE_KEY, country);
    } catch { /* best effort */ }
    return country;
  } catch {
    return null;
  }
}
```

I substitute `<brand-prefix>` with a short slug derived from the brand name (e.g., `mc` for Maya's Consulting → `mc_geo_country`). This keeps localStorage namespaced per project so multiple sites in the same browser don't collide.

**If the user's DNS isn't on Cloudflare**, I swap the fetch URL for `https://ipapi.co/json/` and update the parser. ipapi.co has a free tier of 30k requests/day which is fine for consumer sites.

### Step 2: I write the consent state machine

I create `src/_core/consent.ts`:

```ts
/**
 * Analytics consent state for anonymous visitors.
 * Three states: "accepted" / "declined" / "unknown" (banner shows when unknown).
 *
 * posthog-js maintains its own opt-out state in localStorage under a different
 * key. When we call opt_out_capturing(), posthog persists that across reloads.
 * We track our own key here purely to know whether to render the banner.
 */
const STORAGE_KEY = "<brand-prefix>_analytics_consent";

export type ConsentState = "accepted" | "declined" | "unknown";

export function readConsent(): ConsentState {
  if (typeof window === "undefined") return "unknown";
  try {
    const v = window.localStorage.getItem(STORAGE_KEY);
    if (v === "accepted" || v === "declined") return v;
  } catch { /* localStorage blocked */ }
  return "unknown";
}

export function writeConsent(state: "accepted" | "declined"): void {
  if (typeof window === "undefined") return;
  try {
    window.localStorage.setItem(STORAGE_KEY, state);
  } catch { /* best effort */ }
}

export function clearConsent(): void {
  if (typeof window === "undefined") return;
  try {
    window.localStorage.removeItem(STORAGE_KEY);
  } catch { /* best effort */ }
}
```

For Strict posture's authenticated-users path, I extend this with a server-side mirror (DB columns + version sentinel + cross-device reconciliation matrix). I only do this if the user's site has authenticated users; for marketing-only sites, the anonymous-only state machine is sufficient.

### Step 3 (authenticated-users only): server-side consent persistence

For apps with auth, I add two columns to the users table via the user's preferred migration tool (Drizzle, Prisma, raw SQL):

```sql
ALTER TABLE users ADD COLUMN analytics_consent_at TIMESTAMP;
ALTER TABLE users ADD COLUMN analytics_consent_version VARCHAR(64);
```

I define a sentinel value for "withdrawn" (`ANALYTICS_CONSENT_WITHDRAWN = "WITHDRAWN"`) and the current version (`ANALYTICS_CONSENT_VERSION = "<YYYY-MM-DD>-v1"`) in shared constants. WITHDRAWN means "no" regardless of version (declining means no to whatever set of tools we have).

I bump the version sentinel whenever:
- Banner copy materially changes
- The set of analytics tools changes (added/removed Clarity → bump)
- Privacy policy materially changes

Stale versions in localStorage = "unknown" → banner re-prompts. Server-side stale = same.

I write the cross-device reconciliation logic for login (`null` → authenticated transition):

| Server | Local | Action |
|---|---|---|
| accept | (any) | apply accept on this device, opt-in |
| decline | (any) | apply decline on this device, opt-out |
| unknown | accept | upload accept to server, keep local |
| unknown | decline | upload decline to server, keep local |
| unknown | unknown | show banner |

### Step 4: I write the ConsentBanner component

I check `<project>/.skill-config.json` for `styling.system`, set in Stage 1 (auto-detected from `package.json` deps + presence of `tailwind.config.*` / `postcss.config.*`). The banner picks one of two implementations:

- **`tailwind`** (if Tailwind detected) → I use the JSX class-based version below.
- **`plain`** (everything else: vanilla CSS, CSS modules, styled-components, Webflow's CSS, Framer's css-in-js) → I use the inline-styles version below. The visual outcome is identical; only the styling mechanism differs.

I do NOT assume Tailwind. If the project doesn't have it and the user wants the Tailwind aesthetic, I offer to add Tailwind in Stage 1 (it's a 90-second install) — but if they decline, I ship the inline-styles version and the banner still works.

#### Implementation A — Tailwind version (when `styling.system === "tailwind"`)

```tsx
// src/components/ConsentBanner.tsx
import { useEffect, useState } from "react";
import posthog from "posthog-js";
import { readConsent, writeConsent } from "@/_core/consent";
import { detectCountry, isGdprLocale } from "@/_core/geo";
import { initClarity } from "@/_core/analytics/clarity";

export function ConsentBanner({ posture }: { posture: "strict" | "standard" | "light" }) {
  const [show, setShow] = useState(false);

  useEffect(() => {
    let cancelled = false;
    (async () => {
      if (readConsent() !== "unknown") return;
      if (posture === "strict") {
        if (!cancelled) setShow(true);
        return;
      }
      if (posture === "light") {
        return;  // never show — DNT is the only opt-out signal
      }
      // Standard: show only in GDPR locales
      const country = await detectCountry();
      if (cancelled) return;
      if (isGdprLocale(country)) setShow(true);
    })();
    return () => { cancelled = true; };
  }, [posture]);

  if (!show) return null;

  function accept() {
    writeConsent("accepted");
    setShow(false);
    if (typeof posthog !== "undefined") {
      posthog.opt_in_capturing();
    }
    initClarity();  // (or initClarityIfPermitted for Variant B apps)
  }

  function decline() {
    writeConsent("declined");
    setShow(false);
    if (typeof posthog !== "undefined") {
      posthog.opt_out_capturing();
    }
    // Clarity snippet was never injected → nothing to disable
  }

  return (
    <div className="fixed bottom-0 inset-x-0 bg-white border-t border-gray-200 p-4 shadow-lg z-50">
      <div className="max-w-4xl mx-auto flex flex-col sm:flex-row items-start sm:items-center gap-4">
        <p className="text-sm flex-1">
          We use analytics to understand how visitors use the site. You can{" "}
          <a href="/privacy" className="underline">read our privacy policy</a>{" "}
          for details. Click Accept to allow analytics, or Decline to opt out.
        </p>
        <div className="flex gap-2">
          <button onClick={decline} className="px-4 py-2 text-sm border rounded">
            Decline
          </button>
          <button onClick={accept} className="px-4 py-2 text-sm bg-black text-white rounded">
            Accept
          </button>
        </div>
      </div>
    </div>
  );
}
```

#### Implementation B — Inline-styles version (when `styling.system === "plain"`)

Same hooks logic; only the JSX changes. No CSS framework dependency.

```tsx
// src/components/ConsentBanner.tsx
import { useEffect, useState } from "react";
import posthog from "posthog-js";
import { readConsent, writeConsent } from "@/_core/consent";
import { detectCountry, isGdprLocale } from "@/_core/geo";
import { initClarity } from "@/_core/analytics/clarity";

const styles = {
  banner: {
    position: "fixed" as const,
    bottom: 0,
    left: 0,
    right: 0,
    background: "#fff",
    borderTop: "1px solid #e5e7eb",
    padding: "16px",
    boxShadow: "0 -4px 12px rgba(0,0,0,0.08)",
    zIndex: 50,
  },
  inner: {
    maxWidth: "896px",
    margin: "0 auto",
    display: "flex",
    flexWrap: "wrap" as const,
    alignItems: "center",
    gap: "16px",
  },
  text: { flex: 1, fontSize: "14px", lineHeight: 1.5, margin: 0, minWidth: "240px" },
  link: { textDecoration: "underline", color: "inherit" },
  buttonRow: { display: "flex", gap: "8px" },
  button: {
    padding: "8px 16px",
    fontSize: "14px",
    border: "1px solid currentColor",
    borderRadius: "4px",
    background: "transparent",
    cursor: "pointer",
  },
  buttonPrimary: {
    padding: "8px 16px",
    fontSize: "14px",
    border: "1px solid #000",
    borderRadius: "4px",
    background: "#000",
    color: "#fff",
    cursor: "pointer",
  },
};

export function ConsentBanner({ posture }: { posture: "strict" | "standard" | "light" }) {
  const [show, setShow] = useState(false);

  useEffect(() => {
    let cancelled = false;
    (async () => {
      if (readConsent() !== "unknown") return;
      if (posture === "strict") { if (!cancelled) setShow(true); return; }
      if (posture === "light") return;
      const country = await detectCountry();
      if (cancelled) return;
      if (isGdprLocale(country)) setShow(true);
    })();
    return () => { cancelled = true; };
  }, [posture]);

  if (!show) return null;

  function accept() {
    writeConsent("accepted");
    setShow(false);
    if (typeof posthog !== "undefined") posthog.opt_in_capturing();
    initClarity();
  }

  function decline() {
    writeConsent("declined");
    setShow(false);
    if (typeof posthog !== "undefined") posthog.opt_out_capturing();
  }

  return (
    <div style={styles.banner} role="dialog" aria-label="Privacy consent">
      <div style={styles.inner}>
        <p style={styles.text}>
          We use analytics to understand how visitors use the site. You can{" "}
          <a href="/privacy" style={styles.link}>read our privacy policy</a>{" "}
          for details. Click Accept to allow analytics, or Decline to opt out.
        </p>
        <div style={styles.buttonRow}>
          <button onClick={decline} style={styles.button}>Decline</button>
          <button onClick={accept} style={styles.buttonPrimary}>Accept</button>
        </div>
      </div>
    </div>
  );
}
```

The inline-styles version trades aesthetic refinement for zero framework dependency — visually it's still a clean white footer banner with two buttons, just without Tailwind's polish on the spacing/typography. For a marketing site that lacks Tailwind, this is fine. For an app where the banner sits next to brand-styled UI, I'd suggest adding Tailwind in Stage 1 instead.

I mount the banner in `App.tsx` (or the equivalent app shell) with the posture from the saved config:

```tsx
import { ConsentBanner } from "./components/ConsentBanner";

export default function App() {
  return (
    <>
      <Routes />
      <ConsentBanner posture="standard" />  {/* I substitute the actual posture */}
    </>
  );
}
```

I draft the banner copy based on the user's brand voice from onboarding, then ask them to review before going live.

### Step 5: I adapt PostHog init based on posture

#### Strict posture — start dormant

I update Stage 6's `main.tsx` to start session recording disabled and only enable on Accept:

```tsx
posthog.init(posthogKey, {
  // ... existing config
  disable_session_recording: true,  // ← stays off until consent
  loaded: (ph) => {
    if (readConsent() === "accepted") {
      registerDeployTagOnPostHog();
      registerCustomUtmsOnPostHog();
      ph.capture("$pageview", { ... });
      ph.startSessionRecording();
      initAnalytics();
    } else {
      ph.opt_out_capturing();
    }
  },
});
```

#### Standard posture — start active, opt out for declined

```tsx
posthog.init(posthogKey, {
  loaded: (ph) => {
    const consent = readConsent();
    if (consent === "declined") {
      ph.opt_out_capturing();
    } else {
      registerDeployTagOnPostHog();
      registerCustomUtmsOnPostHog();
      ph.capture("$pageview", { ... });
      initAnalytics();
    }
  },
});
```

#### Light posture — respect DNT only

```tsx
posthog.init(posthogKey, {
  respect_dnt: true,  // ← THIS is the key change vs other postures
  loaded: (ph) => {
    registerDeployTagOnPostHog();
    registerCustomUtmsOnPostHog();
    ph.capture("$pageview", { ... });
    initAnalytics();
  },
});
```

### Step 6: I write the privacy policy + terms pages

Required for any posture. I create two routes:

- `/privacy` — names every sub-processor (PostHog, Clarity, Resend, the membership platform, etc.) with retention notes + DPA status
- `/terms` — terms of service

I draft baseline content then ask the user to review:

```
I've drafted /privacy and /terms based on the services you're using.
The drafts cover:

  • Sub-processors with retention windows
  • User rights (delete, export, opt out — methods for each)
  • DPA status placeholders (DPAs to be signed by you with each vendor)
  • Contact email: privacy@<your-domain> (you'll need to provision this
    inbox if it doesn't exist)
  • Effective date: today

Please open /privacy on the dev server and review. I'll need you to:
  1. Confirm the company legal name + address (required for GDPR text)
  2. Adjust any vendor descriptions that don't match how you use them
  3. Add any custom data uses I don't know about

Want me to draft a paragraph for the homepage footer linking to these pages?
```

For baseline templates I use plain English at the appropriate posture level. I recommend the user have a lawyer review before launch — I produce a starting point, not a substitute for legal advice.

### Step 7 (health-adjacent only): Consumer Health Data policy

If the user's `health_adjacent: true` in onboarding, I write `/consumer-health-data`:

```markdown
# Consumer Health Data Privacy Policy

[Brand Name] complies with applicable consumer health data privacy laws, including state-specific statutes such as Washington's My Health, My Data Act and similar regulations in other jurisdictions where they apply. (Specific statutory citation should be added by your legal counsel based on where you operate.)

## What we collect
[I list specific health-related data based on the user's site, e.g.,
sleep tracking, mood ratings, biometric input — only what their
product actually collects]

## Why we collect it
[I list legitimate purposes from the user's product description]

## Who we share it with
[Sub-processors that touch CHD specifically — PostHog if collecting
health events, Clarity ONLY if not excluded by route-scope from
health-data routes]

## Retention
- Server-side database: 7 years (configurable to user request)
- PostHog events: configurable; defaults to 7 years
- Clarity playback: 30 days (Microsoft-imposed, not configurable)
- Clarity aggregate: 13 months

## Your rights (varies by jurisdiction; example: residents of Washington and similar U.S. states with consumer-data laws)
- Right to know: email privacy@<your-domain> — response within applicable statutory deadline (commonly 45 days)
- Right to delete: in-app account deletion OR email privacy@<your-domain>
- Right to withdraw consent: footer "Analytics preferences" link

(I substitute the user's actual jurisdiction list at write time based on `privacy.eu_visitors` + `privacy.health_adjacent` from onboarding. Specific statute citations should be added by the user's legal counsel based on where they operate; this template uses generic phrasing so we don't ship a wrong citation.)

## Effective date
[today's date]
```

I also add a homepage link to this policy (the statute requires this be prominent — typically in the main footer).

### Step 8: I write the operator runbook for data-subject requests

I create `docs/data-subject-runbook.md`:

```markdown
# Data Subject Request Runbook

## Right to delete (MHMD / GDPR / CCPA)
1. User deletes account via /account/settings (existing flow purges users + content rows)
2. PostHog: admin → Persons → search by email → Delete (manual, ~5 min)
3. Clarity: admin → Settings → CCPA/GDPR page → submit user identifier
4. Resend: no per-user delete; sends are 30-day retention; advise user no action needed
5. Confirm completion to user within the applicable statutory deadline (commonly 45 days for U.S. consumer-data laws and GDPR alike — confirm against jurisdictions you operate in)

## Right to know
- Email response listing categories of data collected, sub-processors, retention windows
- Categories listed in /consumer-health-data + /privacy

## Right to withdraw consent (authenticated users)
- User clicks "Analytics preferences" in footer → clears localStorage → banner re-shows
- If authenticated: also call auth.withdrawAnalyticsConsent mutation → server marks WITHDRAWN
```

This is a Claude-readable file the user (or a future-Claude session) can follow when a data-subject request comes in.

### Step 9: I list DPAs to sign

I produce a checklist for the user:

```
DPAs (Data Processing Agreements) to sign — required for GDPR
compliance, recommended regardless. I can't sign these for you;
each requires you to click through their dashboard or email support:

  ☐ PostHog DPA — request via PostHog support
    https://posthog.com/privacy → DPA Request

  ☐ Microsoft Clarity DPA — covered by Microsoft's standard DPA
    https://www.microsoft.com/licensing/docs/customeragreement

  ☐ Resend DPA — email Resend support if processing EU data at scale
    privacy@resend.com

  ☐ <Membership platform> DPA (if applicable) — varies by vendor

  ☐ Cloudflare DPA — accept via Cloudflare dashboard's compliance page

I'll record signing dates in <project>/docs/compliance-artifacts.md as
you complete each.
```

### Step 10: I add a "Reset analytics preferences" footer link

For users who want to change their consent later (especially under right-to-withdraw):

```tsx
import { clearConsent } from "@/_core/consent";

function ResetPreferencesLink() {
  return (
    <button
      onClick={() => {
        clearConsent();
        window.location.reload();  // banner reappears
      }}
      className="text-sm underline"
    >
      Analytics preferences
    </button>
  );
}
```

Mounted in the site footer.

## Verification (autonomous)

I run these myself before declaring Stage 8 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Banner appears in expected locales | `preview_click` to `/?force_gdpr` (forces "FR" via dev override), check banner DOM | Banner element present |
| Banner does NOT appear non-GDPR (Standard posture) | `preview_click` to `/?force_country=US`, check banner DOM | Banner element absent |
| Decline opts out | `preview_click` Decline button, then `preview_eval('posthog.has_opted_out_capturing()')` | Returns `true` |
| Accept opts in | Same pattern, expect `false` | Returns `false` |
| Privacy + Terms routes resolve | Curl `/privacy` and `/terms` on deployed URL | HTTP 200 |
| (If health-adjacent) CHD route resolves | Curl `/consumer-health-data` | HTTP 200, prominent homepage link present |
| Consent persists across reloads | `preview_click` Accept, reload, check banner doesn't reappear | Banner stays hidden |
| `?force_gdpr` is dev-only (verify Vite drops it) | Build production bundle, grep for `force_gdpr` | Zero matches |

I report:

```
✅ Stage 8 complete:
   • Privacy posture: Standard (banner in EU/UK/EEA only, opt-out elsewhere)
   • Geo detection: Cloudflare cdn-cgi/trace + sessionStorage cache
   • Consent state machine wired (accepted / declined / unknown)
   • Banner mounted in App.tsx
   • PostHog init adapted for Standard posture
   • /privacy + /terms routes live with draft content (please review)
   • CHD policy: skipped (not health-adjacent)
   • DPAs: 4 listed, awaiting your signing
   • Operator runbook at docs/data-subject-runbook.md
   • Footer "Analytics preferences" link added
   • Dev override (?force_gdpr) verified absent from production bundle

Drafts to review:
   • /privacy text — needs your company legal name + address
   • /terms text — generic; you may want a lawyer review

Ready for Stage 9 (membership) or skip to Stage 10 / 11 / 12?
```

## Common gotchas I handle automatically

- **Banner doesn't show in EU**: `cdn-cgi/trace` 404s if not behind Cloudflare. I detect this during Step 1 and swap to ipapi.co automatically.
- **Standard-posture conservative-fallback**: if `detectCountry()` errors, `isGdprLocale(null)` returns true → banner shows. SAFE failure mode; EU users err to seeing the banner.
- **`respect_dnt: true` on Standard posture**: would skip the banner entirely for DNT-using EU users (analytics off because DNT, no banner because Standard doesn't respect DNT). I never mix — pick ONE posture and stick with it.
- **Version sentinel forgot bump after copy change**: existing acceptances persist; users see new copy without re-consenting. I always bump the version when banner copy or processor list changes.
- **Server-side withdrawal not wired**: authenticated users withdraw via banner (clears localStorage) but server still has the OLD accept. I wire the mutation; otherwise re-login restores the stale consent.

## Self-research instruction

Before running this stage, I web-search:
- Current GDPR fine cases for analytics-related violations — these shape what "Strict" should mean today
- New state-level US privacy laws (CCPA, CPRA, VCDPA, CPA, CTDPA, UCPA, MCDPA, MHMD already exist; new states keep adding)
- Whether Cloudflare's `/cdn-cgi/trace` is still available (free + reliable as of 2026, but spot-check)

## Outputs

After Stage 8:
- Privacy posture locked
- Consent state machine implemented (anonymous + authenticated paths)
- Banner UI shipped + mounted
- PostHog + Clarity init adapted to posture
- /privacy + /terms + (if applicable) /consumer-health-data pages live
- DPAs queued for user to sign
- Operator runbook for data-subject requests
- Quarterly verification checklist queued for Stage 12

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Three postures documented (Strict / Standard / Light) plus Custom hook. |
| 2026-05-08 | Standard `geo.ts` ships with permanent dev-only override block: `?force_gdpr` returns "FR", `?force_country=XX` returns the literal code. Guarded by `import.meta.env.DEV`; Vite drops it during minification. Eliminates VPN need for testing. |
| 2026-05-08 | Reframed for non-coder audience: replaced "pick Conservative/Standard/Aggressive/Custom" with auto-derivation from onboarding's two yes/no questions (EU visitors? + health-adjacent?). User confirms the derived posture rather than evaluating tier definitions. Replaced literal author-brand storage key prefixes with `<brand-prefix>_*` substitution markers. Replaced literal author-domain references with `<your-domain>` placeholders. All "Add this to your X file" / "Edit Y" reframed to first-person Claude voice. Verification reframed to autonomous Preview MCP checks. |
