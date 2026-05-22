---
name: external-website-builder-to-production-v2
description: Use when bringing an externally-built website (Readdy zip, Webflow/Framer export, Wix Studio, Bubble, custom code, AI generator output, etc.) to fully-instrumented production AND structured for AI search engine discoverability from day one. Triggers on phrases like "deploy this site to production with AIO", "set up the analytics + AI-search stack on a new site", "bring my marketing site to production as a content hub", "wire PostHog reverse proxy + schema components", "set up a content hub with cornerstones", "make this site rank in ChatGPT Search / Perplexity / Google AI Overview", "add Reoon email verification", "set up Basic Auth staging on Railway", "wire Google Search Console sitemap automation", or any combination of Astro + Railway + PostHog + Clarity + Resend + Reoon + cookie banner + variant tracking + cornerstone content + GSC on a new site.
---

# External Website Builder to Production — v2 (AIO-aware)

**Living skill — last updated 2026-05-21.** Vendor automation surfaces (Railway MCP, PostHog MCP, Clarity MCP, Resend API, Reoon API, Google Search Console API) evolve fast. Before I apply any externally-versioned fact in this skill, I check the "Last verified" line on the relevant stage file. If it's older than 60 days, I re-research by web-searching `<vendor> MCP server <year>` and reading the current README, then append findings to the Changelog so future sessions inherit them.

**This is v2. v1 lives at [`external-website-builder-to-production/`](../external-website-builder-to-production/)** and remains a valid skill for marketing sites where AI-search discoverability is not a goal. See [CHANGELOG.md](CHANGELOG.md) for what v2 adds (Astro 5, schema components, Reoon, GSC automation, Basic Auth staging, cornerstone content workflow).

---

## ⚠️ Automation contract — read first; re-read at the start of every stage

**This is a Claude-driven skill.** When the user invokes me with this skill loaded, I (Claude) execute the technical work. The user makes decisions and provides a small number of inputs at moments where my tooling cannot act on their behalf (account signups, dashboard clicks at vendors that have no API, secret pastes).

For every action this skill describes, the default behavior is:

| What | Who does it | Notes |
|---|---|---|
| Run shell commands (`npm`, `git`, `gh`, `railway`, `curl`, `dig`, `python`, etc.) | I do — via Bash | After one-time `gh auth login` / `railway login` browser approve |
| Create or edit code/config files | I do — via Edit/Write | All `server.js` / `astro.config.mjs` / `package.json` / etc. |
| Verify outputs (network requests, console logs, screenshots, build artifacts) | I do — via Preview MCP / Chrome MCP / Bash | Falls back to user-action if MCPs unavailable |
| Open URLs in user's browser (signup pages, dashboards) | I do via Chrome MCP **if connected**; otherwise user opens manually | I always provide the URL |
| Sign up at vendor (Resend, PostHog, Clarity, Reoon, AppSumo, etc.) | **User does** — Claude can't be them at the identity layer | I provide URL, click path, save the API key the user pastes back |
| Approve OAuth browser flows | User clicks "Approve" once per service | Subsequent operations on that vendor are autonomous |
| Click through dashboard UIs (vendors with no API for the action) | **User clicks** — I provide exact step-by-step click paths | Examples: Whop product setup, Clarity masking config, Google Search Console property verification |
| Add DNS records | I do via Cloudflare API **if user granted a CF API token at onboarding**; otherwise user pastes records into their DNS provider | Asked once at onboarding; unlocks Stages 4 + 5 + 10 + 15 autonomously |
| Provide secrets / API keys | User pastes once into a project-local secrets file; I read + use without displaying | Pattern documented in stage-4-railway.md Step 3b |
| Make business / brand / legal decisions | User decides; I ask plain-language questions in their vocabulary | Pricing, privacy posture, brand voice, cornerstone topics, etc. |
| Write cornerstone article drafts | User decides voice + claims + citations; I draft scaffolding + research helper queries | Articles ship in a separate `aio/articles` branch + worktree — see [stage-14-cornerstones.md](stage-14-cornerstones.md) |

### Voice rule

- **First-person ("I run...", "I write...")** — only when I actually execute via tools (Bash, Edit/Write, MCP)
- **Second-person ("Please open...", "Click here...")** — only when the user must take the action (signup, dashboard clicks at no-API vendors, OAuth approves)
- **Mixed ("Paste the key here; I'll save it...")** — when the user provides input I then process

If a stage describes "I'll do X" but X is something Claude cannot actually do (open the user's web browser to sign them up at a vendor, click in a vendor's account-setup wizard), that's a bug — flag it and I fix.

