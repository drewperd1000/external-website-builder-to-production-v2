# Stage 2: Source-Platform Dependency Migration

**Last verified: 2026-05-08.**

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) execute every migration step in this stage — downloading images, editing source files, removing dependencies, stripping platform-specific data attributes. The user's job is to (1) make the launch-blocker decisions I surface (e.g., what to do about AI-generated team photos), (2) approve any non-default migrations I propose. I do NOT ask the user to run `npm uninstall`, `curl`, or `grep` — I do that via my Bash tool.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 2 needs:
- `origin.platform` — drives which platform-specific bug fixes apply. Stage 0 sets this; if missing, Stage 0 didn't run and I block.
- The `migration-punchlist.md` file at repo root (produced by Stage 0). If missing, I run Stage 0 inline before proceeding.

## What I'm doing in this stage

Resolving every 🟡 pre-cutover and 🔴 launch-blocker item from `migration-punchlist.md` (Stage 0). After this stage, the site no longer depends on the source platform's infrastructure for anything that ships to production.

**Time**: 15–60 minutes depending on how many items the punchlist has. Most projects land in the 20–30 minute range.

## Prerequisites

- Stage 1 complete (clean repo, baseline build passing).
- `migration-punchlist.md` exists with discovered dependencies, confirmed by the user during Stage 0.

## My execution order (earlier items unblock later ones)

1. **Launch-blocker content issues** (FTC concerns, fabricated humans). Cheaper to remove than to defend.
2. **Hardcoded URLs** (logo, fonts, static assets). Quick wins; replace with self-hosted equivalents.
3. **Dynamic / AI-generated images** — download once and commit to repo, OR replace with intentional photography per the user's choice.
4. **Form endpoints** — remove platform URLs, leave a stub until Stage 5 wires real `/api/*` routes.
5. **Dependency cleanup** — remove unused npm packages.
6. **Vite / build config residue** — commented-out plugins, stale `define{}` entries, platform env-var declarations.
7. **DOM attribute leakage** — strip `data-readdy-form`, `data-formspark`, etc.
8. **Hardcoded production domains** — refactor to env var sources.

## Step 1: I resolve launch-blocker content issues

For each fabricated human / fabricated milestone / fabricated press mention identified in Stage 0, I surface a decision to the user with three options:

```
🔴 Launch blocker: TeamSection.tsx has 4 AI-generated team members
   (lines 12–48). FTC §5 + Endorsement Guides treat these as
   fabricated testimonials.

How would you like to resolve?
  (a) Remove the section entirely. I'll archive the file to src/_archived/
      so you can resurrect it later if real testimonials materialize.
  (b) Replace with a real founder bio (you're solo-founded). I'll write
      a single-person bio component using a real photo + your name + role.
      You'll need to provide a headshot URL or local file path.
  (c) Replace with stylized illustrations — graphics with no human faces.
      I'll generate placeholder gradients with your brand colors; you can
      swap in commissioned art later.
  (d) Something else — describe what you'd like.
```

After the user picks, I execute:

