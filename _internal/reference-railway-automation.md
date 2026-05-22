# Reference: Railway Automation Surface (deep dive)

**Last verified: 2026-05-21** against `docs.railway.com/ai/mcp-server`, `docs.railway.com/cli`, and `docs.railway.com/reference/cli-api`. Re-verify if older than 60 days.

This is the deeper-dive companion to Stage 4 — useful when:
- Building automation scripts that drive Railway from outside Claude Code
- Choosing between MCP / CLI / GraphQL for a specific task
- Hitting edge cases the main stage docs don't cover

## ⚠️ v2 gotcha: `latestDeployment.id` does NOT change between deploys

**The wrong assumption**: when you push a new commit and Railway rebuilds, you might expect `latestDeployment.id` (and `latestDeployment.createdAt`) to change. They don't. Railway re-uses the deployment RECORD in place across rebuilds — same ID, same timestamp, just different `status` and different `meta.commitHash`.

**The right polling pattern**: use `meta.commitHash` matched against the pushed commit to detect "deploy has happened for the commit I just pushed."

```bash
# After git push origin main:
PUSHED_COMMIT=$(git rev-parse HEAD)
echo "Polling for deploy of commit $PUSHED_COMMIT..."

for i in {1..20}; do
  RESPONSE=$(railway status --json)
  CURRENT_COMMIT=$(echo "$RESPONSE" | jq -r '.latestDeployment.meta.commitHash // empty')
  CURRENT_STATUS=$(echo "$RESPONSE" | jq -r '.latestDeployment.status')

  if [ "$CURRENT_COMMIT" = "$PUSHED_COMMIT" ]; then
    echo "Deploy of $PUSHED_COMMIT is in status: $CURRENT_STATUS"
    case "$CURRENT_STATUS" in
      SUCCESS)         echo "Deploy succeeded."; exit 0 ;;
      FAILED|CRASHED)  echo "Deploy failed."; railway logs --tail 100; exit 1 ;;
      BUILDING|DEPLOYING|QUEUED) ;;  # keep polling
      *) echo "Unknown status: $CURRENT_STATUS"; ;;
    esac
  fi
  sleep 30
done

echo "Timed out waiting for deploy of $PUSHED_COMMIT."
exit 1
```

**Why this matters for skills/scripts**: any automation that detects "did my deploy go through" based on `id`/`createdAt` will silently think the deploy never happened, even when it did. Threshold-based "fail after N minutes" guards never trigger because the polling never sees a change. The script either hangs forever (no timeout) or fires a false-failure alert (timeout reached even though deploy completed).

**Why the field doesn't change**: Railway's deployment model treats `Deployment` as a long-lived resource attached to a service. The same Deployment goes through multiple builds across the service's lifetime — each new commit creates a new BUILD (which has its own `meta.commitHash`) but the Deployment object persists. The CLI and GraphQL API surface this directly; the dashboard UI flattens it into "Deployment #N at commit X" which gives the impression of distinct records.

**Where to find the commit hash in different surfaces**:
- CLI: `railway status --json` → `.latestDeployment.meta.commitHash`
- GraphQL: `query { service(id) { deployments(first: 1) { edges { node { meta { commitHash } } } } } }`
- MCP `get_deployment`: tool response includes `commitHash` field at the top level
- Dashboard: only the prefix is shown (first 7 chars); use the CLI/GraphQL for the full hash

This pattern is referenced from [Stage 4](../stage-4-railway.md) (deploy verification step) and [Stage 12](../stage-12-launch-checks.md) (pre-launch deploy validation).

## What's automatable (verified 2026-05)

### Layer 1: Railway Remote MCP — `https://mcp.railway.com`

**Source**: `docs.railway.com/ai/mcp-server`

**Auth**: OAuth browser approve, one-time. Token cached by Claude Code.

**Tools** (verified present 2026-05):