**Re-grounding rule**: at the start of every stage, I re-read this Automation Contract before executing. If I catch myself describing what the user should do in a step that I could actually automate via my tools, I stop and reframe — and vice versa, if I catch myself claiming I'll do something my tools can't, I stop and reframe to honest user-action language.

---

## What "AIO" means in this skill

AIO = **AI-search optimization**. The marketing site is structured to be discoverable, citable, and rankable by:

- **Google AI Overview** (the AI-generated answer block above standard search results)
- **Perplexity** (AI search engine that cites sources inline)
- **ChatGPT Search** (web-grounded answers in ChatGPT with citations)
- **Claude with web search**
- **Other AI assistants** that retrieve and cite web content

The mechanics for ranking in those surfaces differ from classic SEO. Schema markup (JSON-LD: `Organization`, `Article`, `FAQPage`, `HowTo`, `Product`, `BreadcrumbList`), reciprocal `sameAs` declarations, named author identity (`Person` schema with credentials and affiliations), high-substance long-form articles (2000–4000 words with explicit Q&A and how-to structure), and rapid sitemap submission to Google Search Console — these are load-bearing for AI-search citation.

v1 of this skill produced a working analytics + deploy stack. v2 adds the content-hub infrastructure and content-production workflow that makes the site rank.

**When v2 is the right choice over v1**: any site where AI assistants discovering and citing the brand would meaningfully drive traffic, leads, or trust. For most marketing sites in 2026 and beyond, that's the default — pure-Vite sites without schema infrastructure are leaving citation surface on the table.

**When v1 is still right**: tightly-scoped landing pages with paid-only traffic (no organic ambitions), internal admin tools that should never be indexed, or sites where the user explicitly wants to skip the content-hub work and accept that AI assistants won't surface the site.

---

## User expertise mode

I behave differently based on the user's comfort level. Onboarding asks the user which mode fits them (or auto-detects from their first few messages).

| Mode | When | What I do differently |
|---|---|---|
| **non_coder** (default) | The user has never opened a terminal, hasn't used git/npm, hasn't edited code beyond website-builder UIs. Marketers, founders, designers using Claude Code as their dev partner. | I never show raw code unless asked. I never ask the user to type shell commands. I explain every dev term in plain language at first use. I confirm major actions before running them ("I'm about to deploy. Ready?"). I default to yes/no or pick-from-list questions. |
| **developer** | The user has shipped a few projects, knows git, has used at least one cloud host, can read a `.tsx` or `.astro` file. | I skip beginner explanations. I show diffs when relevant. I run commands without preamble. Questions can include moderate jargon. |
| **expert** | The user has shipped many sites, runs PostHog routinely, knows Cloudflare proxy modes cold, uses Claude Code daily. | Minimum prose. Maximum throughput. I auto-pick defaults; user vetoes. I can answer "explain less" mid-stage. |

The mode lives in the saved skill config as `user_expertise: "non_coder" | "developer" | "expert"`. If unset, I default to `non_coder` and adjust if the user proves more advanced.

---

## Onboarding mode

Two ways for the user to start. Onboarding asks which fits — and I explain both options in plain language so the user can pick the right one for their situation.

### Mode A — `all_upfront`

**What happens**: I ask every question I'll need across the entire build, all in one sitting. Then the user steps away. I run all 16 stages autonomously, pausing only for OAuth approvals (3–5 browser clicks) and any secret-paste moments where the user has to copy a key from a vendor's dashboard.

**Benefits**:
- The user spends ~45–75 minutes upfront making decisions, then can walk away for the rest. Their site builds while they do other work.
- No mid-day interruptions. Every decision is captured at one moment when the user's attention is allocated to this project.
- Total wall-clock time is shorter because async waits (DNS propagation, SSL cert issuance, build queues, Google Search Console verification) overlap.
- If the user's schedule is "I have an evening free" or "I want this set up overnight" — this is the right pick.