- **Option A**: I move the file to `src/_archived/` (preserves history), remove its imports + usages from the page that consumed it, and run a build to confirm nothing broke.
- **Option B**: I rewrite the component as a single-founder bio. If the user provides a real photo, I save it to `public/images/founder/`. Otherwise I render a styled placeholder (gradient with the founder's initial) until a real photo arrives.
- **Option C**: I commission a placeholder design or use the user's brand colors with abstract shapes.

**What I refuse to do**: replace AI faces with different AI faces; pair stock photos with the original fabricated quotes (misrepresents the stock-photo subjects, same legal category); rely on disclaimers (FTC has explicitly stated disclaimers don't cure fabricated-testimonial violations).

For fabricated press mentions ("Featured in TIME / Wired / The Guardian" with no actual coverage), I verify each citation. If none exist, I rewrite or remove. Same FTC concern.

## Step 2: I migrate hardcoded URLs (logo + static CDN)

For each hardcoded URL pointing at `static.<platform>.com` or `cdn.<platform>.com` that Stage 0 catalogued:

- I create `public/logo/` and `public/images/migrated/` if they don't exist
- I download each unique URL with `curl -o <local-path> <url>`
- I update every `src/` reference from the absolute URL to the relative local path
- I run `npm run build` to verify the build picks up the local file correctly

For fonts loaded from a platform CDN, I prefer (in order):
1. **Google Fonts** if the typeface is available there — fastest setup
2. **Self-hosted** in `public/fonts/` with `@font-face` declarations — full control, slightly more work
3. **System font stack** (`-apple-system, BlinkMacSystemFont, ...`) where the typeface isn't critical to the brand

I tell the user which I picked and why.

## Step 3: I migrate dynamic / AI-generated images

If the source platform generated images on-demand (Readdy's `?query=...&seq=...` pattern, or any `?prompt=...` style URL), every page-load currently re-fetches from their AI. **Risk**: images may re-roll on the platform's side; if the platform goes down, every image 404s simultaneously.

**My migration approach**:

1. I extract every unique URL from `src/` matching the AI-image-generator patterns (Stage 0 already produced this list)
2. I download each image to `public/images/migrated/`, deriving stable filenames from the `seq=` or `query=` parameters
3. I update each source-file reference from the platform URL to the local path
4. I write `docs/archived-platform-deps.md` with a table mapping local path → original query → which file uses it

Example of the table I produce:

```markdown
## Migrated images (originally from <platform>/api/search-image)

| Local path | Original query | Used in |
|---|---|---|
| public/images/migrated/abc123.jpg | "blue ocean at sunset" | HeroSection.tsx |
| public/images/migrated/def456.jpg | "person meditating in nature" | AboutSection.tsx |
```

If the user wants to replace any of these with real photography instead of self-hosting the AI-generated ones, they tell me which (or "all of them") and I leave those URLs in the punchlist as Stage 12 polish items.

## Step 4: I sever form-endpoint platform dependencies

For every form that currently posts to a platform-hosted URL, I replace the action with a placeholder pending Stage 5's real implementation:

```tsx
// Before (the platform's URL):
async function handleSubmit(formData: FormData) {
  await fetch("https://platform.io/api/form/abc123", {
    method: "POST",
    body: formData,
  });
}

// After (placeholder I write):
async function handleSubmit(formData: FormData) {
  // TODO Stage 5: replace with /api/newsletter
  console.log("Newsletter submit (placeholder):", Object.fromEntries(formData));
  return { ok: true };  // success UI shows but data goes nowhere yet
}
```

**Important**: I do NOT leave the platform URL active even temporarily. Once Stage 4 deploys to Railway, those submissions would still land in the platform's dashboard the user can't access — a real data-loss event. The placeholder is safer.

I add a TODO comment with the Stage 5 reference so the placeholder is obvious in code review.

## Step 5: I clean up unused dependencies

For every dependency in `package.json` that Stage 0 flagged with zero imports in `src/`, I run `npm uninstall <pkg>` (often as a single batch command for efficiency):

```
npm uninstall firebase @supabase/supabase-js @stripe/react-stripe-js recharts lucide-react
```

Common AI-generator-scaffolded-but-unused dependencies and what each was for (in case the user wants any of them later):

| Package | Original purpose | Reinstall command if needed later |
|---|---|---|
| `firebase` | Firebase SDK (Auth, Firestore, etc.) | `npm install firebase` |
| `@supabase/supabase-js` | Supabase client | `npm install @supabase/supabase-js` |
| `@stripe/react-stripe-js` | Direct Stripe Elements UI library (not needed if using a managed-checkout platform like Whop or Lemonsqueezy that hosts the payment form) | `npm install @stripe/react-stripe-js` |
| `recharts` | Chart library | `npm install recharts` |
| `lucide-react` | Icon library — namespaced imports defeat tree-shaking | `npm install lucide-react` |
| `serve` | Static-file server — replaced by `node server.js` in Stage 3 | n/a |

I document removals in `docs/archived-platform-deps.md`:

```markdown
## Removed at scaffolding cleanup

| Package | Version | Reason removed |
|---|---|---|
| firebase | 12.0.0 | 0 imports — was scaffolded but never wired |
| ... | | |
```

After removal, I run `npm run build` to verify nothing transitively-required broke.

## Step 6: I clean Vite / build config residue

### Commented-out platform plugins

I scan `vite.config.ts` for commented-out `import` lines and commented-out plugin entries:

```ts
// I find and remove patterns like:
// import { readdyJsxRuntimeProxyPlugin } from "@readdy/vite-plugin";
// ...
plugins: [
  react(),
  // readdyJsxRuntimeProxyPlugin(),  // dev-only on the platform
  AutoImport({ ... }),
],
```

I delete the import line + the commented-out plugin entry. They're dev-only on the platform; in production they don't exist anyway and uncommenting accidentally would crash the build.

### Platform-specific `define{}` entries

I delete platform-specific globals from `vite.config.ts`'s `define` block:

```ts
// I find and remove patterns like:
define: {
  __IS_PREVIEW__: false,
  __READDY_PROJECT_ID__: JSON.stringify("..."),
  __READDY_VERSION_ID__: JSON.stringify("..."),
  __READDY_AI_DOMAIN__: JSON.stringify("..."),
}
```

I also remove the matching declarations in `vite-env.d.ts` and `eslint.config.ts` (if `globals: { __PLATFORM_*: 'readonly' }` is present).

## Step 7: I strip DOM attribute leakage

I scan `src/` for platform-specific data attributes (`data-readdy-form`, `data-formspark`, `data-formsubmit`, `data-netlify-form`, etc.) and remove each match. These are markers for the source platform's form-handling JS — once Stage 5's endpoints are wired, they do nothing. Leaving them in production leaks the origin platform to anyone inspecting the DOM.

## Step 8: I refactor hardcoded production domains

If Stage 0 found hardcoded references to the user's own domain (e.g., `https://mayasconsulting.com` literally embedded in source code), I refactor each to source from an env var at build time:

```tsx
// Before (hardcoded):
<a href="https://mayasconsulting.com/login">Login</a>

// After (sourced from env var):
<a href={`${import.meta.env.VITE_SITE_URL}/login`}>Login</a>

// OR for subdomain targets:
<a href={`${import.meta.env.VITE_APP_URL}/login`}>Login</a>
```

I add `VITE_SITE_URL` and `VITE_APP_URL` to `.env.example` (already there from Stage 1) so the build picks them up.

This insulates against future DNS migrations — marketing sites commonly move apex / subdomain or migrate to a different domain entirely. Hardcoded references are a future-migration footgun.

## Step 9: I update the punchlist

I tick each 🔴 launch blocker and 🟡 pre-cutover item I resolved in this stage. Anything I couldn't resolve here, I move to 🟢 post-launch with a brief note explaining why (e.g., "Real testimonials require user to collect with permission — not a Stage 2 task").

If new dependencies surfaced during migration (common — one fix reveals another), I add them to the punchlist before advancing.

## Verification (autonomous)

I run all of these myself before declaring Stage 2 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Build still succeeds after dep removal | `npm run build` | Output directory populated, no errors |
| Dev server still works | `npm run dev` (then I curl localhost) | HTTP 200, no console errors via Preview MCP |
| No external platform URLs remain in build output | Grep `dist/` for `(?:readdy\|webflow\|framer\|wix\|squarespace)\.[a-z]+` | Empty result |
| No platform-specific DOM attributes ship | Grep `dist/` and `src/` for `data-(?:readdy\|webflow\|framer\|formspark\|formsubmit\|netlify)` | Empty result |
| Type-check still passes | `npm run type-check` | Zero errors |
| Launch blockers resolved | Read `migration-punchlist.md` | Every 🔴 ticked or explicitly accepted-with-mitigation |

I summarize the results to the user as a brief status report:

```
✅ Stage 2 complete:
   • Removed 4 AI-generated team members from TeamSection.tsx (replaced with
     a single-founder bio per your decision; placeholder gradient until you
     provide a headshot)
   • Migrated 9 images from framerusercontent.com to public/images/migrated/
   • Vendored brand logo to public/logo/
   • Stubbed 2 form endpoints (Stage 5 will wire real /api routes)
   • Removed 5 unused npm packages: firebase, @supabase/supabase-js,
     @stripe/react-stripe-js, recharts, lucide-react
   • Cleaned 1 commented-out Vite plugin + 4 platform-specific define{} entries
   • Stripped data-platform-* attributes from 2 form elements
   • Refactored 3 hardcoded domain references to use VITE_SITE_URL

Bundle size: 247 KB → 198 KB gzipped (20% reduction)
node_modules: 412 MB → 218 MB (47% reduction)

Ready for Stage 3 (Express server)?
```

If any verification fails, I debug within this stage and don't advance.

## Outputs

After Stage 2:
- `migration-punchlist.md` updated with resolutions ticked
- `docs/archived-platform-deps.md` documenting what was removed/migrated and why
- A repo with no live dependencies on the source platform's infrastructure
- Bundle size typically 30–50% smaller than baseline
- `node_modules` typically 30–50% smaller too

## Common gotchas I handle automatically

- **Removing a dep that's actually imported transitively**: `npm uninstall X` can break the dep graph if X is a peer dep of Y. I run `npm run build` after each batch removal and recover if it breaks.
- **CSS `url(...)` references to platform CDN**: greps that look at JS/JSX miss CSS-side URLs. I scan `*.css` and `*.scss` separately.
- **Image references in JSON config files**: some sites have `data.json` or `config.json` files with image URLs. I scan `*.json` separately.
- **Inline base64 images**: rare but possible. I look for `data:image/` URIs that may be remnants of the source platform's preview tools.

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. |
| 2026-05-08 | Reframed for non-coder audience: replaced "Edit X file" / "Run X command" with first-person Claude voice ("I edit...", "I run..."). Stripped author-attribution references. Replaced brand-specific name examples with generic placeholders (`<founder-name>`, `/images/founder/<name>.jpg`). Verification section reframed from user-runs-bash to "I run these checks; here's the summary I report." |
