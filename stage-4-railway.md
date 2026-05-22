# Stage 4: Railway Deploy (production + Basic Auth staging service)

**Last verified: 2026-05-21.** Re-verify before this stage if older than 60 days. Search: `Railway MCP server <year>`, `Railway CLI documentation`. Read the official docs at `docs.railway.com/ai/remote-mcp-server` and `docs.railway.com/cli/login`.

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) deploy the site to Railway. The user's only actions: (1) one-time browser approve for Railway MCP OAuth, (2) one-time browser approve for `railway login` CLI OAuth, (3) confirming the custom domain choice when applicable. After those approvals, every Railway operation in this stage AND in subsequent stages 5–10 (where I'll be setting env vars on the user's behalf) is fully autonomous.

> **Upgrading from v1?** See [CHANGELOG.md](CHANGELOG.md). This stage now creates a second Railway service for HTTP Basic Auth staging, and the deploy-verification polling pattern was corrected (`commitHash`, not `latestDeployment.id`).

## The staging service pattern

After production is deployed and verified working (the existing v1 Stage 4 flow), I create a second service:

```bash
# Variant 1: CLI (preferred for repeatability)
railway service create <project-slug>-staging
railway link --service <project-slug>-staging
railway up --detach

# Variant 2: GraphQL via API token (if MCP unavailable)
curl -X POST "https://backboard.railway.com/graphql/v2" \
  -H "Authorization: Bearer $(cat .shared/secrets/railway-api-token.txt)" \
  -H "Content-Type: application/json" \
  --data '{ "query": "mutation { serviceCreate(input: { projectId: \"<project-id>\", name: \"<project-slug>-staging\", source: { repo: \"<github-org>/<repo>\" } }) { id name } }" }'
```

I then set env vars on staging that differ from production:

```bash
# Generate a strong random password for staging Basic Auth
STAGING_PASS=$(openssl rand -base64 18 | tr -d '/+=' | head -c 20)
echo "$STAGING_PASS" > .secrets/staging-basic-auth-pass.txt

# Set on staging service only
railway --service <project-slug>-staging variable set BASIC_AUTH_USER="<your-staging-username>"
railway --service <project-slug>-staging variable set BASIC_AUTH_PASS="$(cat .secrets/staging-basic-auth-pass.txt)"

# Reuse most other env vars (RESEND_API_KEY, REOON_API_KEY, etc.) — staging shares
# the same credit pool with production. Worth verifying which env vars are
# safe to share vs. should be staging-only (typically: SLACK_*_WEBHOOK_URL should
# be UNSET on staging so no real-team Slack alerts fire from test traffic).
```

I store the staging password in `.secrets/staging-basic-auth-pass.txt` (gitignored) and tell the user the URL + credentials in plain language:

```
Staging service is live at:
  https://<project-slug>-staging-production.up.railway.app

To access:
  Username: <your-staging-username>
  Password: <copy from .secrets/staging-basic-auth-pass.txt>

Browser will prompt on first visit; auth persists for the session.

Note: staging shares the same git branch as production by default. If you
want staging on a different branch (e.g., `staging` or `next`), I can wire
that up — but for most workflows, same-branch + Basic Auth is the right
default. The gating is between "deployed but not yet announced" vs "live
to the world," not between code branches.
```

## Deploy verification using commitHash (not id)

Railway re-uses the deployment record in place across rebuilds — `id` and `createdAt` do NOT change between deploys. Only `meta.commitHash` reliably changes. This is Principle 1 in [`_internal/reference-operating-principles.md`](_internal/reference-operating-principles.md); the full gotcha detail and the underlying GraphQL response shape live in [`_internal/reference-railway-automation.md`](_internal/reference-railway-automation.md). The correct polling pattern:

