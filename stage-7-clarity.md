# Stage 7: Microsoft Clarity Setup

**Last verified: 2026-05-08.** Re-research before this stage if older than 60 days. Search: `Microsoft Clarity MCP server <year>`. Read `learn.microsoft.com/clarity/third-party-integrations/clarity-mcp-server` for current capability list.

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) write the Clarity loader, the flag-bridge, the deploy-tag extension, the masking helpers, the Railway env var, and the verification. The user's actions: (1) sign up at clarity.microsoft.com and create a project (~3 min in dashboard), (2) configure dashboard masking (Balanced mode + add `.clarity-mask` to mask-by-element list, ~1 min), (3) optionally generate a Clarity API token and paste it for the MCP. Everything else autonomous.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 7 needs:
- `analytics.clarity.enabled` — `true` for `analytics.stack === "both"` or `"clarity_only"`; `false` skips this stage entirely.
- `analytics.clarity.project_id` — populated by Stage 7 itself once the user creates the project; missing is expected at start.
- `privacy.posture` — drives Variant A (simple gate) vs Variant B (strict gate) for Clarity injection. If unset, Stage 8 hasn't run yet — I block until Stage 8 sets the posture, OR I derive a default from `privacy.eu_visitors` + `privacy.health_adjacent` if those are set.
- The user's Clarity Project ID (a string like `wge0...`), pasted into `<project>/.secrets/clarity-project-id.txt`. **Parallel work**: I can write the loader module, flag-bridge, masking helpers, and verification scripts while the user signs up.

## What I'm doing in this stage

Wiring Microsoft Clarity for heatmaps + session replay alongside PostHog. Clarity's strength is the visual layer (rage clicks, dead clicks, scroll depth, recording playback); PostHog handles structured event analysis. Both run side-by-side.

**After this stage**:
- Clarity snippet loaded on the site (with consent + scope gates appropriate to the site type)
- Triple-redundant content masking active (CSS class + DOM attribute + dashboard mode)
- PostHog feature-flag variants forwarded to Clarity as session-level custom tags
- Clarity sessions identified with PostHog's `distinct_id` + `session_id` for one-click cross-tool pivoting
- Optional: Clarity MCP installed for chat-driven heatmap queries

**Time**: 10–15 minutes of automated work, plus ~5 min user action for dashboard signup.

## Why Clarity alongside PostHog

PostHog has session replay built in. Adding Clarity isn't redundant because:

