# Operating Principles — Reference

> **🤖 This file is for Claude only.** Universal engineering and operating principles I follow when executing v2 stages. The same 10 principles also get scaffolded into the project's `<project>/CLAUDE.md` at Stage 1, so future Claude sessions on the user's project pick them up automatically without me having to re-explain.

## When to read this file

I read this file:
- At the start of any v2 session where I'm executing stages end-to-end.
- When I'm about to make a claim I should verify against a principle (e.g., "deploy succeeded" — check principle 1; "renamed X" — check principle 2).
- When the user asks "what rules does Claude follow?" — point them here AND at their project's `CLAUDE.md`.
- When I catch myself about to violate one — stop and re-read.

## Why these principles exist

Each one prevents a specific class of silent production breakage. None of them are aesthetic preferences — they're scar tissue from real failure modes. The Railway commit-hash gotcha shipped wrong-commit verifications until it was named. The cross-surface naming lock-step rule exists because a single rename in one repo broke production for 15 days. The plain-language rule exists because jargon in user-facing prose creates real friction.

## The 10 principles

### 1. Claude executes; user directs

**Default operating mode**: Claude does ALL the keystrokes — edits files via Edit/Write, runs CLI commands via Bash, pushes commits via git, verifies results via curl / build / test, reports concrete evidence. The user provides decisions (what to build, what voice, what tradeoffs) and a small number of inputs where the tool layer can't act on their behalf.

**The anti-pattern this rejects**: handing the user a list of steps to run themselves. "First, run `npm install`, then edit `foo.json` to add X, then redeploy" is wrong when Claude has the tools to do all three. If Claude has the tools, Claude uses them.

**Cases this principle covers**:
- File edits on code or config — Claude makes them; user reviews after
- CLI scripts (`npm`, build, deploy, git, etc.) — Claude runs via Bash; user doesn't open a terminal
- Triggering Railway redeploys — Claude pushes a commit; Claude verifies the new bundle via curl + commit-hash check (Principle 2)
- Variant changes / feature flag toggles — Claude reads the JSON, computes the change, commits, pushes
- Server restarts during development — Claude restarts via Preview MCP or equivalent; doesn't ask