| Tool | What it does |
|---|---|
| `check-railway-status` | Verify CLI install + auth context |
| `list-projects` | Enumerate projects on the account |
| `create-project-and-link` | Create a new project AND auto-link the current directory |
| `list-services` | Enumerate services in a project; returns service IDs |
| `link-service` | Link a service to the current dir so subsequent CLI ops target it |
| `deploy` | Deploy the linked service (fire-and-forget) |
| `deploy-template` | Deploy a service from Railway's Template Library (use this for managed PostgreSQL, Redis, etc.) |
| `create-environment` | Create a new environment within a project |
| `link-environment` | Link an environment to current dir |
| `list-variables` | List env vars on the linked service |
| `set-variables` | Set env vars on the linked service |
| `generate-domain` | Generate a `*.up.railway.app` subdomain for the linked service |
| `get-logs` | Retrieve service logs (snapshot, not stream) |

**Things the MCP does NOT cover** (verified 2026-05; use CLI or dashboard):
- Service create from a GitHub repo with full webhook + auto-deploy wiring (only blank service create is exposed)
- Custom-domain attachment (`generate-domain` is for `*.up.railway.app`, not user-owned domains)
- Streaming live build logs (`get-logs` is a snapshot)
- Deployment cancellation
- Project/service deletion

### Layer 2: Railway CLI — `npm install -g @railway/cli`

**Source**: `docs.railway.com/cli` and `docs.railway.com/reference/cli-api`

**Auth**: `railway login` opens a browser tab for OAuth. Same approve-once flow as the MCP, but cached separately at `~/.railway/`.

**Capabilities** (verbs verified verbatim 2026-05; the singular-vs-plural distinction below is REAL — `variable` is singular per the docs):

| Command | What it does |
|---|---|
| `railway login` / `railway login --browserless` | Browser OAuth (or token-paste mode for headless) |
| `railway logout` | Clear cached token |
| `railway whoami` | Show authed user/workspace |
| `railway init` | Create a new project + link locally |
| `railway link` | Link local repo to existing project |
| `railway service` | Manage services (create, delete, list) |
| `railway add --repo <user>/<repo>` | Add a service from a GitHub repo |
| `railway up` | Deploy local code, streaming the build output |
| `railway run <cmd>` | Run a command with the linked service's env vars injected |
| `railway variable list` | List env vars (add `--json` for structured output) |
| `railway variable set KEY=value` | Set an env var (note: SINGULAR `variable`) |
| `railway variable get KEY` | Read a single env var's value |
| `railway variable delete KEY` | Delete an env var |
| `railway domain example.com` | Attach a custom domain to the linked service |
| `railway logs` | Stream deployment logs |
| `railway logs --build` | Stream build logs specifically |
| `railway deployment list` | Show recent deploys + status |
| `railway status` | Show service health |
| `railway open` | Open the service in browser |
| `railway down` | Tear down (rare; usually you delete via dashboard) |

### Layer 3: Railway GraphQL API — `https://backboard.railway.app/graphql/v2`

**Source**: Railway's public API docs (search `railway graphql`).

**Auth**: Bearer token from `RAILWAY_TOKEN` (Project Token) or `RAILWAY_API_TOKEN` (Account Token), generated in Railway dashboard.

**When to use**: programmatic automation outside Claude Code (CI scripts, cron jobs, custom dashboards, monitoring integrations). The schema is rich enough to do almost anything the dashboard does.

**Common operations**:
- Query: `me { id email }`, `projects { edges { node { id name } } }`, `service { id name domains { ... } }`
- Mutation: `variableUpsert(input: {...})` for env vars, `customDomainCreate(input: {...})` for domains, `deploymentTrigger(...)` for deploys

**Reference**: introspect the schema at the GraphQL endpoint with any GraphQL client (e.g., GraphiQL, Insomnia). The full schema is too large to reproduce here; use `__schema` introspection.

## Decision matrix: which layer for which task