```bash
# After git push:
PUSHED_COMMIT=$(git rev-parse HEAD)
echo "Polling for deploy of commit $PUSHED_COMMIT..."

# Poll every 30s for up to 10 min
for i in {1..20}; do
  CURRENT_COMMIT=$(railway status --json | jq -r '.latestDeployment.meta.commitHash // empty')
  if [ "$CURRENT_COMMIT" = "$PUSHED_COMMIT" ]; then
    STATUS=$(railway status --json | jq -r '.latestDeployment.status')
    echo "Deploy of $PUSHED_COMMIT is in status: $STATUS"
    if [ "$STATUS" = "SUCCESS" ]; then
      echo "Deploy succeeded."
      exit 0
    fi
    if [ "$STATUS" = "FAILED" ] || [ "$STATUS" = "CRASHED" ]; then
      echo "Deploy failed. Fetching logs..."
      railway logs --tail 100
      exit 1
    fi
  fi
  sleep 30
done
echo "Timed out waiting for deploy of $PUSHED_COMMIT."
exit 1
```

**Common confusion**: when the deploy completes successfully and you re-deploy, `latestDeployment.id` stays the same in many Railway API responses. Don't use `id` as a change-signal. `meta.commitHash` is the only field that reliably changes per deploy.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 4 needs:
- `project.slug` — used as the Railway project name. Auto-derived if missing.
- `project.site_url` — set as the `SITE_URL` env var on the service. If unset and the user said "staging URL only is fine," I substitute the Railway-generated `*.up.railway.app` URL once the service exists.
- `project.github_repo` — used by Railway's "Deploy from GitHub repo" flow. Stage 1 should have populated this; if missing, I prompt now.
- `hosting.dns_provider` — drives whether the custom-domain step uses the Cloudflare API path or generates paste-ready DNS instructions for another provider. If unset, default to "manual_paste" (the safer fallback).
- `hosting.cloudflare_token` — if `dns_provider === "cloudflare"` and this is `"granted"`, I do DNS edits autonomously across this stage and Stages 5 + 10. If `"deferred"` or `"skip"`, I generate paste-ready DNS records instead. If both `dns_provider` and `cloudflare_token` are unset, I ask the Cloudflare-token question now (the user can decline; the work just gets ~5 minutes of paste-instruction interruption later).
- `tools_present.railway_cli` — if false, I install via `npm install -g @railway/cli` as the first action.

## What I'm doing in this stage

Deploying the Express + Vite site from Stages 1–3 to Railway, with all build-time env vars set BEFORE the first build runs. After this stage:

- The site is live at a `*.up.railway.app` staging URL (or the user's custom domain)
- Foundational env vars are set on the Railway service (`PORT`, `SITE_URL`, `APP_URL`, `RAILWAY_CHECKOUT_DEPTH`)
- The Railway CLI is authenticated for ongoing operational use across the rest of the build

**Time**: 10–20 minutes. The DNS-cutover variant (if migrating an existing domain) takes longer due to async DNS propagation.

## Prerequisites

- Stage 3 complete (Express server runs locally; verifications passed).
- Repo pushed to GitHub.
- The user has a Railway account (free dev tier is fine). If onboarding flagged this as deferred, I prompt them to create one now: `railway.app/signup` → email confirm → done.

## Automation surface I have available (verified 2026-05)

Three layers I can use, in order of automation depth:

### Railway Remote MCP — `https://mcp.railway.com`

OAuth browser login (one approve click). Tools available: `list-projects`, `create-project`, `list-services`, `redeploy`, `accept-deploy`, `whoami`. **NOT available**: env-var set/get, custom-domain attach, service-create-from-GitHub-repo, deployment-log retrieval.

### Railway CLI — `npm install -g @railway/cli`

Same OAuth browser login (`railway login`). Covers everything the MCP doesn't: env vars, custom domains, log streaming, GitHub-repo service creation.

### Railway GraphQL API — `https://backboard.railway.app/graphql/v2`

Bearer-token auth (`RAILWAY_TOKEN` from dashboard). I almost never use this from inside a skill — MCP + CLI handle 99% of interactive work. Reserved for non-interactive automation (CI scripts, scheduled tasks).

**My default path: MCP + CLI hybrid.** I use the MCP for project/service discovery and redeploys, and shell out to the CLI for env-var and domain operations. The user clicks "Approve" twice in browser tabs (once each), and I run autonomously after that.

**For expert-mode users** I offer alternatives: CLI-only (skip MCP setup) or manual-dashboard (no CLI either). Most users — especially `non_coder` mode — get the hybrid path without being asked to choose.

## My execution sequence

### Step 1: I authenticate the Railway CLI + MCP

I check current state via `railway whoami`. If not authenticated, I run `railway login`, which opens a browser tab for the user to click "Approve". After approval, the CLI caches the token at `~/.railway/` — every subsequent CLI call I make is autonomous.

For the MCP, I check if it's configured in the user's Claude Code instance. **The Railway MCP may not appear in Claude Code's built-in Connectors picker** — this is normal and expected. It's added as a Custom Connector via the SSE endpoint `https://mcp.railway.com`. Two paths to add it:

- **Path A (vendor-driven; usually easier)**: I open Railway's docs page that has the "Add to Claude Code" button. The user clicks it, Claude Code prompts to approve, done. URL: `docs.railway.com/ai/mcp-server` (verified 2026-05).
- **Path B (manual)**: I add the entry to the user's MCP config directly — server URL `https://mcp.railway.com`, transport `SSE`, auth `OAuth`. The user clicks "Approve" in a browser tab on first call.

I prefer Path A when available because it eliminates a config-file edit. If the user's Claude Code version is older and doesn't support the vendor-driven add flow, Path B always works.

If the CLI isn't installed yet, I run `npm install -g @railway/cli` first.

### Step 2: I create the Railway project + service

#### Tool surface (verified 2026-05 against `docs.railway.com/ai/mcp-server` and `docs.railway.com/cli`)

| What | Tool | Notes |
|---|---|---|
| Create project + auto-link to current dir | MCP `create-project-and-link` | Returns `projectId`. Replaces older two-step `create-project` + manual `railway link`. |
| List services in a project | MCP `list-services` | Returns array of service objects with IDs. |
| Link a service to current dir | MCP `link-service` | Subsequent `railway up` / `railway logs` operate on the linked service implicitly. |
| Deploy code from current dir | CLI `railway up` (or MCP `deploy`) | CLI is the canonical path because it streams build output live; MCP `deploy` works but is fire-and-forget. |
| Set env vars (non-secret) | MCP `set-variables` OR CLI `railway variable set KEY=value` | Either works. |
| Set env vars (secret) | **CLI only** — `railway variable set KEY="$(cat .secrets/foo.txt)"` | Shell substitution keeps the secret out of my chat context. The MCP path requires reading the file content into my context first, which is worse from a leak-defense standpoint. |
| Stream build logs | CLI `railway logs --build` | The MCP's `get-logs` returns a snapshot; CLI streams. |
| List variables | CLI `railway variable list` (or MCP `list-variables`) | `railway variable list --json` for structured output. |
| Add custom domain | CLI `railway domain example.com` | MCP `generate-domain` is for `*.up.railway.app` subdomains, NOT custom domains. |
| Check deployments | CLI `railway deployment list` | Status of recent deploys. |

**The principle**: MCP for orchestration where the value is non-sensitive (project create, service list, public env vars, log fetch). CLI for the "push code + watch build" loop and for setting secrets. Both surfaces are first-class — I pick per-action based on what each does best.

#### Project creation

```
mcp__railway__create-project-and-link { name: "<project-slug>" }
```

This creates the project AND links the current directory to it in one step — equivalent to the old `railway init` + `railway link` flow. Returns `projectId`, which I save to `<project>/.skill-config.json` under `hosting.railway.project_id`.

#### Service creation (via dashboard, then MCP discovery)

For service creation, **I use the Railway dashboard's "Deploy from GitHub repo" flow** rather than the CLI's `railway service create` — the dashboard path is more reliable for the GitHub-repo wiring (the dashboard handles the GitHub OAuth, repo permission grant, and webhook installation in one click flow that the CLI doesn't fully replicate). Before sending the user to the dashboard, I snapshot the existing services in the project so I can identify the new one afterward:

```
mcp__railway__list-services { projectId: "<from-create-project-and-link>" }
```

I save the resulting list as `services_before` (an array of service IDs).

Then I tell the user:

```
Railway is open. Please:
  1. Click "+ Create" → "GitHub Repo"
  2. Authorize Railway's GitHub app if prompted
  3. Select the repo: <github-username>/<repo-slug>
  4. Click "Deploy"

Railway will start the first build. It'll fail because env vars aren't
set yet — that's expected. I'll fix that next.

Just say "done" when you've clicked Deploy. I don't need you to copy the
service ID — I'll auto-detect it from the project.
```

Once the user confirms, I auto-detect the service ID and link it to the current directory:

```js
// Snapshot diff: find the new service that wasn't in services_before
const after = await mcp.railway.list_services({ projectId });
const newService = after.find(s => !services_before.includes(s.id));
if (!newService) throw new Error("No new service detected — is the deploy click stuck?");
const serviceId = newService.id;

// Link the new service to the current dir so railway up / logs / variable set
// all operate on it implicitly without needing serviceId on every call.
await mcp.railway.link_service({ serviceId });
```

If `services_before` was empty (this is the first service in the project), I just take the only service ID returned. If two services were created (rare; e.g., user clicked Deploy twice), I report both IDs and ask the user to pick the one matching the GitHub repo.

**I save `serviceId` to `<project>/.skill-config.json` under `hosting.railway.service_id`** so it persists across stages (Stage 5 sets `RESEND_API_KEY` on this serviceId; Stage 6 sets `VITE_POSTHOG_*`; Stage 9 sets webhook secrets; etc.). The link-service call also persists across CLI invocations, so `railway up` and `railway variable set` in subsequent stages don't need the serviceId on the command line.

For `expert` mode users, I offer the CLI alternative: `railway init` + `railway link` + `railway add --repo <user>/<slug>` — same effect, runs without leaving the terminal. After the CLI path, I run `railway status --json` to capture the service ID and save it to skill state.

### Step 3: I set the foundational env vars

⚠️ **Critical invariant from SKILL.md #1**: Vite bakes `VITE_*` env vars at build time. If they're missing when Railway runs `npm run build`, the deployed bundle is silently dark. I set them BEFORE letting any build complete.

Foundational vars I set in this stage:

| Variable | Value | Purpose |
|---|---|---|
| `PORT` | `3000` | Stage 3's `server.js` binds to `0.0.0.0:$PORT`; Railway requires this |
| `SITE_URL` | `https://<your-domain>` | Used by server-side code in `server.js` and later stages |
| `APP_URL` | `https://app.<your-domain>` | 301 redirect target if migrating off a PWA-served domain; can equal `SITE_URL` for fresh launches |
| `RAILWAY_CHECKOUT_DEPTH` | `0` | Avoids shallow-clone bug for variant labels in Stage 6 (without this, `git rev-list --count HEAD` returns `1` silently and Stage 6's variant tagging breaks) |
| `DEPLOY_TIMEZONE` | `UTC` (default) or any IANA tz | Stage 6 reads this for date-formatting in events. UTC is safe everywhere; only override if the user wants dashboard wall-clock days aligned to a specific region (e.g., `America/New_York` if the team is East Coast and reads daily reports by local day boundaries). |

The remaining env vars (`VITE_POSTHOG_KEY`, `RESEND_API_KEY`, etc.) are set in their respective stages. I'll set them autonomously when those stages run.

I run the bulk-set:

```
railway variable set PORT=3000 SITE_URL=https://<user-domain> APP_URL=https://app.<user-domain> RAILWAY_CHECKOUT_DEPTH=0 DEPLOY_TIMEZONE=UTC
```

(If the user said they want a non-UTC timezone in onboarding, I substitute that IANA name for `UTC`. If they said nothing about timezone, UTC is the default — it's the safe choice that works for any global audience and only matters once dashboards are interpreted by humans.)

Then I verify:

```
railway variable list
```

The output lists everything currently set. I confirm the four are present.

### Step 3b: Reference for setting secrets in later stages

This is documentation for me to use across Stages 5–10 when secret values arrive. I include it here in Stage 4 because that's where the env-var pattern is canonically established.

**The secrets-file pattern**: when the user has a secret to provide (Resend API key, PostHog key, Whop webhook secret, etc.), I ask them to paste it into a project-local file rather than into chat. The file lives at `<project>/.secrets/<vendor>-key.txt` and is gitignored. Then I read the file and set the Railway var without the value entering my chat history:

```
railway variable set RESEND_API_KEY="$(cat .secrets/resend-api-key.txt)"
```

The `$(...)` shell substitution happens before `railway` sees the args, so the secret enters the process via the substitution rather than appearing in my tool-call log as a literal value.

**For fully blind operation**: the user runs `echo "<secret>" > .secrets/resend-api-key.txt` once locally (the secret never reaches my context window), then I do the rest autonomously. This is the recommended path for high-sensitivity keys.

I create the `.secrets/` directory at the project root and add it to `.gitignore` automatically when the user provides their first secret. The directory exists alongside `.skill-config.json` (also project-local + gitignored).

### Step 4: I trigger the first successful build

After the foundational env vars are set, I redeploy the service so the build picks them up. The service is already linked to the current directory from Step 2's `link-service` call, so:

```bash
railway up
```

streams the build logs while the build runs. If I need a non-streaming, fire-and-forget redeploy (e.g., from a script), the MCP path is:

```
mcp__railway__deploy
```

— which deploys the linked service and returns immediately. I default to `railway up` because the streaming output catches build failures faster and the build-time `VITE_*` env-var trap (Invariant #1 in `_internal/claude-invariants.md`) is something I want to surface during the build, not after.

I monitor the build via `railway logs --build`. Expected outcome: build succeeds, no `undefined` value warnings (those will appear in Stage 6 logs once `VITE_POSTHOG_KEY` is added; expected behavior).

If the build fails, I diagnose from logs and report:
- "Build script not found" → check `package.json` `scripts.build`
- "Out of memory" → typically AI-generator-bloat issue; revisit Stage 2's dep cleanup
- Type-check errors → fix and re-deploy

### Step 5: I generate a staging domain and verify the site loads

```
railway domain
```

This returns a URL like `<project-slug>-production.up.railway.app`. I curl it to verify:

| Check | What I do | Pass criteria |
|---|---|---|
| HTTP response | `curl -I https://<railway-url>/` | 200 with `x-railway-edge` header |
| HTML loads | `curl -s https://<railway-url>/` | Body contains expected `<title>` from Stage 1 |
| Server logs | `railway logs` | Stage 3's startup lines visible |
| No 502 errors | DevTools Network tab via Preview MCP if available | All requests succeed |

If the build succeeded but the site shows a blank page, the most common cause is `OUT_DIR` mismatch in `server.js` (the file points at `out/` but Vite's actual output is `dist/` or vice versa). I read `vite.config.ts` and `server.js`, fix the mismatch, and redeploy.

### Step 6 (optional): I attach a custom domain

This is the cutover moment. **For fresh launches**, this is straightforward — attach the domain, configure Cloudflare, done. **For domain migrations** (moving an existing domain off a previous host or off a PWA), I follow the DNS cutover protocol below carefully.

#### Fresh launch (most projects)

```
railway domain add <user-domain>
```

Railway provides a CNAME target. I tell the user the exact records to add at their DNS provider (Cloudflare or otherwise). For Cloudflare specifically:

```
At Cloudflare → Domains → <user-domain> → DNS → Add record:
  Type:   CNAME
  Name:   @ (apex) or "app" (subdomain) — depends on the user's domain plan
  Target: <railway-cname>.up.railway.app
  Proxy:  ON (orange cloud) — Cloudflare handles SSL + DDoS
```

I wait for DNS propagation (usually 5 minutes, can be hours), then verify via `curl -sI https://<user-domain>/`.

#### Cloudflare-side configuration for the main domain

For any domain on Cloudflare, I walk the user through these toggles (or do them via Cloudflare API if they've granted me a token):

- **SSL/TLS mode**: SSL/TLS → Overview → set to **"Full (strict)"** (Railway provides a valid origin cert; Full strict prevents downgrade attacks)
- **Always Use HTTPS**: SSL/TLS → Edge Certificates → scroll down → toggle **"Always Use HTTPS"** ON (forces HTTP → HTTPS at the edge for any proxied subdomain)
- **Automatic HTTPS Rewrites**: same page → toggle **"Automatic HTTPS Rewrites"** ON (rewrites `http://` URLs in the user's HTML/CSS to `https://` automatically)
- **Minimum TLS Version**: same page → set to **TLS 1.2** at minimum (TLS 1.3 if the user's audience is modern)

#### DNS cutover protocol (only if migrating an existing domain)

This applies when the user's domain currently points at another host (their old setup, a previous PWA, etc.) and we're moving it to Railway.

**T-minus 24–48 hours**: I have the user lower their existing DNS TTL on the relevant records to 60 seconds. This shrinks the rollback window if something goes sideways.

**Cutover day**:
1. I have the user detach the domain from the OLD host (if applicable)
2. I attach the domain to the new Railway service via `railway domain add`
3. I tell the user the exact CNAME to update at their DNS provider — apex or subdomain, **PROXIED (orange cloud) for the main site domain**. Note: this is DIFFERENT from the SHORT URL domain in Stage 10, which is **DNS-only (gray cloud)** because Switchy provisions its own SSL. The two are easy to confuse.
4. I apply the Cloudflare-side configuration above (SSL Full strict, Always Use HTTPS, etc.)

**Verify within minutes** (because of low TTL):
- `curl -sI https://<user-domain>/` → expect 200 with `x-railway-edge` header
- `curl -sI http://<user-domain>/` → expect 301 redirecting to `https://`
- Browser test (incognito) confirms the site renders correctly

**Service worker tombstone verification** (only if Stage 3 wrote one): I have the user open the OLD URL on a phone/laptop that previously had the PWA installed. Console should show "[sw] tombstone — no SW registered" within seconds. The PWA icon, if launched, briefly opens the marketing site and redirects to the new app domain. If the tombstone doesn't trigger, I diagnose (could be a cached SW that hasn't checked for updates yet — short cache headers from Stage 3 force the update on next visit).

**External dashboard updates**: I tell the user which external services need their URL updated:
- Google Cloud Console OAuth callback URLs (if migrating an auth-using app)
- Whop / Stripe / etc. webhook URLs
- Slack OAuth + event subscription URLs
- Email DKIM/SPF if the sending domain changed
- Ad platform pixel URLs

I provide click paths for each that the user can follow, since these vendors don't have universal APIs I can drive.

#### Rollback plan

At any point during cutover, the user can reattach the domain to the old host. Users see the old service again within minutes (because of the lowered TTL). The new Railway service stays live at its `*.up.railway.app` URL — not lost, just not the apex. I document this rollback option upfront so the user feels safe proceeding.

### Step 7 (optional): I add a Railway config file

Some projects benefit from explicit build/deploy overrides. Most Vite + React projects don't need either — Railway auto-detects correctly.

If needed, I write `railway.json`:

```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": { "builder": "NIXPACKS", "buildCommand": "npm run build" },
  "deploy": {
    "startCommand": "npm run start",
    "healthcheckPath": "/",
    "healthcheckTimeout": 100
  }
}
```

Or a `nixpacks.toml` for finer-grained control of the Nix build environment.

I only add these if a build issue surfaces or the user has specific requirements.

## Verification (autonomous)

I run all of these myself before declaring Stage 4 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Service is up | `mcp__railway__list-services` + `curl -I` on staging URL | Service status "Active", HTTP 200 |
| Server log lines visible | `railway logs` (briefly) | Stage 3's startup lines printed |
| `x-railway-edge` header present | Curl with `-I` flag | Header in response |
| Foundational env vars set | `railway variable list` | `PORT`, `SITE_URL`, `APP_URL`, `RAILWAY_CHECKOUT_DEPTH` all present |
| (If domain migration) DNS resolves | `dig +short <user-domain>` | Cloudflare's IPs (if proxied) or Railway target (if DNS-only) |
| (If domain migration) Tombstone SW responds | `curl -sI https://<user-domain>/sw.js` | 200 with short-cache headers |
| CLI authenticated for ongoing use | `railway whoami` | Returns user's email + workspace |

I summarize for the user:

```
✅ Stage 4 complete:
   • Project "mayasconsulting-marketing" created on Railway
   • Service deployed from GitHub: mayasconsulting/mayasconsulting-marketing
   • Live at: https://mayasconsulting-marketing-production.up.railway.app
   • Custom domain: not yet attached (you can launch fresh whenever you're ready)
   • Foundational env vars set: PORT, SITE_URL, APP_URL, RAILWAY_CHECKOUT_DEPTH
   • Railway CLI authenticated — I can manage env vars autonomously across
     the remaining stages without prompting you

Ready for Stage 5 (email provider setup)?
```

## Common gotchas I handle automatically

- **First build fails because env vars unset**: expected if I let it run before Step 3. I set the vars then redeploy. I never skip the redeploy — Railway doesn't auto-rebuild on env-var change in some plans.
- **Build succeeds but site shows 502**: Express isn't listening on `0.0.0.0:$PORT`. I re-read `server.js` from Stage 3 and confirm `app.listen(PORT, "0.0.0.0", ...)` has the literal `"0.0.0.0"` string.
- **`railway logs` shows "ENOENT package.json"**: Railway is looking in the wrong directory. I check the service's "Root Directory" setting if the user has a monorepo.
- **DNS migration breaks email**: forgot to update DKIM/SPF for the new sending domain. Stage 5 covers domain verification — I make sure the new domain's DKIM/SPF are in place before any cutover.
- **OAuth callback breaks at cutover**: the user forgot to update Google Cloud Console / Whop / etc. with the new URL. The list I provide is non-exhaustive — I grep the user's codebase for any URL-bearing env var that points at the old domain.
- **MCP tool count mismatch**: as of 2026-05, the MCP doesn't expose env-var or custom-domain tools. I don't waste time retrying via MCP if the tool isn't there — fall through to CLI immediately.
- **Shallow clones + variant labels**: if `RAILWAY_CHECKOUT_DEPTH=0` isn't set, Stage 6's `git rev-list --count HEAD` returns `1` silently and variant labels break. I always set this in Step 3.

## Self-research instruction

Before running this stage on a future project, I web-search:
- `Railway MCP server <year>` — has the tool list expanded since the last verified date?
- `Railway CLI <year>` — any new auth or env-var subcommands?

If new capabilities exist, I USE them and append a dated entry to this file's Changelog so future sessions inherit my findings.

## Outputs

After Stage 4:
- Site live on Railway at staging or production domain
- Railway CLI authenticated for autonomous ongoing use across Stages 5–12
- MCP authenticated for project/service discovery
- Foundational env vars set
- (If applicable) Custom domain attached + Cloudflare configured + tombstone SW verified
- Status report shown to user

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Verified Railway Remote MCP (`mcp.railway.com`) has OAuth login + 6 tools. Env-var/domain ops still require CLI fallback. CLI also uses browser OAuth. |
| 2026-05-08 | Added Step 3's "Can Claude set env vars autonomously?" subsection: explicit yes/no for MCP vs CLI vs GraphQL, secrets-file pattern, bulk-set, snapshots/rollback, per-environment patterns, build-bake reminder. Cross-referenced from every stage that sets env vars. |
| 2026-05-08 | DNS cutover protocol expanded with Cloudflare-side configuration block: SSL/TLS Full (strict), Always Use HTTPS toggle path (SSL/TLS → Edge Certificates → scroll down → toggle ON), Automatic HTTPS Rewrites toggle, Minimum TLS Version. Cross-references the SHORT URL domain proxy mode from Stage 10 to head off the most common mix-up. |
| 2026-05-08 | Reframed for non-coder audience: collapsed Step 0's three-path question into a default ("MCP+CLI hybrid" used silently for non_coder; alternatives offered to expert mode only). Resolved the Step 2 self-contradiction (dashboard "Deploy from GitHub" path picked as primary; CLI alternative noted for experts). All bash commands reframed as autonomous Claude actions. Replaced author-specific home-directory secret paths with project-local `<project>/.secrets/` to match the new placeholder convention. Stripped author-attribution references. Fixed duplicate `4.` numbering in cutover protocol. Replaced literal `yourdomain.com` placeholders with `<user-domain>` substitution markers. |