**Rare exceptions** (cases where the user MUST act because the tool layer can't):
- Dashboard-only operations the MCP doesn't expose (vendor-specific UI without API: Cloudflare Email Routing setup, PostHog DPA signing in a browser, DocuSign, etc.)
- Auth flows requiring physical interaction (OAuth browser approves, MFA codes, password manager retrieval)
- Decisions on judgment calls (brand voice, copy phrasing, design taste, business strategy, pricing) — Claude gathers + presents options; user picks

**When you DO ask the user to act** (the rare exceptions), use plain-language commands:

- `Open Terminal and type:` `<the literal command>`, or
- `Go to <site> → <menu> → <button>`, then `<do this>`

Show the literal copy-pasteable command with full paths. Tell them what they'll see and what to do next. **Banned phrasings**: "OAuth flow," "interactive handshake," "session bootstrapping," "auth dance," "establish authentication," "initiate the flow."

| ✗ Avoid | ✓ Use |
|---|---|
| "Run the interactive browser flow for `railway login`" | "Open Terminal and type `railway login`. Browser opens — click Authorize." |
| "Initiate the OAuth handshake for Google Workspace" | "Open Terminal and type `python <path>/oauth_helper.py`. Follow the prompts in your browser." |
| "Establish the Cloudflare DNS API token authentication" | "Go to Cloudflare dashboard → My Profile → API Tokens → Create Token. Pick the 'Edit zone DNS' template. Copy the token. Paste it here." |

**Where v2 enforces this**: SKILL.md "Automation contract" section (canonical statement of who-does-what for every category); the "Common rationalizations to refuse" table; every stage's voice contract.

### 2. Verify deploys by commit-hash, not bundle-hash or deployment-id

Railway re-uses the deployment record in place across rebuilds. `latestDeployment.id` and `latestDeployment.createdAt` do NOT change between deploys. Only `latestDeployment.meta.commitHash` reliably changes. Polling on `id` will see "no change" forever even when deploys complete.

Bundle hashes also change every build (even doc-only commits, if a build step injects build-time tokens like `new Date()`). A poll waiting for "hash != old_hash" can catch an earlier commit's build and report success while your commit is still queued.

The correct pattern:

```bash
PUSHED_COMMIT=$(git rev-parse HEAD)
for i in {1..20}; do
  CURRENT=$(railway status --json | jq -r '.latestDeployment.meta.commitHash // empty')
  STATUS=$(railway status --json | jq -r '.latestDeployment.status')
  if [ "$CURRENT" = "$PUSHED_COMMIT" ]; then
    case "$STATUS" in
      SUCCESS) echo "Deploy succeeded."; exit 0 ;;
      FAILED|CRASHED) railway logs --tail 100; exit 1 ;;
    esac
  fi
  sleep 30
done
```

**Where v2 enforces this**: [`_internal/reference-railway-automation.md`](reference-railway-automation.md) (canonical pattern), [Stage 4](../stage-4-railway.md) (deploy verification).

### 3. Cross-surface naming lock-step

When a name is shared across more than one surface, renaming in one place without updating all consumers in the same session is silent breakage. "Surfaces" includes: multiple repos, Railway env vars, PostHog event names, DB column names, GSC property identifiers, Reoon webhook URLs, external dashboard allowlists, vendor product/plan IDs, CSS class names referenced by external tools.

Specific names that often cross surfaces:
- **Tier names** (free / plus / pro / lifetime) — code + Whop dashboard + PostHog event filters + marketing copy + pricing page
- **Feature gate keys** — code + admin UI + analytics breakdowns
- **Env-var names** — code + Railway dashboard + `.env.example` + every deployment guide
- **Route names** — code + sitemap + GSC + Cloudflare Redirect Rules + internal links
- **Schema field names** — DB + API types + frontend forms
- **External vendor identifiers** — Stripe product IDs, Whop plan IDs, Switchy slugs (agreement points between systems)
- **CSS class names** in external dashboard allowlists (Clarity Mask-by-element, etc.)
- **PostHog event names** (cross-dashboard joins depend on them matching)

Before any rename: grep across the project AND inventory every external surface that references the name. Apply lock-step in one session. **A 30-second grep is always cheaper than fixing 4 surfaces out of sync after the fact.**

**Where v2 enforces this**: [Stage 0](../stage-0-discovery.md) (initial surface inventory), [Stage 2](../stage-2-platform-migration.md) (verification grep for residual platform refs), [Stage 12](../stage-12-launch-checks.md) (pre-launch sitemap-vs-routes cross-check).

### 4. Never hardcode production domains

Source from `SITE_URL` / `APP_URL` env vars (or from `src/site-config.ts` if using v2's Astro path). One env-var change moves the site; hardcoded references would require code edits at every reference.

Concretely:

```typescript
// ❌ Don't
<a href="https://mayasconsulting.com/login">Login</a>

// ✓ Do (Vite path)
<a href={`${import.meta.env.VITE_APP_URL}/login`}>Login</a>

// ✓ Do (Astro path)
import { APP_URL } from "@/site-config";
<a href={`${APP_URL}/login`}>Login</a>
```

**Where v2 enforces this**: [Stage 1](../stage-1-scaffolding.md) (`src/site-config.ts` scaffold), [Stage 2 Step 8](../stage-2-platform-migration.md) (refactor hardcoded production domains), [Stage 13](../stage-13-content-hub.md) (schema components import from site-config).

### 5. Push after every meaningful commit

Don't accumulate local commits that aren't on GitHub. Pushing protects against:
- Laptop death between commit and push
- Cross-device session handoff (the next session you open elsewhere needs to see the commits)
- Railway auto-deploy triggers (it watches the remote, not local)
- Scheduled tasks that re-ping GSC after a content publish (they read the remote)

The discipline is simple: every `git commit` that produces a meaningful change is followed by `git push` in the same turn.

**Where v2 enforces this**: every stage's Outputs section ends with "commit + push."

### 6. Surface real problems early

When you hit a genuine blocker, tell the user immediately and offer options. Don't proceed with a hidden workaround that creates technical debt.

**Real problems worth surfacing**:
- A contradiction in stated requirements (the user asked for X earlier; this new request conflicts)
- A factual error in copy that would create legal risk
- A vendor behavior that diverges from documented behavior (e.g., Whop webhook payload missing a field the docs promise)
- A dependency that doesn't install / build / deploy as documented

**Not real problems** (proceed silently):
- Pre-existing test failures unrelated to the current task
- Lint warnings that are stylistic
- Files the user has uncommitted changes to (don't stage, don't escalate)

**Where v2 enforces this**: "Common rationalizations to refuse" table in SKILL.md; stage verification gates that stop on failure rather than auto-advance.

### 7. Defer to provider taxonomies before inventing your own

When integrating with external providers (ad platforms, payment processors, CRMs, analytics tools, email senders), use the provider's existing taxonomies, macros, event names, and field naming verbatim. Don't translate provider values to your own convention — the friction compounds with every campaign / event / record.

**Why**:
- Provider systems are battle-tested at scale; their taxonomies reflect real distinctions worth preserving (e.g., Meta's per-placement values exist because placement actually matters for performance and is built into their bidder).
- Forcing your own taxonomy on top creates manual reconciliation work that compounds with every campaign.
- Provider macros (Meta's `{{placement}}`, Google Ads `{network}`, TikTok's `{{placement_name}}`) are designed to do work for you automatically; recreating that by hand throws away free leverage AND can break the provider's own optimization (e.g., one ad per placement breaks Meta's joint bidding model).

**Practical applications**:
- Use Meta's `{{placement}}` macro for sub-placement; adopt `instagram_reels` verbatim (don't translate to `ig-reels`)
- Accept Whop's webhook event names as-is (`subscription_started`, not renamed to `subscription_created`)
- Adopt Google Ads `{network}` macro for sub-channel rather than splitting at ad-creation
- Use Resend's `emails.send()` shape rather than building a custom abstraction that erases their field names

Only invent your own convention when the provider's system is genuinely unavailable, opaque, or wrong for your use case. When you do, document the trade-off explicitly.

**Where v2 enforces this**: [Stage 5](../stage-5-email-resend.md) (ESP integration — research-first pattern; use vendor's current API), [Stage 9](../stage-9-membership-optional.md) (membership webhooks — accept Whop's payload shapes), [Stage 10](../stage-10-affiliate-optional.md) (affiliate macros — adopt platform macros verbatim).

### 8. Wire Slack notifications for ops-relevant events

When shipping a feature that produces ops-relevant signals, evaluate whether it warrants a Slack notification and wire it. Don't ship a notification-worthy feature without surfacing it; don't scaffold a notification function without wiring it (gives false sense of coverage).

**Hook in when the event is**:
- Revenue or growth signal worth seeing in real time (signup, paid conversion, plan change, churn, affiliate-driven sale)
- Critical operational state change (server restart, DB reconnection, threshold crossing)
- Security event requiring attention (account lockout, repeated failed payment, MFA reset)
- Async job completing or failing (long-running TTS, scheduled task fails, batch operation)
- Threshold the user should know about within an hour

**Skip Slack for**:
- Every API call or DB query (use logs / PostHog instead)
- High-volume routine events (renewals, daily logins, pageviews)
- Events already reaching the user through another canonical channel (e.g., Whop's checkout already emails the customer + fires the webhook + lands in PostHog — don't ALSO Slack unless there's an ops question only Slack solves)

**Pattern**: fire-and-forget, production-only (short-circuit on `NODE_ENV !== "production"`), webhook URL from env var.

**Where v2 enforces this**: [Stage 5](../stage-5-email-resend.md) (Reoon 80%/100% threshold alerts — canonical implementation), [`_internal/reference-quota-monitoring.md`](reference-quota-monitoring.md) (generalized template for any vendor with a cap).

### 9. Don't claim "locked in" or "deferred" without the canonical doc current

If you're about to say any of these — or anything semantically equivalent — the canonical project doc must already reflect the current state:

- "X is locked in"
- "Architecture is captured / settled / decided"
- "We can defer / wait on / save X for later"
- "When you're ready, we just <do Y>"
- "Future Claude can pick this up by reading <doc>"
- "It's all documented"

**The standard**: if the user were to ignore Claude entirely and only read the canonical doc cold, would they get the full current picture without surprises, gaps, or stale assumptions? If not, the doc isn't ready for "locked in" claims.

**How to apply**:
1. **Before** claiming "locked in" / "deferred," identify the canonical doc for the topic (architecture spec, plan, runbook, or memory note).
2. **Read it cold** — as if you were a fresh contractor seeing it for the first time. Does it reflect current state?
3. **If stale, update it FIRST.** Don't make the claim until the canonical doc is current.
4. **Single source of truth** — don't fragment findings across many commits or many files. Consolidate into the canonical doc; let memory + commits be pointers to it.
5. **Then** make the claim.

The failure mode this prevents: a long-running effort accumulates findings across many sessions, each session updates its own working notes, but no one consolidates into the canonical doc. Future sessions read the canonical doc cold, see "the architecture is locked in," and operate on stale assumptions.

**Where v2 enforces this**: [Stage 14](../stage-14-cornerstones.md) (cornerstones aren't "published" until content-roadmap doc is current), [Stage 12](../stage-12-launch-checks.md) (pre-launch verification cross-checks every prior stage's claims against shipping reality).

### 10. Backup discipline for production sites

Production-bound secrets (`.env` files, `.secrets/`, API keys, OAuth tokens, the project's working memory) live in encrypted off-machine backup. Loss of the machine = full rebuild without it.

**The minimum**:
- Daily scheduled task that tars `.secrets/`, encrypts with a passphrase from password manager, uploads to durable storage (Backblaze B2 / S3 / R2)
- Encryption passphrase stored in the password manager, NOT in the backup (circular)
- Restore procedure documented in `docs/restore-procedure.md` so a future-user (or future-you) can rebuild without context

**Test the restore once before relying on it.** A backup that never round-trips is a backup that probably doesn't work.

**Where v2 enforces this**: [Stage 12](../stage-12-launch-checks.md) (pre-launch checklist includes "backup plan documented?"). Recommended (not required): scaffold the daily backup script as part of post-launch polish work.

## How these principles get into the user's project

Stage 1 scaffolds a `<project>/CLAUDE.md` at the project root containing the same 10 principles in shorter form. Claude Code auto-loads `CLAUDE.md` from the project directory on every session start, so the principles persist across all future Claude sessions working on that project — without me having to re-explain them and without the user having to remember.

The file is the user's to extend. As project-specific conventions emerge ("always use feature flag X before deploying behavior Y," "every commit message starts with `feat:` / `fix:` / `chore:`"), they add them to the bottom of the file. The starter rules give the floor.

## If a new principle emerges

If a NEW production failure mode surfaces during execution and warrants a principle:

1. Add it here with rationale + where v2 enforces it
2. Add a shorter version to the `<project>/CLAUDE.md` template in [Stage 1](../stage-1-scaffolding.md)
3. Add the same to any existing user projects via a one-liner: "I noticed <X> would have prevented <failure>; want me to add Principle N to your project's CLAUDE.md and to v2's template?"
4. Commit the v2 update + push

The principles list is intentionally small. New entries should reflect a real failure pattern, not a hypothetical concern.