- **Clarity's heatmaps are sharper** — out-of-the-box click + scroll heatmaps overlay perfectly on the live page
- **Clarity surfaces frustration signals** — rage clicks, dead clicks, excessive scrolling are pre-categorized
- **Clarity is the strong free-fallback for replay capacity** — PostHog's free plan caps session replay at **5,000 desktop + 2,500 mobile recordings/month**; past that, replays drop until the next cycle. Clarity has **no recording cap** on its free tier (Microsoft funds it as part of Bing/Edge research), so it keeps recording when PostHog stops. For a site that's pre-product-market-fit and iterating, the unlimited-replay layer is the one that gives you full coverage on growth-spike days.
- **Distinct retention** — Clarity is 30 days playback / 13 months aggregate (different from PostHog's configurable up-to-7-years), useful for compliance separation

The cost: another vendor in the privacy policy, another DPA to sign, another set of masking rules to maintain. Worth it for marketing sites where heatmaps drive iteration.

**I skip Clarity if**:
- The site processes very sensitive content (medical records, financial transactions) and adding Clarity raises the privacy bar more than it's worth
- The user is solo-founder pre-launch with no traffic — defer until traffic justifies the setup time

Onboarding's privacy posture decision (Y/N on health-adjacent + EU visitors) informs whether the strict-gate variant or simple-gate variant applies. I read the saved config at the start of this stage.

## Prerequisites

- Stage 6 complete (PostHog wired with `loaded` callback; `src/_core/analytics/` scaffolded).
- I know from onboarding whether the site renders user-authored content (script editors, form fields beyond email, voice recordings, health data entry) — drives Variant A vs Variant B below.

## Automation surface I have available (verified 2026-05)

### Clarity MCP — `@microsoft/clarity-mcp-server`

API token paste only — no OAuth. Tools available:
- `query-analytics-dashboard` — scroll depth, engagement time, traffic by browser/OS/country/device
- `list-session-recordings` — fetch recording metadata (URLs, not video bytes)
- `query-documentation-resources` — search Clarity's own docs

**Rate limit**: 10 API requests per day per project, max 3 days of data per request, max 3 dimensions per request.

**Tools NOT in MCP**: heatmap export, masking config CRUD, custom-tag management, label management. Those are dashboard-only.

## Decision: simple gate vs strict gate

I pick the variant based on what the site does:

**Variant A — Simple gate (marketing sites without user-authored content)**: one env-only check. If `VITE_CLARITY_PROJECT_ID` is set, inject. This is the right pick for most marketing sites — newsletter signups + contact forms are the only PII surface, and Clarity's dashboard masking handles those automatically.

**Variant B — Strict gate (apps with user-authored content, voice recording, health data)**: three gates — env check, consent state, route scope. Used when the site has routes that render user-authored text, biometric data, or other users' content. Variant B requires Stage 8's consent state machine to exist; if Stage 8 hasn't been run yet, I scaffold the route-scope helper here and Stage 8 wires the consent gate.

Onboarding's `health_adjacent: true/false` and the user's site description tell me which variant to pick. I confirm with the user:

```
Based on what we've discussed, this is a [marketing site / app with user content].
I'll use the [simple / strict] Clarity gate. The difference:

  Simple gate (marketing): Clarity loads on every page if you've enabled it.
    Form inputs are auto-masked by Clarity's Balanced mode. Quick + minimal.

  Strict gate (app with user content): Clarity loads ONLY on routes that
    don't render user-authored content (script editors, voice tools, etc.
    are excluded). User must explicitly accept consent before Clarity
    loads. Triple-redundant masking on remaining sensitive elements.

Confirm: [simple / strict]?
```

## My execution sequence

### Step 1: I have the user create the Clarity project

```
Microsoft Clarity setup — about 3 minutes:

  1. Please open https://clarity.microsoft.com in your browser (I'll open it in a tab via Chrome MCP if connected; otherwise navigate directly).
  2. Sign in with a Microsoft account (or create one — they're free).
  3. Click "+ New Project" — name it after this site.
  4. After creation, you'll see a Setup page. The Project ID is at
     the top — looks like 10 random characters (e.g., "abc1234567").
     Copy it.
  5. Paste here, and I'll save it to a project-local secrets file.
```

When the user pastes:
1. I save to `<project>/.secrets/clarity-project-id.txt`
2. I confirm `.secrets/` is gitignored

### Step 2: I write the Clarity loader

#### For Variant A (simple gate)

I write `src/_core/analytics/clarity.ts`:

```ts
/**
 * Microsoft Clarity snippet loader (simple-gate variant for marketing sites).
 *
 * One gate: VITE_CLARITY_PROJECT_ID env var. If set, inject; if not, skip.
 *
 * On load, identifies the Clarity session with PostHog's distinct_id +
 * session_id so Clarity recordings can be correlated back to PostHog
 * sessions via the filter panel.
 */
import posthog from "posthog-js";

let loaded = false;

declare global {
  interface Window {
    clarity?: (...args: unknown[]) => void;
  }
}

export function initClarity(): void {
  if (loaded) return;
  if (typeof window === "undefined" || typeof document === "undefined") return;

  const projectId = import.meta.env.VITE_CLARITY_PROJECT_ID as string | undefined;
  if (!projectId) return;

  injectSnippet(projectId);
  loaded = true;

  // Correlate with PostHog identity
  const distinctId = posthog.get_distinct_id?.();
  const getSessionId = (posthog as unknown as { get_session_id?: () => string | undefined }).get_session_id;
  const sessionId = typeof getSessionId === "function" ? getSessionId() : undefined;

  if (window.clarity) {
    window.clarity("identify", distinctId ?? "anonymous", sessionId, undefined, undefined);
  }
}

function injectSnippet(projectId: string): void {
  const w = window as Window & { clarity?: ((...a: unknown[]) => void) & { q?: unknown[] } };
  if (!w.clarity) {
    const q: unknown[] = [];
    const fn = ((...args: unknown[]) => { q.push(args); }) as ((...a: unknown[]) => void) & { q?: unknown[] };
    fn.q = q;
    w.clarity = fn;
  }

  const script = document.createElement("script");
  script.async = true;
  script.src = `https://www.clarity.ms/tag/${projectId}`;
  const first = document.getElementsByTagName("script")[0];
  if (first?.parentNode) {
    first.parentNode.insertBefore(script, first);
  } else {
    document.head.appendChild(script);
  }
}
```

#### For Variant B (strict gate)

I write `src/_core/analytics/clarity.ts` with three gates:

```ts
/**
 * Microsoft Clarity snippet loader (strict-gate variant for apps with
 * user-authored content).
 *
 * Three gates (plus pre-condition guards):
 *   1: VITE_CLARITY_PROJECT_ID set
 *   2: getConsentState() === "accepted"
 *   3: isClarityInScope(currentPath) === true
 *
 * If any gate fails: don't inject the script. Excluded routes never
 * contact Clarity's CDN.
 */
