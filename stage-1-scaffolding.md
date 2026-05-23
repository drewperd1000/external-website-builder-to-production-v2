# Stage 1: Project Scaffolding + Baseline Build

**Last verified: 2026-05-21.**

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) run all the scaffolding commands, write all the config files, and verify the baseline build works. The user does NOT run `npm install` / `git init` / `gh repo create` / etc. — I do that via my Bash tool. The user's only actions in this stage are: (1) confirming workspace location, (2) approving GitHub repo creation, (3) clicking through the GitHub OAuth flow if not already authenticated.

> **Upgrading from v1?** See [CHANGELOG.md](CHANGELOG.md). v1 always scaffolded Vite; v2 forks to Astro for content-hub sites (the new default) and to Vite for non-content-hub sites.

## Choose stack: Astro or Vite (based on `aio.content_hub`)

Stage 1 forks based on the onboarding answer to "should this site be discoverable by AI search engines?" (saved as `aio.content_hub: true | false`):

| `aio.content_hub` | Stack to scaffold | Why |
|---|---|---|
| `true` (default for marketing sites) | **Astro 5** + React islands + Express wrapper | Native MDX, content collections, sitemap+RSS, schema-component infrastructure (Stage 13 builds on this) |
| `false` | **Vite 6** + React + Express | Faster hot reload, smaller dep tree, no schema infrastructure overhead. Same as v1's path. |

**Default behavior**: if `aio.content_hub` is unset in skill config, I default to `true` (Astro). The cost of Astro on a non-content-hub site is small; the cost of starting on Vite and later realizing you needed schema infrastructure is large.

The forked sections below are labeled **[Astro path]** and **[Vite path]**. Steps with no label apply to both paths. If you're reading this skill for guidance and your project is on Vite, follow Vite-path steps only; if Astro, follow Astro-path steps only.

## Astro-path additional scaffolding

When `aio.content_hub = true`, in addition to the baseline scaffold, I also:

1. Install `astro@^5`, `@astrojs/node@^9`, `@astrojs/mdx@^4`, `@astrojs/react@^4`, `@astrojs/sitemap@^3`, `@astrojs/rss@^4`, plus `vitest@^2` as dev dep.
2. Write `astro.config.mjs` with `output: "server"` and `adapter: node({ mode: "middleware" })` — middleware mode is what lets Express wrap the handler in Stage 3.
3. **Create `src/site-config.ts`** — the single source of truth for brand identity that every other file imports from. See the next subsection.
4. Relocate any imported React entry points (`main.tsx`, `App.tsx`) to `src/_legacy/` so the Astro page routes can take over `/`, `/about`, etc. (See the migration matrix below for the per-source-stack details.)
5. Set up the `@` path alias pointing at `./src` so schema components and layouts can be imported cleanly.

Stage 13 fills in the schema components, content collections, layouts, blog routes, sitemap filter, and RSS — all importing brand identity from `site-config.ts`. Stage 1 just gets Astro up and serving "hello world" so Stage 3 can wrap it.

### `<project>/CLAUDE.md` — operating principles for every future Claude session on this project

Before any code scaffolding, I write a starter `CLAUDE.md` at the project root containing 10 universal operating principles. Claude Code auto-loads `CLAUDE.md` from the project directory on every session start, so these principles persist across every future Claude session working on this project — without me having to re-explain them and without the user having to remember.

The 10 principles in this template are the same ones documented in [`_internal/reference-operating-principles.md`](_internal/reference-operating-principles.md), written for brevity here so they fit cleanly in the project's rules file:

