# Stage 0: Source-Platform Discovery

**Last verified: 2026-05-08.** Re-research vendor automation surface (Railway / PostHog / Clarity / Resend MCPs) before starting Stage 4 if this date is older than 60 days.

> **🤖 Re-grounding (re-read at start of stage)**: This is a Claude-driven stage. I (Claude) run all the analysis tools, generate the punchlist, and report findings. The user's job is to review what I found and confirm a few decisions. The user does NOT run greps, write the punchlist, or read raw command output unless they explicitly want to.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 0 needs:
- `origin.platform` — set during onboarding; I use it to bias what I look for. If missing, I detect by inspecting the source artifact itself (`package.json`, `vite.config.ts`, file structure) and update the config.
- `origin.artifact_type` and `origin.artifact_path` — I need at least one. If both missing, I prompt for a zip path or GitHub URL before proceeding.

## What I'm doing in this stage

Surfacing every dependency the source code has on its origin platform — forms, images, CDNs, AI-generated content, auth, DB, hardcoded URLs, dependency bloat — and producing a `migration-punchlist.md` that lists each finding with its planned migration target. This document feeds Stage 2 (platform migration) and serves as a launch-readiness check at Stage 12.

**Time**: 5–15 minutes for me to run the analysis. Then a short user-confirmation moment.

## Why this stage matters (plain-language)

Skipping this step is the most common source of post-launch incidents. Real failure modes I want to catch:

- **Form submissions silently routing to a third party's dashboard the user can't access** — the user discovers it after dozens of newsletter signups vanished into someone else's tool.
- **AI-generated team members or testimonials presented as real people** — FTC §5 / Endorsement Guides (16 CFR Part 255) violation. EU equivalents apply too. This is a launch blocker, not a polish item.
- **Hardcoded asset CDNs** that 404 the day the source platform goes down or rotates URLs.
- **Form data attributes left in production** (e.g., `data-readdy-form`) that leak the origin platform to anyone inspecting the DOM.
- **Vite plugin imports referencing dev-only platform tooling** that crash production builds when the dev dependency isn't installed.
- **Hardcoded production domains** scattered through code that all need to change at DNS migration time.

A 5-minute analysis upfront prevents weeks of "why is X broken in prod?" mystery later.

## Prerequisites

- Onboarding complete. I've already gathered the high-level decisions (origin platform, brand, EU visitors, health-adjacent, etc.) and saved them to `<project>/.skill-config.json`. I read this file at the start of Stage 0 so I know what to look for.
- The source code is accessible to me. If the user gave me a zip, I've extracted it. If a GitHub repo, I've cloned it. If they're starting fresh, this stage is mostly skipped (no platform code to discover).

## Process — three steps

### Step 1: I run the static analysis

I scan the source code for known patterns. **The user doesn't need to run anything**. I use my Grep tool (which works on Windows, macOS, and Linux identically) to look for:

#### Forms — full structural extraction (NOT just "this site has forms")

Forms are the highest-value Stage 0 output for Stage 5. I do NOT just note "there's a newsletter form somewhere"; I extract the complete structure of every form so Stage 5 can generate matching API endpoints, DB columns, and ESP-segment mappings without re-asking the user.

**For every form on the site, I extract** (via reading JSX/HTML/component source, NOT by asking the user):

