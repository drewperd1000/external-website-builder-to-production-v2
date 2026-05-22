# Examples Directory

This directory holds **canonical, sanitized copies** of the trickiest code patterns from the skill — the ones with non-obvious gotchas that benefit from being copied verbatim rather than re-typed from the stage docs' markdown blocks.

For everything else, the stage files (`stage-N-*.md`) ARE the canonical source. This README points at where each pattern lives.

## What's here vs in the stage docs

| Pattern | Canonical location | Why |
|---|---|---|
| `posthog-proxy.js` | `stage-6-posthog.md` Step 5 (full content) | Load-bearing; copy from there. |
| `copy-posthog-static.mjs` | `stage-6-posthog.md` Step 4 (full content) | Same. |
| `server.js` | `stage-3-express-server.md` Step 2 (full template) | Full template inline; adapt for your project. |
| `current-variants.json` | `stage-11-variants-and-dashboards.md` Step 1 (schema + example) | Schema is small, fully shown there. |
| `consent.ts` (anonymous) | `stage-8-privacy-consent.md` Step 2 (full content) | Adapt the storage key. |
| `consent.ts` (authenticated, version-aware) | `stage-8-privacy-consent.md` Step 3 (full content) | For apps with login. |
| `geo.ts` (Cloudflare cdn-cgi/trace) | `stage-8-privacy-consent.md` Step 1 (full content) | |
| `flag-bridge.ts` | `stage-7-clarity.md` Step 3 (full content) | |
| `deploy-tag.ts` | `stage-6-posthog.md` Step 8 + `stage-7-clarity.md` Step 4 + `stage-11-variants-and-dashboards.md` Step 5 | Spread across stages because it grows incrementally. |
| `utm-capture.ts` | `stage-6-posthog.md` Step 8 (full content) | |
| `affiliate-capture.ts` | `stage-10-affiliate-optional.md` Step 1 (full content) | |
| `affiliate-link.ts` | `stage-10-affiliate-optional.md` Step 2 (full content) | |
| `clarity.ts` (marketing) | `stage-7-clarity.md` Step 2 Variant A | |
| `clarity.ts` (strict-gate, app) | `stage-7-clarity.md` Step 2 Variant B | |
| `route-scope.ts` | `stage-7-clarity.md` Step 2 Variant B helper | |
| `webhook-stripe.js` (default) / `webhook-whop.js` (alternative) | `stage-9-membership-optional.md` Step 4 + alternative-platforms section | Stripe is the default for Stage 9; Whop is the recommendation for content/community/course platforms. |
| `posthog-server.js` | `stage-9-membership-optional.md` Step 5 (full content) | |
| `email.js` (transport abstraction) | `stage-5-email-resend.md` Step 4 (full content) | |
| `vite.config.ts` (variant + define injection) | `stage-6-posthog.md` Step 7 + `stage-11-variants-and-dashboards.md` Step 2 | |
| `main.tsx` (loaded callback orchestration) | `stage-6-posthog.md` Step 9 (full content) | |

## Why no separate example files

This skill values **single-source-of-truth**. If a code block lives in two places, those two places will drift over time as the skill evolves (each stage's "Last verified" date marches independently). By keeping the code blocks ONLY in the stage docs, the dated changelog at the bottom of each file is the authoritative history.

Future-Claude reads a stage, copies the code block from there, adapts brand-specific names, ships it.

## When to add files here

If a future revision finds a code pattern is too long to live inline (>200 lines, repeated identically across stages, etc.), promote it to this directory and update the table above to point at it. Until then, stage docs are authoritative.