```markdown
# <Project> — Claude Operating Rules

Rules every Claude session working on this project follows. Loaded
automatically by Claude Code from this file on session start.

## 1. Claude executes; user directs

Default operating mode: Claude does ALL the keystrokes — edits files,
runs CLI commands, pushes commits, verifies results, reports concrete
evidence. The user provides decisions (what to build, what voice, what
tradeoffs) and a small number of inputs where the tool layer can't act
on their behalf.

**Anti-pattern this rejects**: handing the user a list of steps to run
themselves. "First, run `npm install`, then edit `foo.json`, then
redeploy" is wrong when Claude has the tools to do all three. If Claude
has the tools, Claude uses them.

**Rare exceptions** (user MUST act because the tool layer can't):
- Dashboard-only ops the MCP doesn't expose (vendor-specific UI without
  an API: Cloudflare Email Routing, DocuSign, etc.)
- Auth flows requiring physical interaction (OAuth browser approves, MFA
  codes, password manager retrieval)
- Judgment calls (brand voice, copy phrasing, design taste, pricing) —
  Claude gathers + presents options; user picks

**When you DO ask the user to act**, use plain-language commands:
- `Open Terminal and type:` `<the literal command>`, or
- `Go to <site> → <menu> → <button>`, then `<do this>`

Show the literal copy-pasteable command with full paths. Tell them
what they'll see. Banned phrasings: "OAuth flow," "interactive
handshake," "session bootstrapping," "auth dance," "establish
authentication."

## 2. Verify deploys by commit-hash, not bundle-hash or deployment-id

Railway re-uses the deployment record in place across rebuilds.
`latestDeployment.id` does NOT change between deploys. Only
`latestDeployment.meta.commitHash` reliably changes. Pattern:
capture `git rev-parse HEAD` right after pushing, then poll
`railway status --json | jq -r '.latestDeployment.meta.commitHash'`
and compare against expected. Never claim "deployed" based on `id`
or `createdAt`.

## 3. Cross-surface naming lock-step

When a name is shared across surfaces (Railway env vars / PostHog
event names / DB columns / GSC property / Reoon webhook URLs / Whop
product titles / CSS class allowlists in external dashboards / etc.),
renaming in one surface without updating all consumers in the same
session is silent breakage. Before any rename: grep across the
project AND inventory every external surface that references the
name. Apply lock-step in one session.

## 4. Never hardcode production domains

Source from `SITE_URL` / `APP_URL` env vars (or from
`src/site-config.ts` if this project uses the Astro path). One
env-var change moves the site; hardcoded references would require
code edits at every reference.

## 5. Push after every meaningful commit

Don't accumulate local commits that aren't on GitHub. Push protects
against laptop-death between commit and push; also keeps Railway
auto-deploy + scheduled tasks + cross-device sessions in sync with
the remote.

## 6. Surface real problems early

When you hit a genuine blocker (contradiction in stated requirements,
factual error in copy that would create legal risk, vendor behavior
diverging from documented behavior, dependency not behaving as
documented), tell the user immediately and offer options. Don't
proceed with a hidden workaround that creates technical debt.

Pre-existing test failures unrelated to the current task are NOT
real problems — proceed silently.

## 7. Defer to provider taxonomies before inventing your own

When integrating with external providers (ad platforms, payment
processors, CRMs, analytics tools, email senders), use the
provider's existing taxonomies, macros, event names, and field
naming verbatim. Don't translate provider values to a custom
convention — the friction compounds with every campaign / event /
record. Only invent your own convention when the provider's
system is genuinely unavailable, opaque, or wrong for the use case.

## 8. Wire Slack notifications for ops-relevant events

When shipping a feature that produces ops-relevant signals (signups,
errors, deploys, threshold crossings, async jobs completing or
failing), evaluate whether it warrants a Slack notification and
wire it. Don't scaffold a notification function without wiring it
(false sense of coverage). Don't Slack high-volume routine events.

## 9. Don't claim "locked in" or "deferred" without the canonical doc current

If you're about to say "X is locked in" / "we can defer Y" / "future
Claude can read <doc> for this" — the canonical project doc must
already reflect the current state. Read it cold; if it'd surprise a
fresh contractor, update it BEFORE making the claim. Don't fragment
findings across many commits without consolidating into the
canonical doc.

## 10. Backup discipline

Production-bound secrets (`.env`, `.secrets/`, API keys, OAuth tokens)
live in encrypted off-machine backup. Daily scheduled tar + encrypt +
upload to Backblaze B2 / S3 / R2. Encryption passphrase in password
manager, NOT in the backup. Test the restore once before relying on it.

## Project-specific extensions

(Add your own rules below as conventions emerge. The skill leaves
this section empty for you to fill in. Examples of what fits here:
"always use feature flag X before deploying behavior Y," "every
commit message starts with `feat:` / `fix:` / `chore:`," "API keys
for vendor Z live in `.secrets/z-api-key.txt`.")
```

**Why I scaffold this in Stage 1, not later**: the principles are load-bearing for the rest of the skill execution. If Stage 4 deploys to Railway and a future Claude session re-deploys without these rules loaded, they'll likely poll on `latestDeployment.id` and silently report success while a different commit is still queuing. Establishing the rules early means every subsequent session inherits them without needing the skill loaded.

**The user can edit anything in this file** — these are STARTER rules, not law. As project-specific conventions emerge (variant-naming schemes, feature-flag procedures, custom command vocabulary, etc.), they get appended under "Project-specific extensions."

I do NOT overwrite a pre-existing `<project>/CLAUDE.md` if one is present. If the user has rules from a previous project they want preserved, I'll merge intelligently (append the 10 starter rules under a "Universal operating rules" section above whatever's already there, ask the user to confirm).

### `src/site-config.ts` — single source of truth for brand identity

Without this file, brand constants get duplicated across `astro.config.mjs`, `BaseLayout.astro`, `BlogPostLayout.astro`, `schema-builders.ts`, `server.js`, the RSS feed, and any cornerstone that hardcodes the brand URL. A rename means find-and-replace across 8+ files with the risk of missing one. Centralizing them means one file edits + one set of env vars, then every consumer picks up the change.