| Property | Source | Example |
|---|---|---|
| `name` | Heuristic from form context — heading nearby, button label, route URL, surrounding copy | `"newsletter_signup"`, `"contact"`, `"request_demo"`, `"event_rsvp"` |
| `route` | The page path the form lives on | `"/"`, `"/contact"`, `"/request-demo"`, `"/events/oct-15"` |
| `current_action` | `<form action="...">` value, or the URL inside the JSX submit handler's `fetch()` / `axios()` call, or the platform-specific data attr (`data-readdy-form="X"` etc.) | `"https://api.formspark.io/abc123"`, `"https://webflow.com/api/v1/form/..."`, `null` (if no current handler) |
| `fields` | Every `<input>`, `<select>`, `<textarea>` inside the form — extracted as `{name, type, required, label, placeholder, validation_pattern, options}` | See example below |
| `submit_button_label` | The CTA text on the submit button | `"Subscribe"`, `"Send Message"`, `"Get a Quote"` |
| `success_state` | What happens after submit — toast text, redirect URL, modal content | `"Thanks for subscribing! Check your inbox."` |
| `inferred_purpose` | Best-guess of what the form is FOR (drives the suggested ESP-segment / CRM-stage / notification routing in Stage 5) | `"lead_capture"`, `"customer_support"`, `"newsletter_optin"`, `"event_signup"`, `"sales_inquiry"`, `"feedback"` |

**Pattern hunt for the form elements themselves**:
- `<form>` tags — easy on plain HTML, slightly harder in JSX where forms may be inside dynamically-rendered components
- React Hook Form / Formik / Vee-Validate / native React state forms — I trace the submit handler to find the actual submission target
- Hardcoded `action="https://..."` URLs in HTML / JSX
- `fetch(...)` or `axios(...)` calls to platform form APIs
- Platform-specific data attributes: `data-readdy-form`, `data-formspark`, `data-formsubmit`, `data-netlify`, Webflow's `wf-form` / `w-form-` classes
- Library-specific patterns: `useForm()` from react-hook-form (the `register` calls list every field), Formik's `<Field name="...">`, `<input {...props}>` in custom hooks

**Output: I write `forms-manifest.md` to repo root** alongside `migration-punchlist.md`. One section per form, structured so Stage 5 can mechanically generate the matching API endpoint, validation schema, DB columns, and ESP/CRM mapping confirmation prompts.

Example entry:

```markdown
## Form: newsletter_signup

- **Route**: /
- **Current action**: data-readdy-form="newsletter" (Readdy default; no real handler)
- **Submit button**: "Subscribe"
- **Success state**: toast "Thanks! Check your inbox to confirm."
- **Inferred purpose**: newsletter_optin

### Fields
- email (input type=email, required, placeholder="you@example.com")

### Stage 5 will generate
- POST /api/newsletter-signup endpoint
- Validation: { email: required + RFC-5322 }
- DB table: form_submissions (default; user can opt out)
- ESP target: ask user at Stage 5 (default: Resend Audiences "All Subscribers")
- PostHog events: newsletter_signup_started + _completed (auto-included in Stage 11 funnels)

## Form: request_demo

- **Route**: /demo
- **Current action**: <form action="https://hsforms.com/abc123"> (HubSpot Forms)
- **Submit button**: "Request a Demo"
- **Success state**: redirect to /thanks
- **Inferred purpose**: sales_inquiry

### Fields
- full_name (input type=text, required)
- work_email (input type=email, required)
- company (input type=text, required)
- company_size (select, options=[1-10, 11-50, 51-200, 201-1000, 1000+])
- use_case (textarea, required, max=2000)

### Stage 5 will generate
- POST /api/request-demo endpoint
- Validation: { full_name, work_email, company, company_size, use_case } per types above
- DB table: form_submissions (default)
- ESP target: ask user at Stage 5 (HubSpot already detected — likely route to HubSpot CRM)
- PostHog events: request_demo_started + _completed
```

**The principle**: Stage 0 discovery is authoritative. Stage 5 confirms with the user where things route (which ESP segment, which CRM stage) but does NOT re-ask what fields the form has — that question was already answered by reading the code. The user's time is preserved for the questions only they can answer.

If a form's purpose is genuinely ambiguous from the code alone (e.g., a generic `<form>` with one `email` field could be a newsletter or a "forgot password"), I list both possibilities in the manifest entry and ask the user once at Stage 5 to resolve.

#### Images and external assets