**Tradeoffs**:
- The user has to make decisions about things they may not yet have full context on. (Mitigation: I default to recommended values for anything they're not sure about, and they can change later.)
- If the user underestimates how long they'll need the questionnaire to take, they may bail out partway and end up in `phase_by_phase` mode anyway. I check at minute 45: "Want to switch to phase-by-phase?"

### Mode B — `phase_by_phase`

**What happens**: I ask only the minimum questions needed to start (~10 minutes' worth — basically: brand name, domain, where the source code lives, whether this is a content hub). Then I begin Stage 0 and work through stages, asking the questions each stage needs as we hit them.

**Benefits**:
- The user sees real progress (commits, deployments, working pages) within the first session.
- Decisions arrive in context — when I ask about the Resend sending domain in Stage 5, the user is already thinking about email and the question makes more sense than it would have in an upfront block.
- No big upfront cognitive block. Easier to start when the user has limited focus or is uncertain about some choices.
- The user can pause between stages and come back tomorrow without having "wasted" any setup.
- If the user's schedule is "I'll work on this in chunks across the week" or "I want to see something working before I keep investing" — this is the right pick.

**Tradeoffs**:
- More interruptions over the next few hours/days. Every stage has a "wait, I need to ask you something" moment.
- The user has to remain reachable (or at least responsive within a reasonable window) for the build to complete.
- Total wall-clock time is longer because async waits don't overlap as efficiently — Resend DKIM verification might block Stage 5 because we didn't start it during onboarding; Google Search Console property verification might block Stage 15 because we didn't start it earlier.

### Default

If the user is unsure which fits, I default to **`phase_by_phase`** — lower risk of bailing partway, and the user can always upgrade to `all_upfront` mode later by saying "go ahead and set up everything you can; I'll be back in an hour."

The mode lives in the saved skill config as `onboarding_mode: "all_upfront" | "phase_by_phase"`. The user can switch modes mid-flight by saying "switch to all_upfront" / "let me answer everything now" / "stop asking me — pick defaults" — I update the config and proceed accordingly.

---

## Stack fork — content hub vs. simple deploy

Onboarding asks **one new question** that v1 doesn't:

> Should this site be discoverable by AI search engines (ChatGPT Search, Perplexity, Google AI Overview)?

This question forks Stage 1 (and downstream stages):

| Answer | Stack | Stages that change |
|---|---|---|
| **Yes** (default — most marketing sites) | **Astro 5** + React islands + Express wrapper, with schema components, content collections, sitemap+RSS, draft-preview, BaseLayout with `head-extra` slot | Stage 1 (scaffold Astro), Stage 3 (Express wraps Astro handler), Stages 13–15 active |
| **No** (admin tools, paid-only landers, internal apps) | **Vite 6** + React + Express — same as v1 | Stage 1 (scaffold Vite), Stage 3 (Express serves Vite dist), Stages 13–15 skipped |

**Default is YES** — Astro is the recommendation for any marketing site in 2026. The cost of Astro on a non-content-hub site is small (slightly slower hot reload, slightly larger dep tree). The cost of starting on Vite and later realizing you needed schema infrastructure is large (migration touches every page).

The fork lives in skill config as `aio.content_hub: true | false`. Saved at onboarding; consulted at the top of every stage that branches on it.

---

## Placeholder conventions used throughout this skill

The skill uses these placeholders in examples and templates. **I substitute these with the user's real values during execution. I never ship a literal placeholder to a user's project.**

| Placeholder | Real-value shape |
|---|---|
| `Maya's Consulting` | The user's brand name |
| `mayasconsulting.com` | The user's primary domain |
| `app.mayasconsulting.com` | The user's app subdomain (if applicable) |
| `mayas.link` | The user's branded short-URL domain (if applicable) |
| `hello@mayasconsulting.com` | The user's notification email address |
| `phc_xxxxxxxxxxxx` | PostHog Project API Key |
| `<your-clarity-id>` | Clarity Project ID |
| `<your-proxy-slug>` | The chosen PostHog reverse-proxy path slug |
| `<your-author-name>` | The site's named author (used in `Person` schema) |
| `<your-author-suffix>` | Author credentials (e.g., `Ph.D.`, `L.Ac.`, `MBA`) — empty string if none |
| `<your-parent-co>` | Parent company / org (used in `Organization.parentOrganization` schema) |
| `biz_xxxxxx` | Whop business ID (if Whop) |
| `plan_xxxxx` | Whop plan ID (if Whop) |
| `<your-domain>` / `<your-X>` | Generic placeholder for any user-specific value |

These are the canonical placeholder names. When I see one of these in an example, I replace with the user's actual value before applying.

---

## What this skill does

Takes a website built somewhere else — a low-code platform export, an AI site generator zip, a Figma → React handoff, custom code — and produces a fully-instrumented production deployment with:

- **Astro 5 hosting** (or Vite if content_hub=false) wrapped in Express, deployed on Railway with proper env-var-before-build discipline + custom domain + HTTP Basic Auth staging service
- **Schema component library** (`Organization`, `WebSite`, `Person`, `Article`, `FAQPage`, `HowTo`, `Product`, `BreadcrumbList`) with reciprocal `sameAs` declarations across all surfaces
- **Content collections** with draft state, tags, reading-time, and per-post schema enums
- **Draft-preview pattern** — URLs work for editorial review but stay out of sitemap, RSS, and the `/blog` index
- **RSS feed + sitemap-index.xml** with draft filtering
- **Google Search Console** sitemap submission automation via `gsc_sitemap_submit.py`
- **PostHog product analytics** with same-origin reverse proxy (recovers 10–25% of events lost to ad-blocker fingerprinting)
- **Microsoft Clarity** heatmaps + session replay with PostHog identity bridging
- **Resend** transactional email behind a swappable transport abstraction
- **Reoon Email Verifier** as a deliverability gate on every form endpoint (quick blocking + power async, fail-open, daily-quota threshold alerts at 80% and 100%)
- **Geo-aware consent banner** with three privacy posture options plus customization
- **Variant tracking system** (global cohort + per-page lineage) supporting sequential single-page CRO methodology
- **Optional**: Whop / Skool / etc. membership integration with server-side webhook → PostHog forwarding
- **Optional**: multi-layer affiliate attribution surviving cookie deletion AND ad-platform username changes
- **Cornerstone-article scaffolding** — 4 long-form posts targeted at AI-search query buckets, with the article-session-vs-integration-session protocol that keeps MDX work out of implementation threads

## When to use this skill

Direct triggers from the user:
- "Set up production for [a website] with AI search optimization"
- "Deploy this Readdy/Webflow/Framer/Wix/etc. site as a content hub"
- "I want this marketing site to rank in ChatGPT Search and Perplexity"
- "Wire analytics + cornerstones on the new marketing site"
- "Bring this site to production with schema markup"
- ANY combination of: Astro + Railway + PostHog + Clarity + Resend + Reoon + cookie banner + variant tracking + cornerstone content + GSC on a NEW site

## When NOT to use this skill

- **The site is already live and the user just wants to fix one integration** → I open the relevant `stage-N-*.md` file directly. Don't run the whole flow.
- **Pure backend service with no marketing surface** → most stages don't apply; cherry-pick.
- **Site is an internal admin tool that should never be indexed** → use v1, skip the content-hub layers, set robots.txt to disallow.
- **User explicitly says "I don't care about AI search; I only want paid-traffic landers"** → use v1.

## Process — onboarding + 16 stages

I execute the stages in order. Each stage outputs artifacts the next stage depends on, so I don't skip ahead.

**Always start with [onboarding.md](onboarding.md)** — it captures expertise mode, onboarding mode, account/tool prereqs, the content-hub fork, and the decisions later stages need. At the end of onboarding I save the answers to a project-local `.skill-config.json` (gitignored) so future sessions pick up where they left off.

| Stage | What | File | Required? |
|---|---|---|---|
| **Onboarding** | Mode + expertise + accounts + content-hub fork + decision capture | [onboarding.md](onboarding.md) | ✅ Always — start here |
| 0 | Code-discovery + migration punchlist + existing-content audit | [stage-0-discovery.md](stage-0-discovery.md) | ✅ Always |
| 1 | Scaffold project + baseline build (FORK: Astro 5 vs Vite per content-hub answer) | [stage-1-scaffolding.md](stage-1-scaffolding.md) | ✅ Always |
| 2 | Migrate platform-specific dependencies | [stage-2-platform-migration.md](stage-2-platform-migration.md) | ✅ Always (if any from Stage 0) |
| 3 | Add Express server (wraps Astro handler or serves Vite dist) | [stage-3-express-server.md](stage-3-express-server.md) | ✅ Always |
| 4 | Railway service deploy (env BEFORE build) + staging service with Basic Auth | [stage-4-railway.md](stage-4-railway.md) | ✅ Always |
| 5 | Email provider + Reoon deliverability gate | [stage-5-email-resend.md](stage-5-email-resend.md) | If forms or transactional emails exist |
| 6 | PostHog setup with reverse proxy | [stage-6-posthog.md](stage-6-posthog.md) | ✅ Always |
| 7 | Clarity setup | [stage-7-clarity.md](stage-7-clarity.md) | Strongly recommended |
| 8 | Privacy + consent | [stage-8-privacy-consent.md](stage-8-privacy-consent.md) | ✅ Always |
| 9 | Membership platform | [stage-9-membership-optional.md](stage-9-membership-optional.md) | If selling subscriptions/memberships |
| 10 | Affiliate tracking | [stage-10-affiliate-optional.md](stage-10-affiliate-optional.md) | If running an affiliate program |
| 11 | Variant system + dashboards | [stage-11-variants-and-dashboards.md](stage-11-variants-and-dashboards.md) | Strongly recommended |
| 12 | Pre-launch checks (includes GSC sitemap submit + schema validator + JSON-LD audit) | [stage-12-launch-checks.md](stage-12-launch-checks.md) | ✅ Always |
| **13** | **AIO content-hub infrastructure (schema components, BaseLayout, content collections, sitemap+RSS, draft-preview)** | [stage-13-content-hub.md](stage-13-content-hub.md) | If `aio.content_hub = true` |
| **14** | **Cornerstone-article workflow (query buckets, schemas, internal-link discipline, article-session protocol)** | [stage-14-cornerstones.md](stage-14-cornerstones.md) | If `aio.content_hub = true` |
| **15** | **Google Search Console property + sitemap automation** | [stage-15-gsc-sitemap.md](stage-15-gsc-sitemap.md) | If `aio.content_hub = true` |

**Stage execution order with the AIO fork**: when `content_hub = true`, Stage 13 runs BEFORE Stages 6–11 (schema and content collections are scaffolded into the Astro project before analytics + privacy + variants get wired). Stage 14 runs in parallel with Stages 6–11 (cornerstones can be drafted in a separate worktree while integration work proceeds). Stage 15 runs immediately after Stage 4 (so GSC verification can complete during the long-tail of other stages).

A reader-friendly ordering for the AIO path:
**Onboarding → 0 → 1 (Astro) → 2 → 3 → 4 → 13 → 15 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 14 (parallel from here) → 12**

Plus the cross-cutting reference files (kept under `_internal/` because they're Claude's working notes — not user-facing):

- **[`_internal/local-preview.md`](_internal/local-preview.md)** — `npm run dev` vs `npm run start`, `.env.local` for prod-parity, autonomous-verification via Preview MCP / Chrome MCP, multi-device LAN access, Lighthouse audits.
- **[`_internal/reference-railway-automation.md`](_internal/reference-railway-automation.md)** — Railway's MCP / CLI / GraphQL surfaces. **v2 update: documents the `latestDeployment.id` non-change gotcha** — only `meta.commitHash` reliably changes between deploys, so polling must check commit-hash not id.
- **[`_internal/reference-cloudflare-dns.md`](_internal/reference-cloudflare-dns.md)** — `cdn-cgi/trace` geo detection, proxy vs DNS-only decisions, Cloudflare Error 1000 explainer, SPF stacking, Redirect Rules.
- **[`_internal/reference-object-storage.md`](_internal/reference-object-storage.md)** — Railway Buckets vs Backblaze B2 decision framework, the public-URL caching gotcha.
- **[`_internal/reference-forms-and-persistence.md`](_internal/reference-forms-and-persistence.md)** — the four-layer model for every form (validate → DB capture → notify-email → ESP segment sync → CRM sync → PostHog event), default-DB-capture rule.
- **[`_internal/reference-quota-monitoring.md`](_internal/reference-quota-monitoring.md)** — NEW in v2. The Reoon pattern generalized: in-memory daily counter + UTC midnight reset + 80%/100% Slack threshold alerts + admin spot-check endpoint. Applicable to Resend monthly cap, PostHog volume cap, B2 storage, any per-day or per-month API budget.
- **[`_internal/reference-aio-schema-library.md`](_internal/reference-aio-schema-library.md)** — NEW in v2. The 8 schema component reference: prop interfaces, JSON-LD output shapes, Vitest assertion patterns, the FTC traps to avoid (`aggregateRating` and `review` on `Product` without real reviews).
- **[`_internal/reference-prerendered-windows.md`](_internal/reference-prerendered-windows.md)** — NEW in v2. The prerendered-directory + Windows-worktree gotcha: explicit middleware to resolve `<path>/index.html` for prerendered blog routes, and the `dotfiles: "allow"` option needed for paths containing `.worktrees/` or any dot-segment.

## A note on Custom Connectors (MCPs not in Claude Code's built-in list)

Several vendors this skill integrates with have shipped MCP servers that **may not appear in Claude Code's built-in Connectors picker**. They can still be added — Claude Code supports "Custom Connector" / custom MCP server URLs added directly through the user's settings or via the vendor's own setup wizard.

**Specific cases I encounter in this skill** (verified 2026-05; vendor MCP availability shifts continuously):

| Vendor | MCP endpoint | How to add (if not in built-in Connectors) |
|---|---|---|
| **Railway** | `https://mcp.railway.com` (SSE) | Custom Connector via Claude Code settings, OR Railway's own setup page links to the install. OAuth approve once. |
| **Whop** | Whop has shipped an MCP for content/community/course operations. | Custom Connector — Whop's docs page usually has an "Add to Claude Code" button. |
| **PostHog** | Built-in installer: `npx @posthog/wizard@latest mcp add` handles config + OAuth in one step. Listed in Connectors on most current Claude Code versions. |
| **Microsoft Clarity** | API-token-paste install (no OAuth). May require manual config rather than Connectors UI. |
| **Google Search Console** | No vendor MCP. I use the `gsc_sitemap_submit.py` CLI helper authenticated via the project's `oauth_helper.py` (Google OAuth with `webmasters` + `webmasters.readonly` scopes). |
| **Reoon Email Verifier** | No vendor MCP. REST API + API key paste; the `email-verifier.js` reference implementation handles the full integration. |

**My discipline** when a vendor MCP isn't appearing in the user's Connectors picker:
1. I check the vendor's own docs page for "Claude Code" / "MCP server" instructions — most vendors who ship an MCP also publish setup steps.
2. If those instructions exist, I walk the user through the Custom Connector add path (settings → MCP → add custom → paste URL → approve OAuth).
3. If no MCP exists for that vendor, I fall back to REST API + API key paste (the pattern Stage 5 uses for Resend and Reoon, which have no MCP yet as of 2026-05).

The user shouldn't need to know which vendors are in the built-in Connectors picker vs. which need Custom Connector setup — I just walk them through whichever path applies for each integration.

## Operating principles Claude follows during execution

Beyond the code-level invariants below, I follow **10 universal operating principles** every time I execute a v2 stage:

1. Verify deploys by commit-hash, not bundle-hash or deployment-id
2. Cross-surface naming lock-step (one rename = scan + update all consumers in one session)
3. Never hardcode production domains; source from env vars
4. Push after every meaningful commit
5. Plain-language commands for users (no jargon)
6. Surface real problems early
7. Defer to provider taxonomies before inventing your own
8. Wire Slack notifications for ops-relevant events
9. Don't claim "locked in" / "deferred" without the canonical doc current
10. Backup discipline for production sites

These are extracted from observed production-deployment failure modes, not aesthetic preferences. Full rationale + where each principle is enforced lives in [`_internal/reference-operating-principles.md`](_internal/reference-operating-principles.md).

**These principles also get scaffolded into the user's `<project>/CLAUDE.md` at [Stage 1](stage-1-scaffolding.md).** Claude Code auto-loads `CLAUDE.md` from the project directory on every future session, so every Claude session working on the user's project inherits the principles without me having to re-explain them. The user extends the file as project-specific conventions emerge.

## Technical invariants Claude follows when writing code

There are now **eight** silent-failure modes Claude needs to avoid when writing the code for this skill — six from v1 plus two new for AIO:

1. (v1) Vite/Astro build-time env-var bake
2. (v1) PostHog reverse proxy mounted BEFORE `express.json()`
3. (v1) Super-properties register BEFORE first capture
4. (v1) `/static/*` served LOCALLY, not proxied
5. (v1) Three Clarity masking layers are belt-and-suspenders
6. (v1) Privacy posture is a project-start decision
7. **(v2) Astro middleware order: PostHog proxy → Basic Auth → JSON body parser → form endpoints → admin endpoints → prerendered-dir resolver → static → Astro handler — in that order**
8. **(v2) Reoon fail-open posture: every network error, timeout, non-200, or 429 returns `{ ok: true }` so signups continue if Reoon is down**

The full technical detail lives at **[`_internal/claude-invariants.md`](_internal/claude-invariants.md)**, which Claude reads and follows. The user can review it if they want to understand exactly what Claude is doing under the hood, but it's not required reading.

## Process loop within each stage

For every stage I execute:
1. **Re-read the Automation Contract** at the top of this file (combat drift to "tell user what to do").
2. **Read the stage file** before doing anything.
3. **Run the required-values check** (see "Required-values check pattern" below).
4. **Follow it in order** — instructions are sequenced because of inter-step dependencies.
5. **Verify the output** matches the stage's "Verification" section using my own tools.
6. **Stop on verification failure** — I debug within the stage. Do not advance.
7. **Show the user a plain-language summary** of what I did and what they need to confirm before next stage.

## Required-values check pattern (universal — runs at start of every stage)

Onboarding may not have captured every value a stage needs. Reasons:
- The user picked `phase_by_phase` mode (only ~5 essential questions answered upfront)
- The user is using a saved config from an older skill version that didn't include a newer field (e.g., `aio.content_hub`, `aio.cornerstones[]`, `email.reoon_enabled`, `hosting.staging_basic_auth`)
- A stage was previously skipped and is now being run

**The pattern, executed at the start of every stage**:

1. I read `<project>/.skill-config.json` and check for the values this specific stage needs (each stage's file lists these in its "Required-values check" preamble — Stages 8 and 13 are the canonical examples).
2. I categorize missing values as either **blocking** (this stage cannot meaningfully produce output without them) or **deferrable** (this stage can do partial work; the value is needed before a specific later step within the stage).
3. **For blocking-missing values**: I prompt the user now in plain language, explaining what the value is and why it matters. I provide an auto-proposal where one is sensible. I save the user's answer to skill config so future stages and future runs don't ask again.
4. **For deferrable-missing values**: I do the work that doesn't depend on them in parallel while asking the user to fetch / decide. The classic case: "I need your Reoon API key, but I can finish writing the email-verifier integration while you grab it from the dashboard."

**Why this exists**: when a saved config is incomplete, the right behavior is to ask narrowly for what's missing — NOT to bail out, and NOT to silently substitute placeholder garbage that breaks at runtime.

## Common rationalizations to refuse

| Thought | Reality |
|---|---|
| "I'll just set the Vite/Astro env vars later" | Both bake at build. Later = silent prod outage until next deploy. |
| "I'll skip the reverse proxy for v1" | 10–25% of events are lost to ad-blockers from day one. Migration cost only grows with traffic. |
| "I'll add the consent banner before launch" | Once analytics is live without it, retention rules may force discarding data. Banner first. |
| "I'll use the standard `i.posthog.com` URL" | Adblocker fingerprints. Same-origin from day 1. |
| "One giant useEffect for pageview tracking is simpler" | Races posthog-js init. Loaded callback is the documented pattern. |
| "Whop's webhook payload has the affiliate, right?" | No (verified 2026-04-28). Affiliate attribution must be DB-mediated. |
| "I'll just hardcode the domain in two places" | When DNS migrates, every hardcoded reference surfaces the hard way. Use `APP_URL` env var. |
| "Skipping Stage 0 — I know what's in this codebase" | Stage 0 surfaces things confident grepping misses. 30 minutes saves a launch-blocker incident. |
| "The user can run this command themselves" | Default is no — I run it. Only ask the user to type a command if I genuinely cannot. |
| **"AIO content hub can be added later"** | Schema infrastructure threads through every page layout. Adding after a Vite site is built means a stack migration. Cheaper to scaffold Astro from the start even if cornerstones come later. |
| **"Drafts are fine; just leave them out of git"** | Drafts in MDX with `draft: true` get prerendered URLs for editorial preview. Leaving them out of git means losing the draft-preview workflow. Commit drafts; the sitemap filter excludes them. |
| **"Reoon is a nice-to-have"** | A welcome email sent to `disposable@mailinator.com` poisons the Resend reputation domain-wide. One disposable signup can degrade deliverability for every real subscriber for weeks. Reoon's blocking gate is cheap insurance. |
| **"GSC sitemap submission is a manual one-time thing"** | True for the first submission. Wrong for ongoing — every cornerstone publish or draft flip should re-ping. Automation via `gsc_sitemap_submit.py` makes this trivial; manual makes it forgotten. |
| **"Polling Railway by `latestDeployment.id` is fine"** | NO. Railway re-uses the deployment record in place. Only `meta.commitHash` changes between deploys. See `_internal/reference-railway-automation.md`. |

## Self-research instruction

Before starting **Stage 4** (Railway), **Stage 6** (PostHog), **Stage 7** (Clarity), or **Stage 15** (Google Search Console):

1. I web-search `<vendor> MCP server <current year>` and read the current README.
2. I compare against the "Last verified" line on the corresponding stage file.
3. If the auth model or capability surface has changed, I USE the new affordances and append a dated entry to the Changelog at the bottom of the relevant stage file.

The vendor landscape moves faster than this skill can. Stage files are written to be REPLACED in place, not patched defensively.

## Pre-final audit checkpoint (before any "skill is shipped" claim)

Whenever this skill is materially edited, I run the audit procedure documented in [`_internal/claude-invariants.md`](_internal/claude-invariants.md) before declaring the edit complete. This is non-optional. The forbidden-strings pattern list in that file is the canonical reference. **Zero matches across the entire skill = ready to commit. Any match = revisit and abstract before committing.**

I run the audit specifically when:
- A new stage file is added or substantially rewritten.
- Code samples have been pasted from a real project (high risk of literal domains, key prefixes, brand colors, statutory citations).
- Voice has been edited (drift between first-person Claude actions and second-person user actions is the source of "I'll do X" fictions where Claude can't actually do X).
- The Automation Contract has been touched.

## Companion skills (optional; pull in when their value matches the moment)

This skill handles deployment + infrastructure + instrumentation + AIO content-hub structure. It deliberately does NOT prescribe a visual aesthetic, copywriting voice, or accessibility methodology. If the user has design-, marketing-, or quality-focused skills installed in their Claude Code environment, I can invoke them at moments where their specific value applies.

| Stage / phase | Companion skills worth invoking |
|---|---|
| After Stage 0/1, before Stage 2 cleanup | `audit`, `security-audit`, `redesign-existing-projects` (if imported design feels generic-AI) |
| Visual polish (anywhere between Stage 1 and 12) | `polish`, `critique`, `accessibility-review`, `optimize`, `layout`, `typeset`, `clarify` |
| Aesthetic direction (one per project) | `high-end-visual-design`, `minimalist-ui`, `industrial-brutalist-ui`, `impeccable`, `emil-design-eng`, `design-taste-frontend` |
| Copy + content (Stage 5 emails, Stage 8 privacy text, Stage 9/10 marketing pages) | `marketing:content-creation`, `marketing:email-sequence`, `marketing:brand-review`, `brand-voice:enforce-voice`, `design:ux-copy` |
| **Cornerstone article drafting (Stage 14)** | `marketing:content-creation`, `brand-voice:enforce-voice`, `marketing:seo-audit` |
| **Schema component validation (Stage 13)** | `superpowers:test-driven-development` (the Vitest scaffold pairs naturally with TDD discipline) |
| Pre-launch (Stage 12 companions) | `audit`, `security-audit`, `accessibility-review`, `optimize`, `legal:compliance-check` |
| Post-launch iteration | `superpowers:executing-plans`, `marketing:performance-report`, `marketing:campaign-plan` |

**My discipline**: I don't auto-invoke these. The user pulls them in when they want them, OR I suggest one when its specific value clearly matches the moment.

## Common command patterns the user might say

When the user says one of these, I run the corresponding flow:

| User says | I do |
|---|---|
| "Set up production for [site]" with AIO defaults | All 16 stages, starting with onboarding (Astro path) |
| "Set up production for [site]" without AIO | All 13 stages, starting with onboarding (Vite path — same as v1) |
| "Add analytics to [existing site]" | Stages 6 + 7 + 8 (skip earlier scaffolding) |
| "Set up the cookie banner" | Stage 8 alone, with quick prompt for posture |
| "Wire affiliate tracking" | Stage 10, after confirming the membership platform from Stage 9 |
| "Migrate [site] from [platform] to production" | Stage 0 with that platform pre-selected, then full flow |
| **"Add a content hub to my existing site"** | Stage 13 + Stage 14 + Stage 15 (skip earlier scaffolding) — but warn if the existing site is Vite-only; the migration is non-trivial |
| **"Set up cornerstone articles"** | Stage 14 alone (requires Stage 13 already complete) |
| **"Submit my sitemap to Google Search Console"** | Stage 15 alone |

## Outputs of a successful run

When all 16 stages complete (AIO path), the user has:

- A Railway-deployed Astro/React/Express site at their chosen domain with HTTP Basic Auth staging at `<name>-staging.up.railway.app`
- PostHog Live Events showing `$pageview` + custom events with deploy + variant super-properties on every event
- Clarity recordings with PostHog `distinct_id` + `session_id` as custom identifiers (cross-tool pivoting works)
- Resend sending welcome / autoresponder / notification emails from a verified domain
- Reoon Email Verifier gating every form endpoint with quick blocking + power async; 80%/100% Slack alerts wired
- A consent banner appearing in the configured locales, with version sentinels persisted in localStorage
- A `/blog` content hub with cornerstone scaffolding, RSS feed, sitemap-index.xml, draft-preview pattern
- 8 schema components emitting valid JSON-LD on every page (verified by Vitest + Google Rich Results Test in Stage 12)
- Google Search Console property verified, sitemap submitted, automation script ready for ongoing re-pings
- Optional: Whop subscription events forwarded server-side to PostHog with affiliate attribution
- Optional: branded short URLs serving as stable indirection for affiliate URLs
- A set of saved PostHog dashboards with a runbook
- A repo-level `migration-punchlist.md` archiving every dependency I discovered + how I resolved it
- An `.env.example` enumerating every required env var so future redeploys don't hit the build-time silent-darkness trap
- A separate `aio/articles` branch + worktree set up for cornerstone drafting (article session vs. integration session protocol)

## Changelog

| Date | Change |
|---|---|
| 2026-05-21 | Initial v2 created. Forks from v1 with 10 deltas: Astro 5 as default stack (replaces Vite-only), schema component library, Reoon email verifier (ported from TODO comment in v1 to actual code), GSC sitemap automation, Basic Auth staging service, client:only React island pattern, prerendered-directory Windows-worktree gotcha, threshold-alert generalized pattern, Railway commitHash deploy-verification gotcha, cornerstone-article workflow. Three new stages (13/14/15) and three new `_internal/` references (`reference-quota-monitoring.md`, `reference-aio-schema-library.md`, `reference-prerendered-windows.md`). See [CHANGELOG.md](CHANGELOG.md) for full delta detail. |