```typescript
// src/site-config.ts
//
// Brand identity for this site. Imported by every layout, schema builder,
// server route, RSS feed, and content collection that needs to reference
// brand values.
//
// Constants are sourced from env vars where the value should differ between
// environments (SITE_URL, APP_URL — staging vs prod), and from inline
// defaults where the value is a fixed brand fact (BRAND name, AUTHOR_NAME).
//
// To change a brand value:
//   - Identity-fixed values (BRAND, AUTHOR_NAME, parent org): edit this file
//   - Environment-specific values (SITE_URL, APP_URL): set on Railway via
//     `railway variable set SITE_URL=...` (Stage 4 documents the pattern)

export const SITE_URL = process.env.SITE_URL || "https://mayasconsulting.com";
export const APP_URL = process.env.APP_URL || "";
export const BRAND = "Maya's Consulting";
export const BRAND_DESCRIPTION = "<your-brand-description>";  // for OG / RSS / meta-description fallbacks
export const LOGO_PATH = "/logo.svg";

// Author identity (used in PersonSchema, ArticleSchema.author, AuthorBio)
export const AUTHOR_NAME = "<your-author-name>";
export const AUTHOR_SUFFIX = "";              // e.g., "Ph.D.", "L.Ac.", "MBA"
export const AUTHOR_JOB_TITLES: string[] = [];
export const AUTHOR_ALUMNI_OF: string[] = [];
export const AUTHOR_AFFILIATIONS: string[] = [];
export const AUTHOR_KNOWS_ABOUT: string[] = [];

// Parent organization (optional — used in Organization.parentOrganization)
export const PARENT_ORG = "";                  // e.g., "<your-parent-co>"
export const PARENT_ORG_URL = "";              // e.g., "https://<your-parent-co>.com"

// sameAs URLs for entity disambiguation (Organization.sameAs, Person.sameAs)
// Each one strengthens entity recognition in AI-search retrieval. Reciprocal
// links (the linked entity also lists this site) reinforce the signal.
export const SAME_AS: string[] = [
  // APP_URL,
  // PARENT_ORG_URL,
  // "https://linkedin.com/in/<handle>",
  // "https://twitter.com/<handle>",
];

// Email identity (used by server.js for transactional sends)
export const EMAIL_FROM = process.env.EMAIL_FROM || `${BRAND} <hello@mayasconsulting.com>`;
export const EMAIL_NOTIFY_TO = process.env.EMAIL_NOTIFY_TO || "hello@mayasconsulting.com";
```

I substitute the user's actual brand values from `.skill-config.json` into this file at write time. Onboarding's named-author identity questions feed `AUTHOR_*` constants; `project.brand` + `project.site_url` feed `BRAND` and `SITE_URL`; `aio.parent_org` feeds `PARENT_ORG`.

For non-Astro (Vite-only) projects, the same file lives at `src/site-config.ts` and Vite's `import.meta.env.VITE_*` is substituted for `process.env.*`. The pattern is identical; only the env-var reference syntax differs.

**Why both env vars AND inline constants**: SITE_URL needs to differ between staging and production (env var). BRAND name and AUTHOR_NAME do not (inline constant — these are identity facts, not configuration). Mixing the two correctly lets the same file serve dev, staging, and prod without per-environment edits.

**The `astro.config.mjs` Stage 1 writes is minimal** — sitemap integration is intentionally NOT added here. It's added in Stage 13 once the draft-aware filter logic is in place. Stage 1's config is just enough for `npm run dev` to serve a page.

```javascript
// astro.config.mjs — Stage 1 minimum viable shape
import { defineConfig } from "astro/config";
import node from "@astrojs/node";
import mdx from "@astrojs/mdx";
import react from "@astrojs/react";

const SITE_URL = process.env.SITE_URL || "https://mayasconsulting.com";

export default defineConfig({
  site: SITE_URL,
  output: "server",
  adapter: node({ mode: "middleware" }),
  integrations: [mdx(), react()],
  vite: {
    resolve: { alias: { "@": new URL("./src", import.meta.url).pathname } },
  },
});
```

**Astro-specific dev/start scripts** (replace the equivalent Vite scripts in `package.json`):

```json
{
  "scripts": {
    "dev":        "astro dev",
    "build":      "astro build",
    "start":      "node server.js",
    "preview":    "node server.js",
    "type-check": "astro check",
    "test":       "vitest run",
    "test:watch": "vitest"
  }
}
```

If the user already has a `predev` / `prebuild` script that vendors PostHog static files (Stage 6 sets this up), I preserve it.

## Source-stack → Astro target migration matrix

Different external builders produce different source structures. The skill converges them all to the same target (Astro 5 + React islands + Express wrapper for the AIO path). This section documents the migration pattern per source-stack shape.