Pattern hunt:
- Common platform CDN domains (`*.readdy.*`, `*.framer.*`, `*.webflow.*`, `*.wix.com`, `*.squarespace.*`, `*.cdninstagram.*`)
- AI-generated dynamic image URLs (e.g., `?query=...&seq=...` on `/api/search-image` endpoints — Readdy, v0, Lovable, Bolt patterns)
- Static logos/icons hosted on `static.<platform>.com`
- Font URLs on platform CDNs

Each unique URL goes into the punchlist as a 🟡 pre-cutover item with the planned destination (`public/images/migrated/<name>` for self-hosting, or replacement-with-real-photography note).

#### Auth / OAuth wiring

Pattern hunt: imports of `firebase`, `@supabase/supabase-js`, `@auth0`, `@clerk`, `magic-sdk`, `next-auth`, etc.

Findings inform whether Stage 9 (membership) or a separate auth setup is needed.

#### Existing tracker scripts

Pattern hunt: `gtag`, `fbq`, `hotjar`, `mixpanel`, `amplitude`, `plausible`, `fathom`, `matomo`.

Each goes into the punchlist as a decision point: keep alongside PostHog, or remove? (Most projects benefit from a single source of truth.)

#### Hardcoded production domains

Pattern hunt: any absolute URL containing the user's brand or domain. (I read the brand/domain from the saved config so I know what to search for — I'm not searching for a literal placeholder.)

These should always be sourced from `SITE_URL` / `APP_URL` env vars rather than hardcoded. Each hardcoded reference goes into the punchlist as a 🟡 item.

#### Dependency bloat

I parse `package.json`, then for each dependency, search the source for actual imports. Anything with zero imports is a candidate for removal.

Common AI-generator-scaffolded-but-unused dependencies I look for:
- `firebase`, `@supabase/supabase-js`, `@stripe/react-stripe-js`, `recharts`, `lucide-react`, sometimes `serve` (replaced by our Express server in Stage 3)

Removing these typically trims 200–400 MB from `node_modules` and roughly halves Railway build time.

#### Vite / build config residue

Pattern hunt:
- Commented-out platform-specific Vite plugins (e.g., `// import { readdyJsxRuntimeProxyPlugin }` — dev-only on the platform, crashes in prod if uncommented)
- Platform-specific `define{}` entries (e.g., `__READDY_PROJECT_ID__`, `__WEBFLOW_*`, `__FRAMER_*`)
- Platform-specific declarations in `vite-env.d.ts`

#### `.gitignore` coverage

I check whether `.gitignore` exists and covers the essentials (`node_modules`, `.env`, `dist`/`out`, OS files). Stage 1 will write a complete one; Stage 0 just notes what's missing.

### Step 2: I generate the punchlist

I write `migration-punchlist.md` at the repo root. I substitute the user's actual brand, domain, source platform, and findings. The structure:

```markdown
# Migration Punchlist — Maya's Consulting Marketing Site

Generated 2026-05-08 from Stage 0 discovery of Webflow export.
Origin platform: Webflow

## 🔴 Launch blockers
[Items I won't let the user ship without resolving — typically FTC content
concerns, broken form endpoints, missing email infrastructure.]

- [ ] AI-generated team members in src/components/TeamSection.tsx (lines 12–48)
      — FTC concern. Either replace with real people (with written permission)
      or remove the section entirely.
- [ ] Form submissions hardcoded to https://webflow.com/forms/abc123 in
      ContactForm.tsx:57 — submissions land in Webflow's dashboard, currently
      inaccessible. I'll replace this with a /api/contact route in Stage 5.

## 🟡 Pre-cutover migration items
[Items I'll address in Stage 2 before Railway deployment.]

- [ ] Images: 9 unique URLs from framerusercontent.com — I'll download to
      public/images/migrated/ in Stage 2.
- [ ] Logo: static.platform.com/logo.png referenced in Navbar.tsx and
      Footer.tsx — I'll vendor to public/logo/brand.png.
- [ ] Vite config: commented-out readdyJsxRuntimeProxyPlugin import on line 4.
      I'll remove in Stage 2.
- [ ] Form data attributes: data-readdy-form on 2 form elements.
      I'll strip in Stage 2 once new endpoints are wired.
- [ ] Hardcoded domain references: 3 occurrences of "mayasconsulting.com"
      that should source from SITE_URL env var.

## 🟢 Post-launch optional
[Polish that's safe to defer past launch.]

- [ ] Bundle dep cleanup: 5 packages with 0 imports. Removal saves ~200MB
      from node_modules and ~50% Railway build time.
- [ ] Real testimonials from real users (with written permission).
- [ ] sitemap.xml + robots.txt.

## Configurations needed (output of Stage 0 → input for later stages)

### Railway env vars (locked at time of first build)
[I'll set these autonomously in Stages 4–9 as they become relevant.
Listed here so the user can see the full set planned for their build.]

- VITE_POSTHOG_KEY (Stage 6)
- VITE_POSTHOG_HOST (Stage 6 — same-origin proxy path I'll choose)
- VITE_CLARITY_PROJECT_ID (Stage 7, if Clarity is in scope)
- RESEND_API_KEY (Stage 5, server-side only — never VITE_-prefixed)
- EMAIL_FROM, EMAIL_NOTIFY_TO (Stage 5)
- SITE_URL, APP_URL (Stage 4)
- PORT=3000 (Stage 4)
- RAILWAY_GIT_COMMIT_SHA (auto-set by Railway; used by Stage 6 for deploy_id)
- RAILWAY_CHECKOUT_DEPTH=0 (Stage 4 — avoids shallow-clone bug for variant labels)
- STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET (Stage 9, if Stripe — the default)
- WHOP_WEBHOOK_SECRET, WHOP_API_KEY (Stage 9, if Whop)
- POSTHOG_PROJECT_API_KEY (Stage 9, server-side capture from webhook handler)

### DNS records needed
- mayasconsulting.com → CNAME → Railway-provided target (Stage 4)
- DKIM / SPF / DMARC for the sending domain (Stage 5)

### External dashboards I'll have the user update during cutover
- Google Cloud Console OAuth callback URLs (if migrating an auth-using app)
- Stripe webhook endpoint URL (if Stripe — the default for Stage 9)
- Whop webhook URL (if Whop)
- Any vendor pointing at an old domain

## Privacy posture (carried over from onboarding)

- **Posture**: standard (banner shows in EU only, opt-out elsewhere)
- **EU visitors**: yes
- **Health-adjacent**: no
- **CHD policy required**: no
- **Banner version sentinel**: 2026-05-08-v1 (bump in Stage 8 if copy changes)

## Decisions captured

- 2026-05-08 — Email provider: Resend (sending from mayasconsulting.com).
- 2026-05-08 — Membership platform: none (marketing site only).
- 2026-05-08 — Affiliate program: deferred to "later".
- 2026-05-08 — Variant system: enabled, tracking 5 pages.
```

The user doesn't need to write any of this. I generate it from my analysis findings + their saved config.

### Step 3: I show the user the punchlist and ask for confirmation

I summarize in plain language:

```
Discovery complete. Here's what I found in your Webflow export:

🔴 LAUNCH BLOCKERS (must resolve before public launch):
  • TeamSection.tsx has 4 AI-generated faces presented as real team members.
    Webflow's template generated these. I recommend removing the section
    entirely (you're a solo founder; no team to display) OR replacing with
    a real headshot of yourself plus an honest "Founder" bio.

🟡 PRE-CUTOVER MIGRATION (I'll handle in Stage 2):
  • 9 images on framerusercontent.com — I'll download these to your repo
  • 1 logo on static.platform.com — I'll vendor it
  • Form on ContactForm.tsx points at Webflow's dashboard — I'll redirect
    to a /api/contact route in Stage 5
  • Some commented-out platform plugins in vite.config.ts — I'll clean

🟢 POST-LAUNCH OPTIONAL:
  • 5 unused npm packages I can remove later for faster builds
  • sitemap.xml / robots.txt

Decisions to confirm:
  1. The TeamSection — remove entirely, or replace with a real founder bio?
  2. Are you OK with me migrating the 9 images to your repo (vs. picking
     new photos)?

Anything you'd add or change?
```