import { isClarityInScope } from "./route-scope";
import { getConsentState } from "./consent";

let loaded = false;

declare global {
  interface Window {
    clarity?: (...args: unknown[]) => void;
  }
}

export function initClarityIfPermitted(currentPath: string): void {
  if (loaded) return;
  if (typeof window === "undefined" || typeof document === "undefined") return;

  const projectId = import.meta.env.VITE_CLARITY_PROJECT_ID as string | undefined;
  if (!projectId) return;
  if (getConsentState() !== "accepted") return;
  if (!isClarityInScope(currentPath)) return;

  injectSnippet(projectId);
  loaded = true;
}

export function onRouteChangeForClarity(path: string): void {
  if (typeof window === "undefined") return;
  const clarity = window.clarity;
  if (typeof clarity !== "function") return;
  if (isClarityInScope(path)) clarity("start");
  else clarity("stop");
}

// (injectSnippet same as Variant A)
```

I also write the route-scope helper at `src/_core/analytics/route-scope.ts`:

```ts
/**
 * Single source of truth for which routes Clarity is ALLOWED to run on.
 *
 * Add a route to CLARITY_EXCLUDED_ROUTES if it renders:
 * - User-authored content (script editors, comments, custom titles)
 * - Biometric capture (voice recording, face scans)
 * - Other users' content (admin views)
 * - Health-data entry
 */
export const CLARITY_EXCLUDED_ROUTES: readonly string[] = [
  // I populate this list from the user's app routes that render
  // user-authored content. Examples for typical apps:
  // "/create",       // script editor
  // "/record",       // biometric voice capture
  // "/library",      // user's own content
  // "/admin",        // admin views (covers all /admin/*)
];

export function isClarityInScope(path: string): boolean {
  const normalized = normalizePath(path);
  for (const excluded of CLARITY_EXCLUDED_ROUTES) {
    if (normalized === excluded) return false;
    if (normalized.startsWith(excluded + "/")) return false;
  }
  return true;
}