I read `origin.source_stack` from `.skill-config.json` (Stage 0 detects this — values: `"react-router-spa"`, `"react-no-router"`, `"static-html-css"`, `"next-pages"`, `"next-app-router"`, `"plain-html-multi-page"`). If unset, I run a quick re-detection now via `ls src/` + `grep -l 'react-router-dom' package.json`.

### Source: React Router SPA (most common for AI generators — Bolt, Lovable, v0, Readdy, some Cursor-built sites)

**Symptom**: `package.json` includes `react-router-dom`. Source has `App.tsx` with `<BrowserRouter>` and `<Routes>/<Route>`. Individual pages live under `src/pages/` or `src/routes/` as React components.

**Migration plan** — wrap-and-defer (recommended for sites <2 weeks old or >10 pages):

1. **Relocate** legacy React entry points to `src/_legacy/`:
   ```bash
   mkdir -p src/_legacy
   git mv src/main.tsx src/App.tsx src/_legacy/
   git mv src/pages src/_legacy/pages   # or src/routes, depending on convention
   git mv src/components src/_legacy/components   # if components are tightly coupled to routes
   ```
   (If components are routing-agnostic — most utility components, schema-irrelevant UI — leave them at `src/components/` so Astro pages can import them directly.)

2. **Create per-page island wrapper components** at `src/components/islands/`. Each wrapper provides BrowserRouter context so the legacy page's transitive `<Link>` and `useLocation()` calls don't throw:

   ```tsx
   // src/components/islands/HomeIsland.tsx
   import { BrowserRouter } from "react-router-dom";
   import HomePage from "@/_legacy/pages/home/page";

   /**
    * Astro island wrapper for legacy React page.
    *
    * Why: legacy components use react-router-dom's <Link> and useLocation(),
    * which require a <BrowserRouter> ancestor. When wrapped as client:only="react",
    * React renders standalone without the BrowserRouter context that App.tsx
    * used to provide. Without this wrapper, hydration throws.
    *
    * Routing is handled by Astro (each page is its own Astro route);
    * BrowserRouter here is effectively a context provider shim.
    *
    * Follow-up: replace <Link> with <a> in legacy components, then this
    * wrapper can be removed and the page can be hydrated directly.
    */
   export default function HomeIsland() {
     return (
       <BrowserRouter>
         <HomePage />
       </BrowserRouter>
     );
   }
   ```

3. **Create one Astro page per route** at `src/pages/`. Each page hydrates its island as `client:only="react"`:

   ```astro
   ---
   // src/pages/index.astro
   import BaseLayout from "@/layouts/BaseLayout.astro";
   import HomeIsland from "@/components/islands/HomeIsland";
   import { BRAND } from "@/site-config";
   ---
   <BaseLayout
     title={`${BRAND} — Home`}
     description="<your-home-meta-description>"
     canonicalPath="/"
     includeSchema={{ organization: true, website: true }}>
     <HomeIsland client:only="react" />
   </BaseLayout>
   ```

   `client:only="react"` (not `client:load` or `client:visible`) is required because the legacy React page uses browser-only APIs (`window`, `useLocation`) that fail under SSR.

4. **The `<Link>` → `<a>` follow-up** (optional, do later). React Router's `<Link>` intercepts clicks for SPA-style navigation. Inside Astro, that breaks Astro's standard navigation (full page loads with shared scripts cached). Replacing `<Link to="/about">` with `<a href="/about">` restores Astro's expected behavior. After this refactor across all legacy components, the BrowserRouter wrapper can be removed and pages can switch to `client:load` for faster hydration.

   This is a separate effort. For Stage 1, the wrap-and-defer pattern gets a working site faster.