I wait for the user to acknowledge each launch blocker decision and any non-default migration choices before advancing to Stage 1.

## Source-stack detection (AIO path)

For the AIO path (Astro target), Stage 1 needs to know what the source-stack shape is so it can pick the right migration plan. I detect this during Stage 0 and save to `.skill-config.json` as `origin.source_stack`:

| Detection signal | `origin.source_stack` value |
|---|---|
| `package.json` includes `react-router-dom` | `"react-router-spa"` |
| `package.json` includes `react` (no router) | `"react-no-router"` |
| `package.json` includes `next` | `"next-pages"` or `"next-app-router"` (check `app/` vs `pages/` dir) |
| No `package.json`, source is folder of `.html` + `.css` files | `"static-html-css"` |
| No `package.json`, source is folder of `.html` files only | `"plain-html-multi-page"` |
| `package.json` includes `astro` already | `"astro-existing"` (rare — source was already Astro; Stage 1 mostly no-ops) |

I run the detection autonomously and save the result. Stage 1 reads it to pick the migration plan from the matrix documented in [stage-1-scaffolding.md](stage-1-scaffolding.md#source-stack--astro-target-migration-matrix). If the user picked the Vite-only path (`aio.content_hub = false`), this detection is informational but doesn't drive migration — Vite is a more flexible target that accepts most source shapes with minimal restructuring.

## Verification (gate before Stage 1)

- [ ] `migration-punchlist.md` exists at repo root with my findings populated
- [ ] Every external URL / dependency I found is accounted for in the punchlist
- [ ] `origin.source_stack` captured in `.skill-config.json` (AIO path only)
- [ ] User has acknowledged each 🔴 launch blocker with a resolution decision
- [ ] User has approved the 🟡 pre-cutover migration approach
- [ ] User has explicitly confirmed: "go to Stage 1"

If any item is incomplete, I stop and resolve before advancing. Stage 1 assumes Stage 0's outputs.

## Common discovery anti-patterns I refuse

- **"This site is simple, skip the analysis"** — Bubble exports especially have hidden dependencies (workflow data, embedded scripts in HTML attributes). I always run the full analysis.
- **"I'll figure out the email provider later"** — domain verification can take hours; I want to start the DNS clock during onboarding/Stage 5.
- **"I'll deal with the AI-generated faces in v2"** — those are launch blockers, not polish items. Disclaimers don't cure FTC violations. The user accepts the legal exposure or resolves before launch.
- **"Use the strictest privacy posture, just to be safe"** — defensible if the user has explicit legal advice. Most projects benefit from the standard posture (banner shows in EU only). The onboarding decision tree already picked the right one.
- **"The membership platform's webhook tells you about affiliates"** — verified false across major payment platforms (Stripe, Whop, Lemonsqueezy as of 2026-05). Webhook payloads typically carry buyer + plan info, not the affiliate who referred them. Affiliate attribution must be DB-mediated against a row written when the visitor first arrived with `?ref=...`. Stage 10 handles this.

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. |
| 2026-05-08 | Reframed for non-coder audience: removed duplicated questionnaire (deferred to onboarding); replaced "run these greps" with "I run the analysis"; replaced "edit the punchlist" with "I generate the punchlist; user reviews"; stripped personal-attribution and brand-specific examples; replaced trademarked methodology references with generic terminology; cleaned references to author-specific memory files. Static-analysis section reframed from raw bash commands to "patterns I hunt for" — works equally on Windows / macOS / Linux. |
