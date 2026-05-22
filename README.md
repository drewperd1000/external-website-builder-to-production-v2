# external-website-builder-to-production-v2

A Claude Code skill that takes a website built somewhere else — Webflow, Framer, Wix Studio, Readdy, Bubble, an AI generator, or custom code — and ships it to fully-instrumented production on Railway with analytics, email, consent banner, **AI-search-optimized content hub infrastructure**, **schema markup**, **Google Search Console automation**, and **deliverability-gated forms**.

> **v2 ships everything [v1](../external-website-builder-to-production/) ships, plus the layers that make the site discoverable by ChatGPT Search, Perplexity, Google AI Overview, and Claude with web search.** v1 remains a valid skill for sites where AI-search isn't a goal. See [CHANGELOG.md](CHANGELOG.md) for the 10 deltas v2 owns.

> **Who this is for**: founders, marketers, and developers who want a real production stack that AI assistants will cite — without having to learn the operational stack themselves. The skill is Claude-driven — Claude executes the technical work, you make the business decisions.

## What you get when this skill runs end-to-end

- **A Railway-deployed Astro 5 + React + Express site** at your custom domain (for content-hub sites — see the AIO bullet below for what makes this the new default). For tightly-scoped paid-traffic landers or internal admin tools without organic ambitions, the skill instead scaffolds Vite + React + Express — same Railway + analytics + email pipeline either way.
- **HTTP Basic Auth staging service** at `<name>-staging.up.railway.app` — a second Railway service pointing at the same git branch with `BASIC_AUTH_USER` + `BASIC_AUTH_PASS` set ONLY on staging. The Express middleware no-ops in production. No Cloudflare DNS/Access work required (no managed-Access subscription either) — Railway's default URL + Basic Auth is the whole staging surface.
- **Cloudflare MCP integration** for easily connecting the correct domain settings, etc. You can opt for using your own Domain registrar DNS service, but you'll need to do more manual work.
- **PostHog analytics** with
  - **same-origin *internal* reverse proxy** - this recovers **10–25% of events** that ad-blockers would otherwise drop (!!! 🤯)
  - `loaded`-callback init pattern (no race), and build-time deploy + both global- and page-variant identifiers attached to every event. This enables A/B/n testing of every page individually in addition to global website versions.
  - **Granular Tracking** by every CTA text, Button, etc that you want to understand the performance of
  - **Prebuilt variant-tracking dashboards** for sequential CRO iteration with 6+ pre-built PostHog dashboards
  - **Easy creation of additional custom dashboards in PostHog:** detailed, unlimited dashboards for measuring every metric you might be interested in (and filtering out the metrics you're not)
  - **this altogether gives you MORE power than the $100+ Mouseflow subscription tier... for free**
- **Microsoft Clarity** heatmaps + session replay alongside PostHog (the unlimited free-fallback for replay capacity past PostHog's free-tier cap of 5k desktop / 2.5k mobile sessions per month). Clarity offers richer UI/UX experience for visual analysis.
- **AIO (AI-search optimization) content-hub infrastructure** so the marketing site is discoverable + citable by ChatGPT Search, Perplexity, Google AI Overview, Claude with web search, etc. — citation surface that pure-Vite sites without schema markup are leaving on the table. With:
  - **8 schema components** emitting valid JSON-LD on every page (`Organization`, `WebSite`, `Person`, `Article`, `FAQPage`, `HowTo`, `Product`, `BreadcrumbList`), with **reciprocal `sameAs` declarations** across all your surfaces (marketing site + app + parent org) so AI assistants recognize you as one entity. Vitest tests included so future edits don't silently break the JSON-LD.
  - **Content collections** with `tags`, `readingTime`, `draft`, `bucket` (A/B/C/D query-bucket alignment), `schemas` enum. Native MDX so you write articles in plain markdown + JSX components.
  - **Draft-preview pattern** — every draft gets a working URL for editorial review but stays out of `sitemap-index.xml`, RSS, AND the `/blog` index. Flip `draft: true → false`, commit, push — sitemap regenerates, GSC re-pings automatically.
  - **RSS feed + sitemap-index.xml** with draft filtering, wired via `@astrojs/sitemap` + `@astrojs/rss`. No manual maintenance.
  - **Google Search Console sitemap automation** via the included `gsc_sitemap_submit.py` CLI helper (Google OAuth with `webmasters` scopes — one-time consent, then silent forever). One command per publish — Google re-indexes new cornerstones in hours, not days.
- **Cornerstone-article workflow** — 4-cornerstone seed strategy (one per query bucket A/B/C/D), 2K–4K word target, inline FAQPage + HowTo schema, internal-link discipline, draft-to-publish flip protocol. Plus an **article-session-vs-integration-session protocol**: cornerstone drafting happens in a separate `aio/articles` git branch + worktree to keep MDX work out of the implementation thread (voice drift between TypeScript edits and cornerstone prose is real — separating contexts keeps each thread crisp).
- **Resend** transactional email behind a swappable transport abstraction (welcome emails, contact-form notifications, autoresponders) with DKIM/SPF wired, **gated by Reoon Email Verifier** so disposable / spamtrap / invalid emails never poison your sender reputation. ('Swappable' means you can easily plug-in your own preferred ESP... easy ESPeasy).
- **Reoon Email Verifier deliverability gate** on every form endpoint, with
  - **Quick mode (blocking, ~250ms)** rejecting `invalid` / `disposable` / `spamtrap` at submission time — disposable Mailinator/guerrillamail addresses never enter your ESP list (one disposable signup can poison Resend reputation for every real subscriber for weeks)
  - **Power mode (fire-and-forget async)** surfacing `catch_all` / `role_account` / `inbox_full` to Slack so you triage real-but-risky signups after the fact
  - **80% and 100% daily-quota Slack alerts** so you know when to bump your AppSumo Lifetime Deal tier (Reoon's AppSumo LTD = ~$59 one-time per tier, stackable; Tier 2 = 1,200 verifications/day forever — as of this writing)
  - **`/api/_admin/reoon-usage` spot-check endpoint** for at-a-glance "how much budget do I have left today"
  - **Fail-open posture** — if Reoon is down, real signups still go through (the deliverability gate must never be the thing that creates an outage)
- **A geo-aware cookie consent banner** that adapts to your privacy posture (strict / standard / light) with three-layer Clarity content masking. Allows you to geo-fence to only show banners to GDPR locales, to GDPR+California, etc. You choose your risk-comfort level. This is highly contingent on your marketing niche.
- **Optional: payments** via Stripe (default)
- **Optional: memberships** via Whop (our default recommendation for content/community/course businesses) with HMAC-verified webhooks forwarding subscription events server-side to PostHog. You can swap in Skool / Lemonsqueezy / Paddle / Gumroad, etc.
- **Optional: affiliate program tools and tracking** with
  - multi-layer attribution surviving cookie deletion, Safari ITP, AND ad-platform username changes.
  - Automated short-URL creation for affiliates via Switchy.io API ($39 LTD via Appsumo with very easy set-up, as of this writing)
  - The native integration here is with Whop's affiliate program, but the tools we've created work with any affiliate program that provides API or webhook access. 
- **A pre-launch checklist** that catches the otherwise silent-failures before paid traffic hits the site — now including, for the AIO path:
  - **JSON-LD schema validation** on every prerendered page (valid syntax, `Organization` present, `BreadcrumbList` present on blog posts, no `aggregateRating`/`review` on `Product` — that's a $1M-scale FTC trap if you don't have real reviews)
  - **GSC sitemap reachability + zero-errors check** (sitemap submitted, returns 200, excludes drafts, includes every published cornerstone)
  - **Cornerstone readiness audit** — every published cornerstone has Article + at least one of {FAQPage, HowTo}, AuthorBio rendered, ≥2 internal links to other cornerstones
  - **Google Rich Results Test** run programmatically on every cornerstone URL
- **`src/site-config.ts` single source of truth** for brand identity (SITE_URL, BRAND, AUTHOR_NAME, parent org, sameAs URLs) — imported by every schema builder, layout, RSS feed, blog route. A brand rename = one file edit + Railway env vars, not 8 files in find-and-replace.

The skill ships **16 stages** (0 through 15), each with a "Last verified" date and re-research instructions for when vendor APIs evolve.

## Prerequisites

- **Claude Code** ([install](https://docs.anthropic.com/claude-code))

*Claude will set the following up via the skill if missing:*
- **Node.js ≥ 20** ([install](https://nodejs.org/))
- **Git** ([install](https://git-scm.com/))
- **GitHub CLI** (`gh`) ([install](https://cli.github.com/))
- **GitHub MCP** — connect via "connectors" inside Claude Code
- **Python 3.10+** with `google-auth`, `google-auth-oauthlib`, `google-api-python-client` (needed for the GSC helper script)
- A **Railway** account (free tier works for getting started)
- Optional accounts created during the skill run: PostHog, Microsoft Clarity, Backblaze, Resend, Reoon (free trial then AppSumo LTD recommended), Cloudflare, Stripe / Whop, Google Search Console.

The skill walks you through signing up at each vendor when it's time. You don't need them all upfront.

## Install

### Option 1 — Personal use (most common)

Clone into your Claude Code skills directory:

```bash
# macOS / Linux
git clone https://github.com/drewperd1000/external-website-builder-to-production-v2.git \
  ~/.claude/skills/external-website-builder-to-production-v2
```

```powershell
# Windows PowerShell
git clone https://github.com/drewperd1000/external-website-builder-to-production-v2.git `
  $env:USERPROFILE\.claude\skills\external-website-builder-to-production-v2
```

You can install BOTH v1 and v2 side by side — they're separate skills with different names. Pick the right one per project.

### Option 2 — Workspace-local

Clone into your project's `.claude/skills/` directory instead:

```bash
git clone https://github.com/drewperd1000/external-website-builder-to-production-v2.git \
  /path/to/your/project/.claude/skills/external-website-builder-to-production-v2
```

## How to use it

Start a Claude Code session in the project directory where the website code will live. Tell Claude what you're trying to do — any of these phrasings (or similar) will trigger v2:

```
I have a Webflow export and I want to bring it to production as a content hub.

Set up production for this Readdy site with AI search optimization.

Build the production stack for this landing page, with Astro and schema markup.

Wire PostHog reverse proxy + Reoon + GSC + cornerstones on a new site.

Set up the AIO stack on this marketing site.
```

Claude reads `SKILL.md`, then the appropriate stage files, and starts the onboarding flow. Onboarding asks you a few questions about how technical you are (`non_coder` / `developer` / `expert`), how you want to work (everything upfront vs. phase-by-phase), **whether the site should be discoverable by AI search engines** (this is the new v2 fork), and a handful of project details. Then Claude executes.

## What's in this repo

```
SKILL.md                              ← skill entry point; Claude reads this first
README.md                             ← this file
CHANGELOG.md                          ← what v2 adds vs v1 (10 deltas)
onboarding.md                         ← onboarding flow (modes, expertise, content-hub fork)
stage-0-discovery.md                  ← source-platform dependency + existing-content discovery
stage-1-scaffolding.md                ← clean repo + baseline build (FORK: Astro vs Vite)
stage-2-platform-migration.md         ← migrate off the source platform's infrastructure
stage-3-express-server.md             ← Express server (Basic Auth + prerendered-dir + Astro handler)
stage-4-railway.md                    ← Railway deploy with env-vars-before-build + staging service
stage-5-email-resend.md               ← transactional email + Reoon deliverability gate
stage-6-posthog.md                    ← PostHog with reverse proxy + loaded callback
stage-7-clarity.md                    ← Microsoft Clarity with three-layer masking
stage-8-privacy-consent.md            ← consent state machine + geo-aware banner
stage-9-membership-optional.md        ← Stripe / Whop / Skool / etc. webhook handler
stage-10-affiliate-optional.md        ← multi-layer attribution + branded short URLs
stage-11-variants-and-dashboards.md   ← variant system + 6+ PostHog dashboards
stage-12-launch-checks.md             ← pre-launch verification (incl. GSC + schema audit)
stage-13-content-hub.md               ← NEW: AIO infrastructure (schemas, layouts, collections)
stage-14-cornerstones.md              ← NEW: cornerstone-article workflow + article-session protocol
stage-15-gsc-sitemap.md               ← NEW: Google Search Console property + sitemap automation

_internal/
  claude-invariants.md                ← technical rules Claude follows when writing code
  local-preview.md                    ← local dev verification patterns
  reference-railway-automation.md     ← Railway MCP / CLI / GraphQL deep dive (incl. commitHash gotcha)
  reference-cloudflare-dns.md         ← Cloudflare DNS patterns + cdn-cgi/trace
  reference-object-storage.md         ← Railway Buckets vs Backblaze B2 decision tree
  reference-forms-and-persistence.md  ← four-layer form model + ESP/CRM research-first pattern
  reference-quota-monitoring.md       ← NEW: threshold-alert + admin-endpoint pattern generalized
  reference-aio-schema-library.md     ← NEW: 8 schema component reference + prop interfaces
  reference-prerendered-windows.md    ← NEW: prerendered-dir Express middleware + dotfiles gotcha

examples/
  README.md                           ← pointers to where each canonical code pattern lives
```

## Updating

This skill evolves over time as vendor APIs change, as MCPs become available, and as AI-search ranking signals are better understood (see the "🚧 LEARNINGS-PENDING" callouts in stages 13–14).

If we make updates directly to the skill, you can pull updates:

```bash
cd ~/.claude/skills/external-website-builder-to-production-v2
git pull
```

The bottom of each stage file has a Changelog section dated by when it last shifted.

## Companion skills (optional, highly recommended)

Same recommendations as v1, plus these new pairings that come into play for the AIO surfaces:

| Stage | Companion skill | What it adds |
|---|---|---|
| Stage 13 (content-hub scaffolding) | `superpowers:test-driven-development` | The Vitest schema tests pair naturally with TDD discipline — write the test for the next schema component first, watch it fail, implement the builder, watch it pass. |
| Stage 14 (cornerstones) | `marketing:content-creation`, `brand-voice:enforce-voice`, `marketing:seo-audit` | Draft cornerstones in brand voice, audit for AI-search query coverage, verify SEO basics. |
| Stage 12 (pre-launch, AIO-specific checks) | `marketing:seo-audit` | Comprehensive on-page + technical SEO sweep against the new content hub. |

All other v1 companion skill recommendations apply unchanged. See the full mapping in [SKILL.md](SKILL.md#companion-skills-optional-pull-in-when-their-value-matches-the-moment).

## Voice + automation contract

Same as v1 — the skill follows a strict voice convention so you always know who's doing what:

- **First-person ("I run...", "I write...")** — Claude actually executes via tools (Bash, Edit/Write, MCP)
- **Second-person ("Please open...", "Click here...")** — you must take the action (sign up at a vendor, click in a vendor's no-API dashboard, OAuth approve)
- **Mixed ("Paste the key here; I'll save it...")** — you provide input, Claude processes it

If you ever see Claude claim "I'll do X" but X is something it can't actually do, flag it.

## Privacy + safety

- **No vendor accounts are created without your consent.** When the skill needs you to sign up somewhere, it pauses and asks.
- **Secrets stay out of the chat.** API keys are pasted into project-local `.secrets/<vendor>-key.txt` files (gitignored) and shell-substituted into Railway env vars without entering Claude's context.
- **Reoon's fail-open posture** means a Reoon outage does not block real signups. The skill explicitly documents this as a design choice, not an oversight.
- **The `_internal/claude-invariants.md` file** documents the technical rules Claude follows when writing code (now extended with the Astro middleware-order invariant and the Reoon fail-open invariant).

## Compatibility

| Item | Verified |
|---|---|
| Last vendor-API verification | 2026-05-21 (see "Last verified" lines on each stage file) |
| Source platforms | Readdy, Webflow, Framer, Wix Studio, Bubble, custom code, AI generators (Lovable, v0, Bolt) |
| Frameworks | Astro 5 + React islands + TypeScript (default for content-hub sites); Vite + React + TypeScript (default for non-content-hub sites) |
| Hosts | Railway (canonical) |
| Operating systems | macOS, Linux, Windows (Claude Code Bash tool abstracts shell differences; Windows-worktree gotcha documented in `_internal/reference-prerendered-windows.md`) |

## Contributing

Issues + pull requests welcome. The skill is a living document — each stage file's Changelog is appended in place when patterns evolve.

The 🚧 LEARNINGS-PENDING callouts in stages 13 and 14 are open invitations for contribution — if you've shipped cornerstones long enough to have ranking and citation data, the synthesis would feed back into refined topic-selection criteria, schema combinations, internal-link patterns, and refresh-cadence guidance.

## License

MIT — see [LICENSE](LICENSE) for the full text. Use it freely, fork it, adapt it. Attribution appreciated but not required.

## Background

v2 was built by extracting patterns from a real production reference implementation that shipped the AIO content-hub structure (Astro 5 + 8 schema components + Reoon + GSC + Basic Auth staging + cornerstone scaffolding) over the two weeks prior to the skill's authoring. The patterns are vendor-current as of 2026-05-21; the skill's self-research instruction tells Claude to re-verify the externally-versioned facts before any stage older than 60 days.

The cornerstone-content learnings are deliberately marked as 🚧 LEARNINGS-PENDING because the reference implementation's cornerstones were still in draft at skill-write time. v2 will be updated with refined topic-selection criteria and schema-combination guidance once 30–60 days of ranking and citation data accumulates after the first cornerstones publish.

If you fork this skill, the same construction discipline as v1 applies: forbidden-strings audit on every commit, voice contract enforcement, required-values check pattern, multi-perspective auditing, cross-platform shell discipline. See v1's README "Background" section for the methodology lineage.