**Migration plan** — refactor-at-port-time (for sites >2 weeks old, <10 pages, or sites where the legacy code is messy enough that "preserve as-is" isn't worth it):

1. Skip the island wrapper. Convert each React Router page directly into an Astro page (`src/pages/<route>.astro`) that hydrates pure Astro components and React utility components (no router).
2. Remove `react-router-dom` from `package.json`.
3. Replace all `<Link>` with `<a>`.

This is more work upfront but eliminates the BrowserRouter shim and the dual-routing-system mental model.

### Source: Plain React (no router)

**Symptom**: `package.json` does NOT include `react-router-dom`. Source has `App.tsx` or `main.tsx` that renders one big component (single-page site, no internal navigation).

**Migration plan**:
1. Relocate to `src/_legacy/` as above.
2. Create ONE Astro page (`src/pages/index.astro`) that hydrates the legacy App as a `client:only="react"` island.
3. No BrowserRouter wrapper needed — the legacy code doesn't depend on one.
4. If you want to add additional pages later (about, contact, blog), create Astro pages with their own islands or pure Astro content.

### Source: Static HTML/CSS (Webflow export, Framer export, some Readdy outputs)

**Symptom**: source is a `.zip` of `.html`, `.css`, `.js` files. No build step. No React. Pages live as individual `.html` files with shared CSS.

**Migration plan**:
1. Convert each `.html` file to a corresponding `.astro` page under `src/pages/`. Astro accepts HTML directly inside `.astro` files — you can copy-paste the `<body>` content into an Astro page wrapped in `<BaseLayout>`.
2. Move CSS files to `src/styles/`. Import the global stylesheet from `BaseLayout.astro`.
3. Move JS files (if any non-trivial JS exists) to `src/scripts/` and either:
   - Inline small scripts directly in the Astro page (`<script>` tag)
   - Wrap larger scripts as Astro components with `<script>` tags
   - Wrap React-style interactive components by rewriting them as Astro components or as small React islands
4. Skip the React-island machinery entirely. Pages are static or have minimal interactivity.

This is the cleanest target — no React in the mix at all, faster builds, smaller bundles.

### Source: Next.js (Pages Router or App Router)

**Symptom**: source has `next.config.js`, `pages/` or `app/` directories, server components, getStaticProps, etc.

**Migration plan** — this is the heaviest migration; consider whether it's worth it:

- If the source site uses Next-specific features heavily (server components, server actions, Image optimization, middleware, ISR), the Next → Astro migration is a substantial rewrite. Stage 1 fork to **stay on Next.js** is often the right call. (v2's content-hub patterns DO work in Next.js with adapter changes — Next has its own MDX support and content-collection equivalents.)
- If the source is "Next.js shaped like a React Router SPA" (most AI-generator Next.js outputs — they use `pages/` routing but don't depend on server-side features), the React Router migration steps above apply with minor adaptation.

For most users, I default to: **detect Next.js → ask the user whether they want to migrate to Astro or stay on Next.js + adapt the AIO infrastructure for Next.** Most pick "stay on Next.js" for simplicity; the schema components and content collections work in Next with small changes (Next has `Head` instead of Astro's `head-extra` slot; same JSON-LD output, different injection mechanism).

If the user picks "stay on Next.js," I write a Next-adapted version of Stage 13's content hub in their project. This is out-of-scope for the canonical v2 stages — flag as bespoke per-project work.

### Source: Plain multi-page HTML (no build step)

**Symptom**: no `package.json`, no `node_modules`, just a folder of `.html` files. Often from Squarespace exports, hand-coded sites, or very old Wix exports.

**Migration plan**: same as the "Static HTML/CSS" case above — convert each `.html` to `.astro`, move CSS to `src/styles/`, no React in the mix.

### Decision tree (TL;DR)

| Source contains | Use this migration plan |
|---|---|
| `react-router-dom` in `package.json` | React Router SPA → island wrappers OR refactor-at-port-time |
| React but no router | Plain React → single Astro page hydrating one big island |
| Static `.html` files, no React | Static HTML → Astro pages, no islands |
| Next.js (heavy use of server features) | Default to staying on Next.js + adapting AIO patterns |
| Next.js (just SPA-shaped) | Treat as React Router SPA above |
| Plain multi-page HTML | Same as Static HTML |

The decision determines what Stage 1 actually scaffolds and how Stage 2's platform-dep cleanup interacts with the result. I confirm the chosen migration plan with the user before executing.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 1 needs:
- `project.slug` — used as the GitHub repo name and Railway project name. Auto-derived from `project.brand` if missing; user confirms.
- `project.brand` — used as default `<title>` and OG metadata. If missing, I prompt; this is blocking because every later stage references it.
- `project.site_url` — substituted into `.env.example`. If missing, I use the Railway staging URL placeholder and flag for Stage 4 to update.

I detect `styling.system` (`tailwind` / `plain` / `css-in-js`) at this stage and save it for Stage 8's ConsentBanner implementation choice.

## What I'm doing in this stage

Getting the source code into a fresh repo with a clean baseline build that runs locally. After this stage, the project's build pipeline is verified working — `npm install` + `npm run build` + `npm run dev` all succeed and a basic page renders at `http://localhost:5173` (or the framework's default port).

**Time**: 5–10 minutes of automated work.

## Prerequisites

- Stage 0 complete with `migration-punchlist.md` confirmed by the user.
- The source artifact is in hand (zip, repo, or files copied from production), or the user is starting fresh.
- The user has GitHub CLI authenticated (`gh auth status` returned green during onboarding's account check). If not, I install it and run `gh auth login` now — they click "Authorize" once in their browser.

## My execution sequence

### Step 1: I confirm workspace location with the user

```
Where should I put the new project on your machine?

  (a) Default: <user's home>/projects/<project-slug>
  (b) Inside an existing parent directory you specify
  (c) I'll just use the current directory

Which would you like?
```

If they pick (a), I create `~/projects/<slug>/` (or the platform equivalent). If (b), I use their path. If (c), I work in cwd.

### Step 2: I extract / clone / scaffold the source

**Invariant: I never introduce a subdirectory between the project folder and the project files.** Doing so puts `.git` one level too deep (`<project>/wrapper/.git` instead of `<project>/.git`), which breaks any tool that auto-walks upward to find the nearest repo (`git rev-parse --show-toplevel`, IDE Git plugins, `gh` CLI, etc.). This was a real multi-script recovery on a prior Readdy import — don't recreate it. Project files land DIRECTLY at the project root.

Branching on origin platform (which I know from onboarding):

- **Zip from a website builder**: I extract directly into the project folder (no wrapper). Either shape works — I pick whichever reads cleanly:
  ```bash
  # Option A — make the project folder first, then unzip into it
  mkdir <project-slug>
  cd <project-slug>
  unzip "<path-to-zip>" -d .

  # Option B — unzip into the named folder, then cd
  unzip "<path-to-zip>" -d <project-slug>
  cd <project-slug>
  ```
  Both leave `.git` at `<project-slug>/.git` after Step 6's `git init`. **NEVER** `unzip ... -d extracted; cd extracted` — that creates the wrapper this invariant forbids.
- **Existing GitHub repo**: I run `git clone <url> <project-slug>` (NOT `git clone <url>` into a wrapper-then-rename pattern), and `cd <project-slug>`. Same no-wrapper rule.
- **Hot-pulled HTML/CSS/JS from production**: I scaffold a fresh Vite + React project at the project root with `npm create vite@latest <project-slug> -- --template react-ts`, then copy the user's files in.
- **Starting fresh**: I scaffold a fresh Vite + React + TypeScript + Tailwind project at the project root via `npm create vite@latest`.

### Step 3: I install dependencies and verify the baseline build

I run `npm install` and `npm run build`. If both succeed, I move on. If either fails, I diagnose:

- **`npm install` fails**: I check Node version (most modern stacks need ≥ 20). I look for platform-specific deps that don't install on the user's OS. If a different package manager's lockfile is present (yarn.lock, pnpm-lock.yaml), I report this to the user and ask which to standardize on.
- **`npm run build` fails**: TypeScript errors from the platform's auto-generated types — I either fix or annotate with `// @ts-expect-error` plus a comment. Missing files referenced by the platform's runtime — I cross-reference with the `migration-punchlist.md` from Stage 0; these get addressed in Stage 2. Vite plugin errors — I look for commented-out platform-specific plugins (Stage 0 already flagged these); the platform may have left a dev-only plugin import that crashes in production.

If I can't recover within 5 minutes, I stop and ask the user how they'd like to proceed (try a different Node version, accept some warnings, etc.).

### Step 3.5: I configure the `@/` path alias

Stages 6, 7, 8, 9, and 11 all import from `@/_core/...` (e.g., `import { readConsent } from "@/_core/consent"`). The alias must be configured in BOTH `tsconfig.json` (so TypeScript resolves it) AND `vite.config.ts` (so Vite's bundler resolves it at runtime). If either is missing, builds break with cryptic "Cannot find module '@/_core/consent'" errors.

**`tsconfig.json` — I add or merge into `compilerOptions`:**

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

If the file uses references (e.g., `tsconfig.app.json`), I add the same `baseUrl` + `paths` to the referenced config — Vite reads the leaf config, not the root.

**`vite.config.ts` — I add or merge the `resolve.alias` block:**

```ts
import path from "node:path";
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

I verify by running `npm run build` once after wiring both files. If it builds clean, the alias is wired correctly for everything later stages will throw at it.

If the user's source platform (Webflow / Framer / Readdy) shipped a `vite.config.ts` that already has a `resolve.alias` block, I merge mine in rather than overwriting. If they shipped a `jsconfig.json` instead of `tsconfig.json`, I add the same `baseUrl`/`paths` there.

### Step 4: I write `.gitignore`

If absent, I create at repo root:

```
node_modules
dist
out
build
.env
.env.local
.env.*.local
.skill-config.json
*.log
.DS_Store
desktop.ini
.vite
.cache
auto-imports.d.ts
*.tsbuildinfo
.idea
.vscode
*.swp
```

If a `.gitignore` exists from the source platform, I audit it and add anything missing. Specifically I make sure `node_modules`, `.env`, `.skill-config.json`, and `desktop.ini` (Windows + cloud-sync corruption risk) are all listed.

### Step 5: I write `.env.example`

Created at repo root with EVERY env var that later stages will set, using safe placeholder values. This is the future-proofing against the silent-Vite-build trap (SKILL.md invariant #1).

```bash
# Build-time vars (Vite bakes into bundle — must be set BEFORE every build)
VITE_POSTHOG_KEY=phc_xxxxxxxxxxxx
VITE_POSTHOG_HOST=https://<your-domain>/<your-proxy-slug>
VITE_CLARITY_PROJECT_ID=<your-clarity-id>

# Runtime vars (server-side; set on Railway service)
RESEND_API_KEY=re_xxxxxxxxxxxx
EMAIL_FROM=<Brand Name> <hello@<your-domain>>
EMAIL_NOTIFY_TO=hello@<your-domain>
SITE_URL=https://<your-domain>
APP_URL=https://app.<your-domain>
PORT=3000

# Optional: timezone for date-formatting in events (Stage 6)
# Any IANA tz like "America/New_York", "Europe/London", "Asia/Tokyo".
# Defaults to UTC if unset — safe everywhere; only override if dashboards
# need wall-clock days aligned to a specific region.
DEPLOY_TIMEZONE=UTC

# Optional: webhook integration (Stage 9)
WHOP_WEBHOOK_SECRET=whsec_xxxxxxxxxxxx
WHOP_API_KEY=xxxxxxxxxxxx
POSTHOG_PROJECT_API_KEY=phc_xxxxxxxxxxxx
```

I substitute `<your-domain>` with the user's actual domain at write time. Placeholders for keys remain `xxxxxxxxxxxx` style — the user pastes real values later (or I fetch them from a vendor's API where possible).

### Step 6: I initialize git (if not already a repo)

I run `git init`, then check if `git config user.name` and `user.email` are set globally. If yes, I use those. If not, I ask the user once and then set them. I make the initial commit:

```
git add .
git commit -m "Initial commit: <platform>-generated baseline"
```

### Step 7: I create the GitHub remote

I run `gh repo create <slug> --private --source=. --remote=origin --push` (or public if the user prefers). If `gh` isn't authenticated, I prompt the user to run `gh auth login` once — they click "Authorize" in their browser. After that, every subsequent `gh` call is autonomous.

Stage 4 (Railway) will connect to this GitHub repo for continuous deploys.

### Step 8: I detect the framework

Most modern marketing sites are Vite + React + TypeScript + Tailwind. If the project is something else, I auto-detect from `package.json` and adapt:

| Framework | Build output | Dev port | Stage 3/4 changes I apply |
|---|---|---|---|
| **Vite + React** | `dist/` (Vite default) | 5173 | Default — Stages 3/4 assume this |
| **Next.js** | `.next/` (server) | 3000 | Stage 3 may not need an Express layer (Next has its own server). PostHog reverse proxy goes in `middleware.ts` or `pages/api/` instead. |
| **Astro** | `dist/` | 4321 | Stage 3 Express works; Astro's adapter setup may add complexity |
| **SvelteKit** | `.svelte-kit/output/` | 5173 | Stage 3 needs the Node adapter; PostHog proxy in `hooks.server.ts` |
| **Remix** | `build/` | 3000 | Built-in Express, integrate PostHog proxy in the Express server |

**Important**: Vite's default `outDir` is `dist/`. Some templates (notably some Readdy exports) override this to `out/`. I check `vite.config.ts` and report what the actual output directory is — Stage 3's `server.js` template needs to point at the right one.

If the framework is a non-React-non-Vite stack, I document the adaptation in this stage file's changelog as I go.

### Step 8b: I detect the styling system

Stage 8's ConsentBanner has two implementations (Tailwind-class and inline-styles). I detect which one to ship by inspecting the project:

| Signal | Result |
|---|---|
| `tailwindcss` in `package.json` `dependencies` or `devDependencies` AND `tailwind.config.{js,ts,cjs,mjs}` exists | `styling.system = "tailwind"` |
| `@emotion/*` or `styled-components` in deps | `styling.system = "css-in-js"` (treat as `plain` for ConsentBanner — inline-styles version is the safest baseline) |
| Plain `.css` files imported with `import "./foo.css"` and no Tailwind detected | `styling.system = "plain"` |
| Webflow export (Stage 0 flagged) | `styling.system = "plain"` (Webflow's own CSS handles the rest of the site; the banner is independent) |
| Framer export | `styling.system = "plain"` |

I save `styling.system` to `<project>/.skill-config.json` and Stage 8 reads it to pick the right ConsentBanner implementation.

**If the user has no styling system at all** (rare; would be a hand-rolled HTML site), I offer to install Tailwind in this stage (~90 seconds) so the rest of the skill can lean on its conventions. The user accepts/declines; declining means inline-styles everywhere and a slightly less polished aesthetic.

## Common platform-specific bugs I fix automatically

If Stage 0 identified one of these origin platforms, I apply the relevant fixes during Step 4 / Step 5 above without extra prompting.

### Readdy-origin fixes

#### Bug R1 — Vite port hardcoded

Readdy's `vite.config.ts` ships with `port: 3000` literal. I change to read from `process.env.PORT`:

```ts
server: {
  port: process.env.PORT ? Number(process.env.PORT) : 3000,
  host: "0.0.0.0",
}
```

Without this, Railway's `PORT` env var is ignored at dev preview time.

#### Bug R2 — Missing scroll-to-top on route change

Readdy uses react-router-dom without the standard scroll-restoration wrapper. I add the standard `useEffect([pathname], () => window.scrollTo(0,0))` to `src/router/index.ts` (or wherever `useRoutes` is called). This same `useEffect` is where the manual `posthog.capture('$pageview', ...)` will go in Stage 6.

#### Bug R3 — Empty `<title>` + placeholder favicon

Readdy's `index.html` ships with empty title + the Vite default `vite.svg` favicon. I update with:
- The brand name and tagline as `<title>` (using values from onboarding)
- A real favicon path (or remove the placeholder line if `public/favicon.ico` exists)
- Full OpenGraph + Twitter card metadata (required for affiliate URL unfurling at Stage 10):

```html
<meta name="description" content="<from onboarding's brand description>">
<meta property="og:title" content="<from onboarding>">
<meta property="og:description" content="<from onboarding>">
<meta property="og:image" content="https://<your-domain>/og-image.png">
<meta property="og:url" content="https://<your-domain>/">
<meta property="og:type" content="website">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="<from onboarding>">
<meta name="twitter:description" content="<from onboarding>">
<meta name="twitter:image" content="https://<your-domain>/og-image.png">
```

If the user doesn't have an OG image yet, I leave a placeholder URL and flag it in the punchlist for Stage 12.

### Webflow-origin fixes

- **jQuery dependency**: Webflow exports include `webflow.js` which depends on jQuery. I leave both in place by default — they work fine. If the user wants to rewrite the interactive parts (animations, dropdowns) in vanilla JS / React, I flag it as Stage 12 polish.
- **Form action URLs**: Webflow forms POST to `https://webflow.com/api/v1/form/...`. I note these in the punchlist; Stage 5 replaces them.
- **Webflow-specific CSS classes** (`w-form`, `w-button`, `w-container`): safe to keep (just CSS).

### Framer-origin fixes

- **Asset CDN**: `framerusercontent.com` URLs throughout. Stage 0 catalogued these; Stage 2 self-hosts them.
- **Font CDN**: Framer injects from `fonts.framer.app`. I migrate to Google Fonts or self-hosted in Stage 2.
- **Code components**: Framer's "code components" export as React components but may reference internal Framer APIs. I audit each one; some are pure React (keep), some need rewriting (flag in punchlist).

## Verification (autonomous)

I run all of these myself before declaring Stage 1 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Build succeeds | `npm run build` | Output directory populated, no errors |
| Dev server starts cleanly | `npm run dev` (then I curl localhost) | HTTP 200 returned |
| Type-check passes (if TypeScript) | `npm run type-check` (or `npx tsc --noEmit`) | Zero errors |
| Lint passes (if linter configured) | `npm run lint` | Zero errors (warnings OK to defer) |
| Git is clean | `git status` | Nothing to commit |

I report results to the user as a brief summary:

```
✅ Stage 1 complete:
   • Project at ~/projects/mayasconsulting-marketing/
   • Pushed to github.com/maya-username/mayasconsulting-marketing
   • npm run build: success (output in dist/, 247 KB gzipped)
   • npm run dev: starts on port 5173, returns 200
   • Type-check: 0 errors
   • Git status: clean

Ready for Stage 2 (platform migration)?
```

If any verification fails, I debug within this stage and don't advance.

## Common gotchas I handle automatically

- **`npm run check` vs `npm run type-check`**: some projects use `check`. I verify the actual script name in `package.json` before running.
- **`--legacy-peer-deps` may be required** if the source platform pinned old peer deps (some AI-generator outputs hit this with React 19 + older plugin versions). I detect the failure mode and retry with `--legacy-peer-deps`, documenting in `package.json` notes.
- **Cross-platform shell**: I use my Bash tool which abstracts away Windows/macOS/Linux differences. The user doesn't need to know about PowerShell vs bash distinctions.

## Outputs

After Stage 1:
- A clean repo on GitHub with one initial commit
- `.gitignore` + `.env.example` at root with appropriate placeholders
- Verified working baseline build
- All Stage 0 platform-specific scaffolding bugs fixed (per-platform)
- A summary report ready for the user

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. |
| 2026-05-08 | Reframed for non-coder audience: every "Run X" / "Edit Y" instruction now first-person Claude voice ("I run...", "I write..."). Removed author-specific workspace path conventions. Cleaned `.env.example` template to use generic `<your-domain>` placeholder + standard `phc_xxxxxxxxxxxx` key shape. Added explicit note that Vite's default outDir is `dist/` (some Readdy templates override to `out/`); Stage 3's server.js needs to point at whichever is actually present. Verification section reframed from user-runs-bash to "I run these checks; here's the summary I report." Stripped author-specific git-worktree gotcha. |
