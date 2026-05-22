# Onboarding — Start Here

**Last verified: 2026-05-21.**

> **🤖 Re-grounding (re-read at start)**: This is a Claude-driven skill. I (Claude) execute the work; the user provides decisions and a small number of inputs. Onboarding's job is to figure out (1) how the user wants to work with me, (2) whether this site should be discoverable by AI search engines, (3) named-author identity for schema markup, and (4) the minimum information I need to start. I do NOT make the user do technical setup during onboarding unless they explicitly want to.

> **Upgrading from v1?** See [CHANGELOG.md](CHANGELOG.md) for what's new in v2 onboarding (content-hub fork, named-author questions, cornerstone seed topics, Reoon tier). The existing v1 flow is preserved; the new questions slot in as additional steps.

## What this skill does for the user (in plain language)

The user has a website built somewhere — Webflow, Framer, Wix Studio, Readdy, an AI generator, or custom code. They want it live on a real domain with:

- A working site at their own URL
- Visitor analytics so they can see what's working
- Heatmaps + session replay so they can watch how people use the site
- Email forms that actually deliver (welcome emails, contact forms)
- A cookie consent banner that's legally appropriate for their audience
- Optional: subscriptions / paid memberships
- Optional: an affiliate program with stable referral links

I (Claude) build all of this. The user makes decisions and provides a few inputs along the way (account approvals via browser clicks, secret keys pasted into a file once, dashboard clicks where vendors don't have an API). I don't ask the user to write code, run terminal commands, or edit configuration files unless they explicitly want to be involved at that level.

## Estimated time

| Onboarding mode | Upfront time | Then what |
|---|---|---|
| `all_upfront` | 30–60 min answering questions | I run all 13 stages autonomously while the user does other work. Total: ~3–6 hours of mostly-async wall-clock time. |
| `phase_by_phase` | 5–10 min answering essential questions | I start Stage 0 immediately. We work through stages together; I ask follow-up questions as needed. Total spread across hours or days depending on the user's availability. |

If the user has used this skill before for another project, I can clone settings from a previous project's saved config and skip 80% of the questions.

---

## Step 1 — User expertise mode

I behave differently based on how comfortable the user is with developer-side work. I ask plainly:

```
First — quick check on how you'd like me to work with you:

(a) I'm new to this — I've used website builders and analytics dashboards,
    but I don't write code or use a terminal. Talk to me in plain language,
    don't show me code, and just do the work. (Most users pick this.)

(b) I'm comfortable with code — I've shipped a few projects, can read a
    .ts/.js file, used git/npm. Skip the beginner explanations.

(c) I'm an expert — I run Railway/PostHog/Cloudflare/Claude Code daily.
    Minimum prose, maximum throughput. Auto-pick defaults; I'll veto if
    I disagree.

Which fits you best?
```

**I save the answer** to the skill config as `user_expertise: "non_coder" | "developer" | "expert"`.

This drives how I phrase questions in later steps:

- **non_coder**: every question is yes/no or pick-from-list, every term explained at first use. I never show code or commands. I default to recommended values and confirm before each major action.
- **developer**: I skip beginner explanations, can show diffs/snippets, run commands without preamble. Questions can use moderate dev jargon.
- **expert**: minimum prose. I auto-pick defaults, present a summary, and ask "any deviations?" once.

**If the user is uncertain, I default to `non_coder`** — it's safer to over-explain than to leave them lost. I can always upgrade mid-stream when the user proves more advanced.

---

## Step 2 — Onboarding mode

```
How do you want to work through this?

(a) Tell me everything upfront, then walk away — I'll answer all your
    questions now (about 30–60 minutes), and you can build everything
    while I'm doing other things. I'll just need to click "Approve" in
    a browser tab a few times when you ping me.

(b) Get started fast, ask me questions as we go — I'll answer just the
    essentials now (5–10 minutes), and you can begin work immediately.
    You'll ask me more questions as each stage needs them.

Which fits your schedule today?
```

(See SKILL.md "Onboarding mode" section for the full benefits/tradeoffs of each. If the user is unsure, I describe both at the level of detail they want.)

**I save the answer** as `onboarding_mode: "all_upfront" | "phase_by_phase"`. The user can switch mid-stream by saying "switch to all_upfront" / "stop asking, pick defaults" / "let me answer everything now" — I update the config and proceed.

**Default if user is unsure**: `phase_by_phase` (lower risk of bailing partway, more forgiving).

---

## Step 3 — Load saved config (if available)

Before I ask anything else, I check if the user has used this skill on a previous project. The saved configs live at `<project>/.skill-config.json` (project-local, gitignored) and optionally at `~/.claude/skill-configs/<slug>.json` (cross-project library).

If I find any:

```
I found previous skill configs from these projects:
  - mayasconsulting-marketing.json (saved 2026-04-12, Webflow origin, no membership)
  - acme-saas.json (saved 2026-03-08, custom code, Whop affiliates)

Would you like to:
  (a) Start fresh — answer all questions for this new project
  (b) Clone settings from a previous project — I'll copy the answers,
      and we'll just review project-specific items (domain, GitHub repo)
  (c) Use a previous project's answers as suggested defaults — I'll show
      each saved value and you confirm or change
```

If they pick (b) or (c), I load the JSON, present the relevant answers, and only ask about things that actually change between projects (project name, domain, brand, etc.).

---

## Step 4 — Account check (autonomous; user just confirms)

Before asking the user to set up anything, I check what they already have. I run these checks myself via my Bash tool:

| Check | Command I run | What I learn |
|---|---|---|
| GitHub auth | `gh auth status` | Is the user already authed to GitHub? |
| Railway auth | `railway whoami` | Same for Railway? |
| Node version | `node --version` | Are they on Node ≥ 20? |
| Git installed | `git --version` | Is git available? |
| Railway CLI installed | `which railway` (or `where railway` on Windows) | Have I got the CLI to call? |
| Preview MCP available | check available tools list | Can I do autonomous browser verification? |
| Chrome MCP available | check available tools list | Same for Chrome MCP |
| Existing skill config | check filesystem | Have they done this before? |

I report results in plain language:

```
✅ Node 20.18.0 installed
✅ git 2.43 installed
✅ GitHub authenticated as @username
❌ Railway CLI not installed yet — I'll set this up when we hit Stage 4
✅ Preview MCP available (I can verify changes in a browser autonomously)
❌ Chrome MCP not connected (skip; we don't need it unless we're testing
   against a live production URL)
```

For anything missing that the user can fix in seconds, I offer to install / set up now. For anything that requires a vendor signup, I defer until the relevant stage:

```
Should I:
  (a) install the Railway CLI now? (one command, ~30 seconds)
  (b) wait until Stage 4 and do it then?
```

---

## Step 4b — Content-hub fork (Astro vs Vite)

This is the key v2 question. It determines whether the site gets the AIO content-hub infrastructure (Astro 5 + schema components + content collections + sitemap+RSS + cornerstone workflow) or stays on the simpler Vite-only path.

```
Should this site be set up to be discovered by AI search engines
(ChatGPT Search, Perplexity, Google AI Overview, Claude with web search)?

  (a) Yes (recommended for marketing sites) — I'll scaffold an Astro 5
      stack with schema markup, a content hub at /blog with cornerstones,
      sitemap+RSS infrastructure, and Google Search Console automation.
      This is the right pick for any site where AI assistants citing
      the brand by name would meaningfully drive traffic, leads, or
      trust. Adds about 1-2 hours of setup time vs the simpler path,
      and an ongoing content-production effort for cornerstones (which
      is the real differentiator — schema markup without long-form
      content underneath it ranks for very little).

  (b) No — this is a paid-traffic lander, an internal admin tool, or
      a tightly-scoped site without organic ambitions. Scaffold with
      Vite, skip the AIO infrastructure.

  (c) I'm not sure — explain the tradeoffs more.

Which fits?
```

If (c), I walk through:

```
The tradeoffs:

CHOOSING (a) AIO/Astro:
  + AI assistants are increasingly the discovery layer between users and
    websites — ChatGPT Search, Perplexity, Google AI Overview, Claude.
    A site without schema infrastructure is invisible to them.
  + Schema markup is also good for classic SEO (Google Rich Results,
    featured snippets).
  + Cornerstone articles are evergreen content marketing — they keep
    earning traffic for years after publish.
  - Adds ~1-2 hours to initial setup.
  - Requires committing to writing 4 cornerstones (2,000-4,000 words
    each). Without cornerstones, the schema infrastructure is wasted.
  - Astro has a slightly slower hot reload than pure Vite during
    development. Minor.

CHOOSING (b) Vite-only:
  + Simpler setup.
  + Smaller dep tree.
  + Faster hot reload.
  - No AI-search discoverability. AI assistants won't cite the brand
    even when they're answering questions in the brand's domain.
  - If you later decide you want AIO, migrating Vite to Astro is
    a project, not a tweak.

Most marketing sites in 2026 should pick (a). Pick (b) if:
  - The site has zero organic ambitions (paid traffic only, internal
    admin tools, throwaway landers)
  - You don't have time or willingness to write 4 cornerstone articles
    in the first ~3 months post-launch
  - You're certain the brand won't benefit from AI-search citations

Which fits?
```

**I save the answer** as `aio.content_hub: true | false`.

**Default if unsure**: `true`. The cost of Astro on a non-content-hub site is small. The cost of starting on Vite and later realizing you needed schema infrastructure is large (stack migration touches every page).

If the user picks (a), I also collect the named-author identity for `Person` schema:

```
The Person schema is the most-cited entity for AI-search "author of"
queries. AI assistants weight authored content over anonymous content
significantly. To set up the schema:

  Q1. Who's the named author of the site's articles? (Real human name,
      not a brand name or pen name — AI assistants disambiguate by
      Person entity, not Brand entity, for author attribution.)
      Default: <your-name-from-git>. Tell me if it's someone else.

  Q2. Any author credentials to include? (e.g., "Ph.D.", "L.Ac.", "MBA",
      "MD" — appears next to the author name in some Rich Result formats.)
      Default: none.

  Q3. Author's job title(s)? (e.g., "Acupuncturist", "Researcher",
      "Software Engineer at X" — used for `Person.jobTitle`.)
      Default: none (leave empty).

  Q4. Author's alma mater(s)? (Schools/universities — used for
      `Person.alumniOf`, reinforces credentials.)
      Default: none.

  Q5. Author's affiliations? (Current employer, advisory roles, etc. —
      used for `Person.affiliation`.)
      Default: none.

  Q6. URLs that represent the same author/brand entity? (LinkedIn,
      Twitter, GitHub, app subdomain, parent company site — used in
      `sameAs` arrays to strengthen entity disambiguation.)
      Default: just the site's own URL.

You can answer "skip" to any of these and we'll add them later.
```

I save the answers as `aio.author_name`, `aio.author_suffix`, `aio.author_job_titles[]`, `aio.author_alumni_of[]`, `aio.author_affiliations[]`, `aio.same_as[]`.

I also collect the parent organization (used for `Organization.parentOrganization`):

```
  Q7. Is this brand part of a parent organization or holding company?
      (e.g., "<your-parent-co>" is the parent of "Maya's Consulting"
      — used in the Organization schema's parentOrganization field.)
      Default: none (most sole-proprietorships and stand-alone brands
      leave this empty).
```

I save as `aio.parent_org` and `aio.parent_org_url`.

## Step 5 — Services this skill will use (informational)

I show the user a quick overview of what services will be involved across the build, so they can flag deal-breakers EARLY (e.g., "I refuse to use Whop") rather than discover them mid-stage:

| Service | Used in | Cost | Required? |
|---|---|---|---|
| **GitHub** | Source-of-truth git repo | Free | Yes |
| **Railway** | Hosting | Free dev tier; ~$5/mo for prod-grade single service | Yes |
| **PostHog** | Product analytics, session replay, feature flags | Free up to 1M events/mo, 5,000 desktop + 2,500 mobile session replays/mo | Yes |
| **Cloudflare** | DNS + free SSL + geo detection | Free for what we use | Strongly recommended (see Step 5b for WHY) |
| **Resend** (or alternative) | Transactional email | Free 3k emails/mo, 100/day | If site has forms |
| **Microsoft Clarity** | Heatmaps + session replay (effectively unlimited) | Free at any scale, no recording cap | Strongly recommended (see "Why both" note below) |
| **Stripe** (default for payments) or **Whop** (recommended for content/community/course businesses; can also handle payments via its API if you prefer to consolidate) or Skool / Lemonsqueezy / Paddle / Gumroad | Subscriptions / memberships / one-time payments | Stripe: 2.9% + 30¢/transaction; Whop: ~3% of revenue + built-in marketplace + affiliate program | Only if selling |
| **Branded short URL** | Affiliate URL stability + general tracking | Domain registration ~$10–60/yr | Only if affiliates |
| **Railway Buckets** (default for private files) or **Backblaze B2 / R2 / S3** (default for public media or when uncertain) | Object storage: user uploads, media library, variant screenshots, etc. | Railway: $0.015/GB-month + free egress; B2: ~$0.006/GB-month + ~$0.01/GB egress with free 3× daily window | Only if site stores files |

**For each "Required" service the user doesn't already have**: I help them sign up at the relevant stage, with click-by-click guidance. They don't need to do all of this upfront.

**For each "Only if X" service**: I ask in Step 6 whether they want it, then defer signup to the relevant stage.

If the user has a deal-breaker for any service (e.g., "I want Stripe, not Whop" / "I'll skip Clarity"), they say so now and I adapt subsequent stages.

### Why install BOTH PostHog and Clarity (not either-or)

These two tools are **complementary, not competing**. The default install puts both on the site for these reasons:

- **PostHog** is the analytics + product-decision tool: events, funnels, retention, feature flags, dashboards, A/B-test math, cohorts, person profiles. Its session replay is good but capped at **5,000 desktop + 2,500 mobile recordings per month** on the free plan. (Source: PostHog free-tier pricing as of 2026-05.)
- **Clarity** is a **session replay + heatmap appliance with no recording cap, ever, on the free tier.** Microsoft funds it as part of Bing/Edge product research; for users like you, it's effectively unlimited. Quality of replays is comparable to PostHog. Heatmaps (click maps + scroll maps + dead-click detection) are uniquely good. Aggregate retention is 13 months; replay retention is 30 days.
- **The practical play**: PostHog answers *"what happened, by how much, for whom"* (the metrics). Clarity answers *"what does this LOOK like for a real visitor"* (the qualitative replay), and is your **strong free-fallback** for replay capacity if any single page/campaign starts producing more than 5k+2.5k recordings/month — Clarity will keep recording while PostHog stops.
- **Cost of running both**: zero on free tier, two `<script>` tags, ~3 KB extra page weight gzipped (Clarity's bundle is small). The integration is in the privacy-banner consent gate so the user gives one yes/no, both tools obey it.

If a user insists on installing only one: I default to **PostHog** (the metrics layer is harder to bolt on later and powers the variant-tracking system in Stage 11). Clarity can always be added in 5 minutes anytime later. The reverse isn't true.

---

## Step 5b — Cloudflare DNS + (optional) API token

The skill works with any DNS provider — GoDaddy, Namecheap, Squarespace, Google Domains (now Squarespace), Route 53, Cloudflare, whatever. **Cloudflare is the strong recommendation.** Here's why I bring it up at onboarding rather than buried in Stage 4:

### Why Cloudflare is the recommendation

Everything below is on Cloudflare's free tier — there's no upsell I'm steering toward.

- **Free SSL/TLS cert, auto-renewed.** No Let's Encrypt cron job to babysit, no cert expiry pages. The cert just works the moment a domain proxies through Cloudflare.
- **Free DDoS protection at the edge.** A bot flood or a sudden traffic spike (good viral moment, bad scraper) gets absorbed by Cloudflare instead of hitting Railway and burning your monthly hosting allowance.
- **`cdn-cgi/trace` endpoint for geo detection.** This is what powers the geo-aware consent banner in Stage 8 — Cloudflare tells the visitor's browser their country code, no IP-geolocation library needed, no GDPR friction. **No other DNS provider gives us this.**
- **Fast DNS propagation.** Cloudflare TTL changes propagate in under a minute in practice. Other registrars routinely take 30–60 minutes, which means slow iteration when wiring up custom domains.
- **Redirect Rules (free tier).** Stage 10's short-URL setup uses a Redirect Rule to bounce `mayas.link/something` → `whop.com/whatever?ref=xyz`. Other providers force me to spin up a tiny Express server just to do redirects, which adds a Railway service the user pays for.
- **Email Routing (free tier).** Stage 5 uses this for catch-all forwarding (`anything@<domain>` → `<user>@gmail.com`) so the user doesn't lose mail addressed to addresses I haven't provisioned. Other providers require an MX-host signup somewhere.
- **One-click DNSSEC, HSTS, Always-Use-HTTPS, IPv6.** Each of these is a manual ticket with most registrars. On Cloudflare it's a checkbox.

If the user is on a **different DNS provider** and doesn't want to migrate: that's fine. The skill still works. **What changes:** every place Stages 4/5/10 would have done DNS edits autonomously, I'll instead generate exact click-paths and DNS-record values for the user to paste into their provider's dashboard. More interruptions, same final result. I do NOT push a Cloudflare migration on a user who already has a working setup elsewhere — domain migrations carry small risks (mail outage if MX records are skipped, downtime if the registrar lock isn't lifted properly) and the user is the right judge of whether the autonomous-DNS upside is worth the migration cost.

### Optional: Cloudflare API token (one ask, eliminates 3+ later interruptions)

If the user is on Cloudflare (or willing to migrate), I ask **once** for a scoped API token. With it, I can edit DNS records autonomously across:

| Stage | What I'd otherwise interrupt the user for |
|---|---|
| **Stage 4** (custom domain) | Adding the CNAME pointing the apex/www to Railway's edge |
| **Stage 5** (email/Resend) | Adding 3 DNS records (DKIM, SPF, return-path) for sender verification |
| **Stage 10** (short URLs, optional) | Adding the CNAME for the branded short-URL domain |

Without the token, each of those becomes a "please paste these records into your Cloudflare dashboard" pause, where I generate the exact records and the user clicks through Cloudflare's UI to add them. Net cost: ~5 minutes of interruption per stage. With the token: zero interruption.

**The ask** (only shown if the user picked Cloudflare or wants to migrate):

```
Cloudflare API tokens let me edit DNS autonomously. The token I'll ask
for is scoped — I'll need exactly these permissions, nothing more:

  - Zone → DNS → Edit (for the specific zones you list)
  - Zone → Zone → Read (so I can find the right zone)

I will NOT request:
  - Account-level permissions (so I can't create/delete zones)
  - Workers / R2 / Pages permissions
  - Email Routing permissions (you'll do that one manually — it's a click)

Do you want to:
  (a) Create a token now — I'll walk you through Cloudflare's "Create
      Custom Token" UI (~3 minutes), and we save the token to a local
      secrets file (.env, gitignored).
  (b) Skip — I'll generate paste-ready DNS-record instructions for each
      stage instead. You'll have 3 small interruption moments.
  (c) Add it later — start without; I can ask again at Stage 4 once
      DNS first matters.
```

**If the user picks (a)**, I provide click-by-click instructions for `https://dash.cloudflare.com/profile/api-tokens` → "Create Token" → "Custom token", with the exact permission rows to add. I never ask for the user's Cloudflare login credentials. **The user creates the token themselves and pastes the value into the chat OR into a file I name** (e.g., `<project>/.env.local` with `CLOUDFLARE_API_TOKEN=...`); the file is gitignored, and I never log the token in plaintext outputs.

**If the user picks (b) or (c)**, I save the choice (`cloudflare_token: "skip" | "deferred"`) and proceed without it.

I save the user's answer here as `dns_provider: "cloudflare" | "<other>"` and `cloudflare_token: "granted" | "skip" | "deferred"`.

---

## Step 6 — The questions

How many questions I ask depends on the onboarding mode (`all_upfront` vs `phase_by_phase`) and expertise (`non_coder` / `developer` / `expert`). All-upfront + non-coder mode is the longest path; expert + phase-by-phase is the shortest.

### Path A — `phase_by_phase` mode (4 essential questions, then start)

I ask only what I need to begin Stage 0. The rest is deferred to "I'll ask you when we get there."

**Question 1 — Brand + domain**:

```
What's the name of the brand or business this site is for?
And do you have a domain name picked out (or already registered)?
```

If the user has a domain, I auto-detect their DNS provider via `dig <domain> NS +short` and use it to plan Stage 4. If they don't have a domain, we'll figure it out in Stage 4 — for now, we'll plan to use Railway's staging URL.

**Question 2 — Source code**:

```
Where does the website code currently live?
  (a) I have a zip file from a website builder (Readdy, Webflow, Framer, etc.)
  (b) It's already in a GitHub repo
  (c) I want to start from scratch — I don't have any code yet
  (d) Something else (describe it to me)

If (a): which builder, and where's the zip file?
If (b): paste me the GitHub URL.
```

**Question 3 — Audience reach**:

```
Will any of the people visiting this site be located in the European Union,
United Kingdom, or European Economic Area? (This affects the cookie banner
and a couple of legal requirements — not whether we can launch.)
  (a) Yes — at least some EU/UK visitors
  (b) No — strictly North America / other non-EU regions
  (c) I'm not sure — assume yes to be safe
```

**Question 4 — Health adjacency**:

```
Does the website's content involve health, mental health, sleep, fitness,
medical advice, or biometric data in any way?
  (a) Yes — there's a health-related angle to what this brand sells
  (b) No — this is unrelated to health

(This drives whether I add a Consumer Health Data privacy page, which
is required for some U.S. states.)
```

**That's it for `phase_by_phase`**. I save these answers and start Stage 0. When later stages need a decision, I ask in context.

### Path B — `all_upfront` mode (full set; ~30–60 minutes)

In addition to the 4 above, I ask the questions below. **For non_coder users**, every question is yes/no or pick-from-list with explanations. **For developer/expert users**, I skip the explanations and accept terser answers.

**Section A — Project metadata**:
- Slug for the project (auto-suggested from brand name)
- GitHub repo location (existing? want me to create one?)
- Going to a real domain at launch, or staging URL only?
- **Brand prefix** for storage-key namespacing — see explanation below

#### What's a "brand prefix" and why I ask

The brand prefix is a short, lowercase, alphanumeric token (typically 2–6 characters) I use to namespace cookies and `localStorage` keys so the site's data doesn't collide with other sites the visitor has open. Examples:
- `mc_` for Maya's Consulting → cookie name `mc_consent_v1`, localStorage key `mc_affiliate_code`
- `acme_` for Acme Co. → `acme_consent_v1`, `acme_session_id`

It is NOT:
- the brand display name (`Maya's Consulting`)
- the project slug (`mayasconsulting-marketing`)
- the domain (`mayasconsulting.com`)
- a vendor key (`whop_`, `posthog_`, `stripe_`)

**Where it shows up in shipped code**:
- Stage 8 consent banner: `<brand-prefix>_analytics_consent`, `<brand-prefix>_consent_version`
- Stage 10 affiliate capture: `<brand-prefix>_affiliate_code` (cookie + localStorage)
- Anywhere else I write client-side persistence

**My auto-proposal logic**:
```
slug                = "mayasconsulting-marketing"
first_word          = "mayasconsulting"
proposal            = first 5 lowercase chars + "_"
                    = "mayas_"
```

I show the user my proposal and ask:

```
For storage-key namespacing on the visitor's browser, I'm going to use
the prefix "mayas_". This is invisible to visitors — it just keeps the
site's cookies / localStorage from colliding with other sites they have
open. Sound good? Or pick your own (2–6 lowercase chars, no spaces,
trailing underscore optional — I'll add it if you skip it).
```

If the user accepts (most do), I save `project.brand_prefix = "mayas_"` to skill config. If they suggest their own, I sanitize (strip spaces, lowercase, ensure trailing `_`, cap at 6 chars + `_`) and save.

**Section B — Source platform deeper-dive**:
- Has any cleanup been done already?
- (Critical for non_coder) Are any testimonials / team photos / "users" on the source site AI-generated or fictional? *(I flag this as a launch blocker if yes — FTC concern. We address in Stage 2.)*

**Section C — Forms** (discovery-driven; minimal user input here):

I do NOT ask the user to enumerate forms or list fields. Stage 0 reads the source code and produces `forms-manifest.md` with every form's path, fields, current submission target, and inferred purpose. The user reviews that manifest at Stage 5 and confirms routing decisions per form.

The only things I ask in onboarding for Section C are the integration-direction questions Stage 5 needs to plan for:

- **Existing ESP / list provider?** *"Do you have an existing email service provider where you already manage subscriber lists? (Mailchimp, Klaviyo, ConvertKit, Beehiiv, HubSpot, ActiveCampaign, MailerLite, Brevo, etc.) If yes, I'll research that provider's current API + MCP options and integrate at Stage 5. If no, I default to Resend Audiences (in-band with the transactional email account from Section D)."*
- **Existing CRM?** *"Do you have a CRM where form submissions should be routed? (HubSpot CRM, Salesforce, Pipedrive, Attio, Close, Folk, etc.) If yes, I research the CRM's API + MCP at Stage 5 and wire the integration. If no, skip."*
- **DB capture default**: *"By default, every form submission is also captured into the project's PostgreSQL DB on Railway as a safety net so no data is ever lost. Confirm OK, or opt out (only recommended if you've confirmed your chosen CRM captures every form's data completely)."*

If the user says "I don't know yet" to ESP/CRM, I defer to Stage 5 — when forms are actually being wired, there's a concrete decision moment.

I save: `forms.esp_provider`, `forms.crm_provider`, `forms.db_capture_enabled` (default `true`).

**Section D — Email**:
- Any transactional emails to send (welcome, verify, password reset)?
- Recommended ESP: Resend. Alternatives: AWS SES, Postmark, SendGrid, Mailgun. Default to Resend unless the user has an existing ESP relationship.
- Sending domain (must be a domain the user controls)

**Section E — Analytics**:
- Any existing PostHog or Clarity accounts to use? (If yes, I'll integrate; if no, I create them in Stage 6/7)
- Default install puts BOTH PostHog and Clarity on the site (see "Why install both" note in Step 5). Confirm or override:
  - `both` (default — metrics from PostHog, unlimited replay/heatmaps from Clarity)
  - `posthog_only` (cap recordings at 5k desktop / 2.5k mobile per month; can add Clarity later in 5 min)
  - `clarity_only` (replay + heatmaps only; no event-level metrics, no funnels, no flags — strongly discouraged unless cost-paranoid)
- For multi-site users: should this site share an analytics project with another site? (Recommended: yes for cross-site funnel tracking.)
- Recommended PostHog region: US (`us.posthog.com`) unless EU residency required.

**Section F — Privacy posture** (I auto-derive from Q3 + Q4 above):

I phrase this without legalese. The recommended posture comes from a simple decision tree:

| Q3 (EU visitors?) | Q4 (Health-adjacent?) | Recommended posture |
|---|---|---|
| Yes | Yes | "Strict": banner everywhere, opt-in everywhere |
| Yes | No | "Standard": banner in EU only, opt-out elsewhere |
| No | Yes | "Standard": still show banner per state laws like Washington MHMD |
| No | No | "Light": no banner, just respect Do-Not-Track header |

I tell the user the recommendation and ask if they want to override. (Override needed rare — most users accept the recommendation.)

**Section G — Membership / payments**:
- Is the site selling subscriptions, memberships, courses, or one-time access? (yes / no)
- If yes: which platform?
  - **Stripe** (default for general payments — most flexible, used by most SaaS/e-commerce)
  - **Whop** (recommended for content/community/course businesses — built-in affiliate program, marketplace exposure, course/community hosting; its robust API can also handle payments if you want to consolidate vendors)
  - **Skool** (community + courses; no native subscription webhooks as of 2026-05 → analytics integration via API polling, less mature path)
  - **Lemonsqueezy / Paddle / Gumroad** (similar webhook patterns to Stripe; good for digital downloads + simple subscriptions)
  - **Other** (describe; I'll adapt the generalized HMAC-webhook pattern from Stage 9)
- Tier names + pricing (e.g., "Plus $25/mo, Pro $39/mo with 7-day trial, Lifetime $319 once")

**Section H — Affiliate**:
- Running or planning an affiliate program? (yes / no / maybe later)
- If yes: default commission rate (varies by platform — Whop default is 30%, Stripe + a tracking layer like Trackdesk lets you set arbitrary; common starting range 20%–40%)

**Section I — Short URLs**:
- Do you want branded short URLs? (Useful for affiliate links AND for general tracking — campaign URLs, podcast show notes, QR codes, ad copy.)
- If yes: do you have a short-URL domain in mind? (Examples: `mayas.link`, `joins.mayasconsulting.com`, `goto.<brand>`)

**Section J — Variant tracking**:
- Want me to set up variant tracking for sequential CRO iteration? (Recommended yes — adds zero complexity until the user starts iterating, then is load-bearing.)
- Which pages should be tracked separately? (Default: home, about, pricing, contact, plus the main sales page if applicable.)

**Section K — Variant archive**:
- When you bump a variant, do you want to keep the old one viewable? Three options:
  - Local-only: free, just for the user reviewing later
  - Live archived URLs: each variant gets its own URL like `v1.mayasconsulting.com` (costs Railway resources)
  - Screenshot archive: I take screenshots after each deploy and save to cloud storage
  - Skip (default for non-coder)

**Section L — Object storage** (only ask if the site stores files — uploads, media library, generated images, screenshot archive, etc.):

```
Does the site need to store files anywhere — user uploads, a media library,
generated images, a screenshot archive, etc.?

  (a) Yes — PRIVATE files (user account-scoped, signed-link access only).
      Examples: user-uploaded documents, account avatars, downloadable PDFs
      tied to a logged-in session.
  (b) Yes — PUBLIC files (stable URLs, browsers and CDNs cache by URL).
      Examples: a podcast feed, an image gallery, a video lesson library,
      any media that's the same for all visitors and benefits from caching.
  (c) Yes — BOTH private + public.
  (d) Just variant screenshots from Stage 11 — minimal storage need.
  (e) No — no file storage needed (most simple marketing sites).
  (f) You decide — read what I've already told you and pick.
```

Recommendation logic I apply (full reasoning in `_internal/reference-object-storage.md`):

| Answer | Provider I provision |
|---|---|
| (a) Private | Railway Buckets — same invoice, free egress, presigned URLs handle access |
| (b) Public | Backblaze B2 — public buckets with stable cacheable URLs |
| (c) Both | Backblaze B2 for everything (consolidate; running two backends roughly doubles operational surface) |
| (d) Variant only | Railway Buckets (low traffic, private, same-invoice convenience) |
| (e) None | Skip; no provisioning |
| (f) You decide | I read saved config and recommend with rationale; default to Backblaze if the public-vs-private question is uncertain |

**Why default to Backblaze when uncertain**: Backblaze supports BOTH public and private buckets. If the user later realizes they need public URLs (e.g., adds a podcast or image gallery to a previously private app), Backblaze handles it without migration. The reverse — Railway Buckets → Backblaze migration when public URLs become needed — is a real re-upload + re-config exercise. Forecloses fewer future choices.

**The caching nuance**: public URLs (Backblaze) are stable like `https://media.<your-domain>/library/file-42.jpg` and browsers + CDNs cache by URL. Private buckets (Railway) require presigned URLs that change every request, so browsers re-download even identical bytes. For files users access once, this doesn't matter. For files served repeatedly across users (a media library — audio, video, images all qualify), the caching difference is real bandwidth + perceived-load cost.

---

## Step 7 — Save settings

When the questionnaire is complete (Path A or Path B), I summarize for the user:

```
Here's what I've got:

PROJECT:        Maya's Consulting (mayasconsulting-marketing slug)
SOURCE:         Webflow export at ~/Downloads/maya-export.zip
HOSTING:        Railway with custom domain mayasconsulting.com
EMAIL:          Resend, sending from mayasconsulting.com (will need to verify DKIM/SPF)
ANALYTICS:      PostHog (US) + Clarity, both new accounts I'll create in Stage 6/7
PRIVACY:        "Standard" — banner shows for EU visitors only
MEMBERSHIP:     None — this is a marketing site only
AFFILIATES:     Not now (you can add later)
SHORT URLS:     Not yet
VARIANTS:       Yes, tracking 5 pages

I'll save these answers so future skill runs can clone them.

Save to:
  (a) <project>/.skill-config.json (project-local; this project only)
  (b) <project>/.skill-config.json AND ~/.claude/skill-configs/mayasconsulting-marketing.json
      (cross-project library; future projects can reference this one)
  (c) Don't save — keep in memory for this session only

Which?
```

If saved, the file goes in `.gitignore` automatically (these contain identifying info even when secrets are kept separate).

The saved-config schema:

```json
{
  "schema_version": "1",
  "saved_at": "2026-05-08T22:00:00Z",
  "saved_with_skill_version": "2026-05-08",

  "modes": {
    "user_expertise": "non_coder",
    "onboarding_mode": "phase_by_phase"
  },

  "project": {
    "name": "Maya's Consulting Marketing Site",
    "slug": "mayasconsulting-marketing",
    "brand": "Maya's Consulting",
    "brand_prefix": "mayas_",
    "site_url": "https://mayasconsulting.com",
    "app_url": null,
    "github_repo": null
  },

  "origin": {
    "platform": "webflow",
    "artifact_type": "zip",
    "artifact_path": "~/Downloads/maya-export.zip",
    "cleanup_done": []
  },

  "hosting": {
    "deploy_target": "railway",
    "railway": {
      "project_id": null,
      "service_id": null
    },
    "dns_provider": "cloudflare",
    "cloudflare_token": "granted",
    "domain_status": "registered",
    "domain_migration_scenario": "fresh_launch"
  },

  "email": {
    "provider": "resend",
    "sending_domain": "mayasconsulting.com",
    "from_address": "Maya's Consulting <hello@mayasconsulting.com>",
    "notify_to": "hello@mayasconsulting.com",
    "domain_verified": false
  },

  "forms": {
    "discovered": [],
    "esp_provider": null,
    "esp_credentials_pasted": false,
    "crm_provider": null,
    "crm_credentials_pasted": false,
    "db_capture_enabled": true,
    "db_capture_table": "form_submissions",
    "per_form_routing": {}
  },

  "analytics": {
    "stack": "both",
    "posthog": {
      "enabled": true,
      "region": "us",
      "project_id": null,
      "share_with_other_sites": false,
      "mcp_installed": false
    },
    "clarity": {
      "enabled": true,
      "project_id": null,
      "mcp_installed": false
    },
    "proxy_path_slug": null
  },

  "privacy": {
    "posture": "standard",
    "eu_visitors": true,
    "health_adjacent": false,
    "consent_version": "2026-05-08-v1"
  },

  "membership": null,

  "affiliate": {
    "enabled": false
  },

  "short_urls": {
    "enabled": false
  },

  "variants": {
    "enabled": true,
    "tracked_pages": ["/", "/about", "/services", "/pricing", "/contact"],
    "archive_approach": "local_only"
  },

  "storage": {
    "needed": false,
    "use_case": "none",
    "provider": "none",
    "bucket_name": null,
    "is_public": false,
    "decided_by": null
  },

  "tools_present": {
    "node_version": "20",
    "github_cli": true,
    "railway_cli": false,
    "preview_mcp": true,
    "chrome_mcp": false
  },

  "setup_status": {
    "completed": ["github_account"],
    "deferred": ["railway_account", "posthog_account", "clarity_account", "resend_account", "domain_verification"],
    "deferred_to_stage": {
      "railway_account": 4,
      "posthog_account": 6,
      "clarity_account": 7,
      "resend_account": 5,
      "domain_verification": 5
    },
    "async_in_flight_at_save": []
  }
}
```

---

## Step 8 — Optional: enable autonomous verification (one-time setup)

This step is OPTIONAL and skippable. It only matters if the user wants me to verify changes (screenshots, network checks, console-error checks) without prompting them per-action.

**For non_coder users**: I describe this in plain language:

```
One quick optional setup — should I be able to take screenshots of your
site, check that things load correctly, and inspect what's running, all
without asking your permission each time?

(a) Yes — set me up to verify autonomously (I'll edit one config file once)
(b) No — ask me each time you want to verify something (more friction)
(c) Maybe later — leave as default for now

This affects how I verify work in Stages 6, 7, 8, 11, and 12. Either
choice works; (a) is just faster.
```

If yes, I run this autonomously:
1. I read `~/.claude/settings.json` (creating it if missing).
2. I check if a `permissions.allow` array already exists.
3. I add the verification tool names (Preview MCP read-only tools, plus optional UI-interaction tools if user picks Level 2 below) to the array, deduplicating.
4. I show the user a diff before saving (so they can see exactly what's changing).
5. After save, I read back the file to confirm valid JSON.

I default to **Level 1 (read-only autonomy)** for non_coder users:
- Pre-approves screenshots, network inspection, console logs, accessibility tree, server logs
- Tools that mutate state (clicks, form fills, JS execution) keep prompting per-use

For developer/expert users I offer **Level 2 (full autonomy)** which adds:
- Click / fill / eval pre-approved (lets me run end-to-end smoke tests without asking)

The user can switch levels later by editing `~/.claude/settings.json` directly, or by saying "give me back the prompts" / "switch to full autonomy" — I update the file accordingly.

---

## Step 9 — Hand off to Stage 0

```
Onboarding complete. Saved your answers to <project>/.skill-config.json.

Next: Stage 0 — I'll examine your source code and identify any dependencies
on the original platform (image CDNs, form endpoints, hardcoded URLs, etc.)
that I need to migrate before deployment. Estimated time: 5–10 minutes
of automated analysis, then I summarize what I found.

Ready to start Stage 0?
```

If yes → I open [stage-0-discovery.md](stage-0-discovery.md) and execute. The user doesn't need to read it; I drive.

If `phase_by_phase` mode and the user wants a break, I save state and resume next time.

---

## Quick mode-switch reference

The user can always say one of these to change behavior:

| User says | I do |
|---|---|
| "Switch to all_upfront" / "Let me answer everything now" | Switch mode, ask remaining questions, then run autonomously |
| "Switch to phase_by_phase" / "Stop asking so much" | Switch mode, defer remaining questions to relevant stages |
| "I'm a developer / expert" / "Skip the explanations" | Upgrade expertise tier; subsequent questions get terser |
| "Explain less" / "Just do it" | Move to expert mode for the rest of this session |
| "Explain more" / "I don't understand" | Move down a tier; provide more context |
| "Use defaults" / "Pick what's recommended" | I auto-pick everything safe; show the user a summary |

---

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Comprehensive onboarding flow extracted from Stage 0's interview section. |
| 2026-05-08 | Added Part 1B: interactive setup walkthrough with five tiers; setup_status tracking; permissions allowlist. |
| 2026-05-08 | Promoted URL shortening to its own concern; added Tier 4 walkthrough for domain purchase + Switchy connection. |
| 2026-05-08 | Cleanup: removed VPN entries; reframed Lighthouse as built-in DevTools. |
| 2026-05-08 | Expanded Cloudflare DNS section with full migration walkthrough + Always Use HTTPS toggle path. |
| 2026-05-08 | Expanded Tier 4 short-URL DNS setup with explicit per-record proxy mode + Cloudflare Redirect Rules walkthrough. |
| 2026-05-08 | Added autonomous-verification permissions Tier 2 sub-section. |
| 2026-05-08 | **Major reframe to non-coder-first audience.** Cut 1,240 lines to ~600. Replaced 56-question questionnaire with mode-aware path: `phase_by_phase` mode asks only 4 essential questions then starts; `all_upfront` mode asks the full set organized by section. Replaced privacy-tier picker with auto-derivation from two yes/no questions (EU visitors? Health-adjacent?). Compressed Tier 1B per-tier walkthrough into autonomous account-check (I run `gh auth status` / `railway whoami` / etc. and report; user only sets up what's missing). Compressed Step 8 permissions allowlist procedure to "I do it autonomously after asking once." Stripped all personal/brand-specific references; locked placeholder conventions. Removed all references to `.shared/` paths in favor of `<project>/.skill-config.json` + `~/.claude/skill-configs/`. |
| 2026-05-08 | Added Step 5b: Cloudflare DNS recommendation with the WHY (free SSL, free DDoS, `cdn-cgi/trace`, fast TTL, Redirect Rules, Email Routing — all free tier) and the optional API token ask that unlocks autonomous DNS edits across Stages 4/5/10. Added `cloudflare_token` to saved-config schema. Honest fallback documented: if user declines token or uses different DNS provider, I generate paste-ready DNS-record instructions instead. |
| 2026-05-08 | Added "Why install BOTH PostHog and Clarity" framing in Step 5: tools are complementary, not competing. PostHog free tier is 5,000 desktop + 2,500 mobile session replays/month; Clarity is effectively unlimited at any scale, making it the strong free-fallback past PostHog's recording cap. Section E now explicitly offers `both` (default) / `posthog_only` / `clarity_only` and discourages clarity-only unless extreme cost-paranoia. Schema gains `analytics.stack` field. |
| 2026-05-08 | Added Section L (Object storage) with Railway Buckets vs Backblaze B2 decision framework. Default for private files: Railway Buckets (free egress, same invoice). Default for public media or when public-vs-private is uncertain: Backblaze B2 (public buckets supported, forecloses fewer future choices). Includes the caching gotcha (presigned URLs defeat browser + CDN caching for repeatedly-served files). "You decide" mode reads saved config and recommends with rationale. Schema gains `storage` block with `decided_by` field for transparency about Claude-default vs user-stated choices. Full reasoning in `_internal/reference-object-storage.md`. |
| 2026-05-08 | Section C reframed: discovery-driven instead of interrogative. Stage 0's `forms-manifest.md` (extracted from source code) replaces the user-enumeration question. Section C now only asks the integration-direction questions (existing ESP? existing CRM? confirm DB capture default). Saved-config schema gains `forms` block with `discovered`, `esp_provider`, `crm_provider`, `db_capture_enabled` (default `true`), and `per_form_routing`. Default rule: capture every submission to PostgreSQL `form_submissions` table; opt-out gated on CRM-coverage confirmation. ESP/CRM integration uses research-first pattern (WebFetch vendor docs + WebSearch for MCP/SDK at integration time, not at skill-write time). Full framework in `_internal/reference-forms-and-persistence.md`. |