| Task | Best layer | Notes |
|---|---|---|
| List projects / services | MCP | `list-projects` / `list-services` are fastest from chat |
| Create new project + link | MCP | `create-project-and-link` is one step |
| Service create from GitHub repo | Dashboard (preferred), CLI `railway add --repo` (fallback) | MCP doesn't expose; dashboard's "Deploy from GitHub repo" handles OAuth + webhook in one click |
| Set env vars (NON-secret) | MCP `set-variables` OR CLI `railway variable set` | Either works; pick for consistency with surrounding ops |
| Set env vars (SECRET) | CLI with shell substitution | `railway variable set RESEND_API_KEY="$(cat .secrets/resend-api-key.txt)"` keeps secret out of Claude's context window. MCP path requires reading file content into context first, which is worse for leak defense |
| Read env vars | CLI `railway variable list --json` OR MCP `list-variables` | CLI's `--json` is easier to grep |
| Attach custom domain | CLI `railway domain example.com` | MCP's `generate-domain` is for `*.up.railway.app` only |
| Trigger redeploy | CLI `railway up` (streams) OR MCP `deploy` (fire-and-forget) | CLI for visibility into build progress; MCP for scripted hands-off redeploys |
| Stream build logs | CLI `railway logs --build` | MCP's `get-logs` is a snapshot, not a stream |
| One-off scripts with prod env injected | CLI `railway run npm run db:migrate` | The only way to get all linked-service env vars into a local shell |
| Provision a managed PostgreSQL | MCP `deploy-template` with `template: "postgres"` (or dashboard) | No dedicated `create-database` tool exists |
| Programmatic automation outside Claude Code | GraphQL | Token-based, no browser-OAuth dependency |
| Bulk operations | GraphQL | MCP/CLI are interactive; GraphQL is scriptable |

## Sample GraphQL queries (for reference / future automation)

### Get project + service IDs

```graphql
query MyProjects {
  me {
    id
    email
    projects {
      edges {
        node {
          id
          name
          services {
            edges {
              node {
                id
                name
              }
            }
          }
        }
      }
    }
  }
}
```

### Set an env var (mutation)

```graphql
mutation SetVar($input: VariableUpsertInput!) {
  variableUpsert(input: $input)
}

# variables:
{
  "input": {
    "projectId": "...",
    "environmentId": "...",
    "serviceId": "...",
    "name": "VITE_POSTHOG_KEY",
    "value": "phc_..."
  }
}
```

### Get deployment logs

```graphql
query DeployLogs($deploymentId: String!) {
  deploymentLogs(deploymentId: $deploymentId) {
    timestamp
    message
  }
}
```

## Common automation patterns

### Pattern 1: Set N non-secret env vars from a script

```bash
#!/bin/bash
# Set all marketing-site non-secret env vars in one go
declare -A vars=(
  ["PORT"]="3000"
  ["VITE_POSTHOG_HOST"]="https://<user-domain>/<your-proxy-slug>"
  ["RAILWAY_CHECKOUT_DEPTH"]="0"
  # etc. — non-secret values only
)
for key in "${!vars[@]}"; do
  railway variable set "$key=${vars[$key]}"
done
```

### Pattern 2: Set secrets from local files

```bash
#!/bin/bash
# Set secrets via shell substitution so values never enter Claude's context
railway variable set RESEND_API_KEY="$(cat .secrets/resend-api-key.txt)"
railway variable set VITE_POSTHOG_KEY="$(cat .secrets/posthog-public-key.txt)"
railway variable set WHOP_WEBHOOK_SECRET="$(cat .secrets/whop-webhook-secret.txt)"
```

### Pattern 3: Snapshot env vars before changes (rollback safety)

```bash
railway variable list --json > /tmp/env-snapshot-$(date +%Y%m%d-%H%M).json
# Make changes...
# Roll back if needed (set vars back from snapshot):
jq -r 'to_entries | .[] | "\(.key)=\(.value)"' /tmp/env-snapshot-...json \
  | xargs -I {} railway variable set "{}"
```

### Pattern 4: Trigger a redeploy from Claude Code (MCP)

```
You: redeploy the marketing service
Claude: [calls mcp__railway__list-services]
        [identifies the marketing service via name match]
        [calls mcp__railway__link-service if not already linked]
        [calls mcp__railway__deploy]
        Redeployed. Tail-watching build:
        [calls Bash: railway logs --build]
```

### Pattern 5: Cron-driven canary checks

Several scheduler options work, in rough order of effort:

- **GitHub Actions cron**: free for public repos, included in private-repo minutes; commits a `.github/workflows/canary.yml` to the repo
- **Railway cron jobs**: native Railway feature; configure via dashboard or CLI on a separate service
- **A scheduled-tasks MCP** (if installed in the user's Claude Code environment): runs the check on the user's local machine on a cron expression

I pick based on what the user already has set up. The check itself is straightforward:

```
# Pseudocode for any scheduler:
schedule:    "0 * * * *"  # hourly
command:     curl -I https://<user-domain>/
on_failure:  alert via the user's preferred channel (Slack webhook, email, etc.)
```

## Edge cases + gotchas

### Shallow clones break `git rev-list --count HEAD`

Railway's default checkout depth is shallow (1 commit). If the build derives variant labels from `git rev-list --count HEAD`, you get `1` for every deploy → all events tagged `variant-a` regardless of actual deploy.

**Fix**: `railway variable set RAILWAY_CHECKOUT_DEPTH=0` and redeploy.

### `RAILWAY_GIT_COMMIT_SHA` reliability

Set automatically by Railway on every deploy. Reliable. The `?: 'devlocal'` fallback in `vite.config.ts` only fires for local builds.

### Custom domain provisioning lag

After attaching a custom domain in Railway, SSL provisioning can take 5–10 minutes. During that window, `https://<domain>/` returns SSL errors. Don't trigger a DNS cutover until SSL is fully provisioned (Railway's UI shows "Active" status when ready).

### `railway run` env var leak

`railway run npm run db:migrate` injects ALL the linked service's env vars into the local shell for the duration of that command. If your migration script logs env vars at startup, secrets show up in your terminal. Use `set +x` discipline.

### MCP `set-variables` vs CLI for secrets

`mcp__railway__set-variables` requires the value to be passed in the tool call payload — which means I have to read the secret value into Claude's context first. The CLI's `railway variable set KEY="$(cat .secrets/foo.txt)"` shells out and the substitution happens before `railway` sees the args, so the secret never enters my chat history. **For any value the user treats as sensitive, the CLI is the right choice even though MCP would technically work.**

### MCP rate limits (unverified as of 2026-05)

Railway's docs don't currently document an MCP-specific rate limit. The underlying GraphQL API has rate limits (varies by plan tier). Hitting them returns 429 with retry-after.

### CLI version drift

`@railway/cli` updates frequently. If a command behaves differently than this doc describes, run `railway --version` and check the [release notes](https://github.com/railwayapp/cli/releases). Update with `npm install -g @railway/cli@latest`.

The most common drift in our domain: `railway variables` (plural) was an alias in older versions but `railway variable` (singular) is the canonical form per current docs. If the singular form errors with "unknown command" on an older CLI, update before working around it.

## Self-research instruction

Re-verify the MCP tool list every 60 days. Search:
- `Railway MCP server <year>` — official docs page
- `Railway CLI changelog <year>` — new subcommands or renames
- `Railway API documentation <year>` — capability changes

Append findings to this file's Changelog.

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Documented MCP at `mcp.railway.com` with 6 tools (list-projects, create-project, list-services, redeploy, accept-deploy, whoami), CLI for env-var + domain ops + log streaming, GraphQL at `backboard.railway.app/graphql/v2` for non-interactive automation. |
| 2026-05-08 | **Major correction** after re-verifying against the official Railway docs (`docs.railway.com/ai/mcp-server`, `docs.railway.com/cli`, `docs.railway.com/reference/cli-api`). The MCP tool inventory was wrong: actual toolset is 13 tools, not 6. Specifically: `create-project-and-link` (not `create-project`); `deploy` (not `redeploy`); `set-variables` AND `list-variables` DO exist on the MCP (previously claimed they didn't); `get-logs` exists; `deploy-template` is the path for managed databases; `generate-domain` is for `*.up.railway.app` subdomains only, not custom domains. CLI variable command is `railway variable set` (SINGULAR `variable`), not `railway variables set` — the plural form is an older alias that may error on current CLI versions. New decision matrix distinguishes secret-setting (CLI required for context-leak defense) from non-secret-setting (either surface works). |