function normalizePath(path: string): string {
  const withoutQuery = path.split(/[?#]/, 1)[0] ?? "";
  const withoutTrailing = withoutQuery.length > 1 && withoutQuery.endsWith("/")
    ? withoutQuery.slice(0, -1)
    : withoutQuery;
  return withoutTrailing || "/";
}
```

I ask the user which routes render user-authored content during this stage, then populate `CLARITY_EXCLUDED_ROUTES` accordingly.

**Note**: Variant B's `clarity.ts` imports `getConsentState` from `./consent`. If Stage 8 hasn't run yet, I create a temporary stub that always returns `"unknown"` (so Clarity never loads), and Stage 8 replaces it with the real consent state machine. This avoids forcing Stage 8 to run before Stage 7.

### Step 3: I write the flag-bridge

Every PostHog feature-flag variant becomes a Clarity custom tag. When the user runs a PostHog experiment, Clarity recordings are auto-tagged with the variant string — they can filter recordings by variant in Clarity's UI without per-experiment code.

I create `src/_core/analytics/flag-bridge.ts`:

```ts
import posthog from "posthog-js";

type FlagValue = string | boolean | undefined;
type FlagVariants = Record<string, FlagValue>;

export function forwardActiveFlagsToClarity(flagVariants: FlagVariants): void {
  if (typeof window === "undefined") return;
  const clarity = window.clarity;
  if (typeof clarity !== "function") return;
  for (const [key, value] of Object.entries(flagVariants)) {
    if (value === undefined) continue;
    clarity("set", key, String(value));
  }
}

/**
 * Subscribe once to PostHog's flag-change callback and forward the active
 * variants to Clarity whenever they change. Returns an unsubscribe function.
 */
export function initFlagBridge(): () => void {
  if (!posthog.__loaded) return () => {};
  const unsubscribe = posthog.onFeatureFlags((flagKeys, flagValues) => {
    const variants: FlagVariants = {};
    for (const key of flagKeys) {
      variants[key] = (flagValues as Record<string, FlagValue>)[key];
    }
    forwardActiveFlagsToClarity(variants);
  });
  return unsubscribe;
}
```

### Step 4: I extend deploy-tag for Clarity-side registration

Stage 6 registered deploy identifiers as PostHog super-properties. Now I mirror to Clarity (where they become custom tags). I update `src/_core/analytics/deploy-tag.ts`:

```ts
// Add these exports:

export function registerDeployTagOnClarity(): void {
  if (typeof window === "undefined") return;
  const clarity = window.clarity;
  if (typeof clarity !== "function") return;

  const id = import.meta.env.VITE_DEPLOY_ID as string | undefined;
  const variant = import.meta.env.VITE_DEPLOY_VARIANT as string | undefined;
  const sha = import.meta.env.VITE_DEPLOY_VARIANT_SHA as string | undefined;
  if (id) clarity("set", "deploy_id", id);
  if (variant) clarity("set", "deploy_variant", variant);
  if (sha) clarity("set", "deploy_variant_sha", sha);
}
```

The same property name on PostHog AND Clarity means dashboards on both tools filter by the same dimension — `deploy_variant=variant-X` matches in both.

### Step 5: I wire `initAnalytics()` in `src/_core/analytics/index.ts`

```ts
import { initClarity } from "./clarity";          // Variant A (or initClarityIfPermitted for Variant B)
import { initFlagBridge } from "./flag-bridge";
import { registerDeployTagOnClarity } from "./deploy-tag";

export { initClarity, initClarityIfPermitted } from "./clarity";
export { forwardActiveFlagsToClarity, initFlagBridge } from "./flag-bridge";
export {
  registerDeployTagOnPostHog,
  registerDeployTagOnClarity,
} from "./deploy-tag";
export { registerCustomUtmsOnPostHog } from "./utm-capture";
export { registerAffiliateCodeOnPostHog } from "./affiliate-capture";
export { track, extractEmailDomain } from "./track";

/**
 * Post-capture analytics initialization.
 * Call AFTER the first posthog.capture() from PostHog's loaded callback.
 */
export function initAnalytics(): void {
  initClarity();           // Variant A
  // OR for Variant B: initClarityIfPermitted(window.location.pathname);
  initFlagBridge();
  registerDeployTagOnClarity();
}
```

I substitute the right `initClarity` call based on the variant chosen in the decision above.

### Step 6: I configure dashboard masking

Three masking layers:
1. **CSS class**: any DOM element with `class="clarity-mask"` gets masked
2. **DOM attribute**: any element with `data-clarity-mask="True"` gets masked (Clarity's native fallback)
3. **Dashboard Balanced mode**: form inputs auto-masked + images auto-masked

Layer 3 needs the user to configure in the Clarity dashboard. I tell them:

```
Last manual step in Clarity — configure masking (about 1 minute):

  1. Open Clarity dashboard → Settings → Masking
  2. Masking mode → select "Balanced"
  3. Mask by element → click "+ Add element"
  4. Paste this CSS selector: .clarity-mask
  5. Click Save

Tell me when done; I'll verify by checking the next session capture.
```

### Step 7 (Variant B only): Sensitive-content wrapper component

For apps that render user-authored content alongside non-sensitive UI on the same route, I create `src/components/SensitiveText.tsx`:

```tsx
interface SensitiveTextProps {
  children: React.ReactNode;
  as?: "span" | "div" | "p";
}

export function SensitiveText({ children, as: Tag = "span" }: SensitiveTextProps) {
  return (
    <Tag
      className="clarity-mask ph-no-capture"
      data-clarity-mask="True"
    >
      {children}
    </Tag>
  );
}
```

Use anywhere user content renders:

```tsx
<h1>Welcome, <SensitiveText>{user.displayName}</SensitiveText>!</h1>
```

The `ph-no-capture` class is PostHog's native masking — same triple-redundant pattern.

### Step 8: I set Railway env var + redeploy

```
railway variable set VITE_CLARITY_PROJECT_ID="$(cat .secrets/clarity-project-id.txt)"
railway up
```

### Step 9 (optional): I install Clarity MCP

If the user wants chat-driven heatmap queries:

I tell them:

```
Quick option — install the Clarity MCP for chat-driven heatmap queries
(about 2 minutes):

  1. In Clarity dashboard → Settings → Data Export → "Generate new API token"
  2. Copy the token (it's shown once)
  3. Paste here, and I'll add to Claude Code's MCP config + restart

Skip this if you don't see yourself querying Clarity from chat. The
Clarity dashboard works fine without the MCP.
```

When the user pastes the token, I:
1. Save to `<project>/.secrets/clarity-api-token.txt`
2. Update Claude Code's MCP config (in `~/.claude.json` or via the MCP setup helper) to include:

```json
{
  "mcpServers": {
    "clarity": {
      "command": "npx",
      "args": ["@microsoft/clarity-mcp-server", "--clarity_api_token=${CLARITY_API_TOKEN}"],
      "env": { "CLARITY_API_TOKEN": "<the-token>" }
    }
  }
}
```

3. Verify via `mcp__clarity__query-documentation-resources` after restart

**Rate limit reminder**: 10 requests per day per project. I don't loop or batch-poll.

### Step 10: I update the privacy policy draft

I add Clarity as a sub-processor (Stage 8 owns the policy file):

```markdown
## Sub-processors
- Microsoft Clarity (heatmaps + session replay) — collects user interaction
  data on non-sensitive routes. Retains playback for 30 days; aggregate data
  for 13 months. Privacy: https://clarity.microsoft.com/privacy
```

For health-adjacent products, I add a sentence to the `/consumer-health-data` policy page (also Stage 8) distinguishing PostHog's retention (configurable up to 7 years) from Clarity's (30 days playback / 13 months aggregate).

## Cross-tool correlation pattern

PostHog has NO first-party Clarity integration ("View in Clarity" button doesn't exist in PostHog dashboards). Cross-referencing happens via shared identifiers:

| From → To | How |
|---|---|
| PostHog event → Clarity recording | Copy `distinct_id` + `session_id` from PostHog event; paste into Clarity's filter panel |
| Clarity recording → PostHog session | Copy `distinct_id` from Clarity's session metadata (it's there because of Step 2's `clarity("identify", ...)` call); search PostHog Persons by that ID |

The shared `deploy_variant` tag lets the user filter both tools by iteration without correlating individual sessions.

## Verification (autonomous)

I run these myself before declaring Stage 7 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Env var set on Railway | `railway variable list` filtered on `VITE_CLARITY_*` | Project ID present |
| Clarity script injected | Curl deployed `/index.html` and check for `clarity.ms/tag/<id>` | Script tag present |
| Clarity tracking active | `preview_network` filter on `clarity.ms` | 200 responses, both tag.js and tracker requests |
| (Variant B) Clarity NOT on excluded routes | `preview_network` on each excluded route | Zero `clarity.ms` requests |
| (Variant B) Masking works | Inspect form via `preview_snapshot` | `clarity-mask` class present on user-content elements |
| MCP works (if installed) | `mcp__clarity__query-documentation-resources` with sample query | Returns results |

I report:

```
✅ Stage 7 complete:
   • Clarity project active (ID: <user-clarity-id>)
   • Variant: simple gate (marketing site, no user-authored content)
   • Clarity snippet loads on all pages; no excluded routes
   • Dashboard masking: Balanced mode + .clarity-mask selector configured
   • PostHog → Clarity flag bridge wired
   • Cross-tool identity: clarity("identify", posthog.distinct_id, session_id)
   • MCP: skipped (you said maybe later)
   • Privacy policy draft updated to list Clarity as sub-processor

Ready for Stage 8 (privacy + consent banner)?
```

## Common gotchas I handle automatically

- **`maskAllInputs` masks too aggressively**: legit fields like dropdown selects come through as `***`. If the user needs a select unmasked, I add `data-ph-no-mask="false"` to the specific element.
- **Custom tags not appearing**: tags are session-scoped, registered AFTER snippet loads. I order calls so `clarity("set", ...)` happens after Clarity init.
- **Mask config didn't propagate**: Clarity's masking is server-side. Existing recordings keep the OLD mask state; only new recordings get the new state. I tell the user this so they're not confused if old recordings show unmasked content briefly after a config change.
- **MCP rate limit hit**: 10/day per project. If exhausted, I wait 24h or fall back to dashboard.

## Self-research instruction

Before running this stage, I web-search:
- `Clarity MCP server <year>` — capability list expansion
- `Clarity API documentation <year>` — new endpoints
- `Clarity privacy controls <year>` — masking model changes

## Outputs

After Stage 7:
- Clarity snippet loaded with appropriate gating (Variant A or B based on site type)
- Triple-redundant masking active (CSS class + DOM attr + dashboard mode)
- PostHog → Clarity flag bridge wired
- Cross-tool identity correlation via shared IDs
- (Optional) Clarity MCP available in chat
- Privacy policy draft updated

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Verified Clarity MCP `@microsoft/clarity-mcp-server` is API-token-paste only (no OAuth). Three tools. Rate limit 10/day. |
| 2026-05-08 | Reframed for non-coder audience: replaced "Use this template" / "Create file X" with first-person Claude voice. Replaced literal author Clarity project ID with `<user-clarity-id>` placeholder. Variant A vs B decision now driven by onboarding's `health_adjacent` field with a confirmation prompt — user doesn't need to evaluate the technical difference. Added explicit note that Variant B's `clarity.ts` depends on Stage 8's consent state machine; if Stage 8 hasn't run yet, I create a temporary stub that always returns "unknown" so the import resolves and Stage 8 replaces the stub with the real implementation. Verification reframed to autonomous Preview MCP checks. |
