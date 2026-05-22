# Stage 14: Cornerstone-Article Workflow

**Last verified: 2026-05-21.**

> **🤖 Re-grounding (re-read at start of stage)**: This is a Claude-driven skill — but cornerstone drafting is a hybrid effort. I (Claude) scaffold the MDX files, write the schema markup, suggest research queries, draft sections to spec, and validate that the final article emits valid JSON-LD with proper internal links. The user owns voice, claims, citations they trust, and the final go-no-go on every published sentence. Cornerstones become the highest-citation surface on the site — Claude should not generate any factual claim, statistic, or research citation without the user's review and approval.

**This stage only runs if `aio.content_hub = true`.** Stage 13 must be complete (schema components + content collection + draft-preview pattern in place).

## Required-values check (run at stage start)

I read `<project>/.skill-config.json`. Stage 14 needs:

| Field | Used for | If missing |
|---|---|---|
| `aio.cornerstones[]` | The 4 cornerstone topics + buckets + target queries | Blocking. I run the topic-collection interview (Step 1 below). |
| `aio.author_name` | `Article.author` schema + AuthorBio component | Inherited from Stage 13. |
| `project.brand` | Cornerstone publisher field | Inherited from Stage 13. |
| `aio.cornerstone_voice` | "first-person founder" / "third-person editorial" / "academic" / "conversational" — sets default voice for drafts | Deferrable. Default: "first-person founder" (most authentic for AI-search citation). |
| `aio.cornerstone_length_target` | Word count target per article. Default 2500–3500. | Deferrable. |
| `aio.article_worktree_branch` | Git branch name for cornerstone drafting | Deferrable. Default: `aio/articles`. Set up in Step 2. |

**Parallel work I can do while waiting**: while the user thinks through cornerstone topics, I prepare the article-session-vs-integration-session protocol (Step 2), set up the empty worktree, and draft the cornerstone templates with schema scaffolding ready to fill.

## What I'm doing in this stage

Producing the editorial workflow for cornerstones + scaffolding the 4 seed articles to a state the user can finish drafting:

1. Run the cornerstone topic-collection interview (4 topics, mapped to query buckets A/B/C/D).
2. Set up the article-session-vs-integration-session protocol — separate `aio/articles` branch + worktree so cornerstone drafting doesn't conflict with integration work.
3. Scaffold each cornerstone MDX file: frontmatter (with `draft: true`), inline schema components (Article + FAQPage + HowTo), section outline with H2/H3 structure aligned to query buckets, AuthorBio component placement.
4. Suggest research queries per cornerstone (for the user to validate claims against — I propose, the user approves before incorporating).
5. Draft section openings to spec — the user reviews voice + claims + citations, then either approves, edits, or rewrites.
6. Add internal links between cornerstones following the discipline below.
7. Validate emitted JSON-LD via the Vitest harness + Google Rich Results Test (Stage 12 covers the latter formally; I spot-check during drafting).
8. When a cornerstone is ready to publish: the user flips `draft: true → false`, commits, pushes — Stage 15's GSC re-ping handles the rest.

## Why this stage matters (plain-language)

Cornerstones are 2,000–4,000 word long-form articles that anchor the site's topical authority. They're what AI assistants cite when answering substantive questions in the brand's domain. A site without cornerstones has weak entity authority — even with perfect schema markup, there's no concentrated long-form content for an AI to extract and quote.

The reverse is also true: a single weak cornerstone (factually shaky, voice-mismatched, internally inconsistent, padded with filler) becomes a liability when it gets cited. AI-search citations attribute claims to the brand by name. A bad citation damages the brand's perceived credibility in ways that outlast the article itself.

So cornerstones get more care than ordinary posts. They warrant a separate drafting workflow (article sessions don't share context with implementation work), explicit user review of claims, and explicit schema validation. The draft-preview pattern from Stage 13 lets editors iterate on cornerstones in private (working URLs, no search-engine discovery) before flipping to live.

## Prerequisites

- Stage 13 complete (schema components, content collection, draft-preview pattern)
- Stage 15 can run after Stage 14 OR before — GSC is independent of cornerstone content
- The user has at least a high-level idea of the brand's topic authority (what does the brand know? what would readers come here to learn?)

## My execution sequence

### Step 1: Cornerstone topic-collection interview

If `aio.cornerstones[]` is empty, I run this interview. Phrasing adapts to `user_expertise` mode — `non_coder` gets plain-language; `developer`/`expert` gets a single batched prompt.

**Plain-language version (non_coder):**

```
We're going to set up 4 cornerstone articles. These are the long-form pieces
that AI assistants will quote when they cite the brand — so they need to be
the strongest content on the site.

Each one targets a different kind of question someone might ask AI:

  A: Symptom-driven ("Why does X happen?" / "What causes Y?")
     — these answer pain or curiosity, in the brand's domain.

  B: Mechanism explainer ("How does X work?" / "What is Y exactly?")
     — these explain the underlying science, system, or method.

  C: Comparison ("X vs Y vs Z" / "Which is better for...?")
     — these help readers choose between options.

  D: Outcome / how-to ("How do I achieve X?" / "What's the best way to Y?")
     — these are practical, action-oriented guides.

For each one, I'll need:
  - A working title (we can refine later)
  - The 3-5 queries you'd most want this article to rank for in AI search
  - Any specific claims, frameworks, research, or stories you want included

Want to start with Bucket A? What's a question your customers ask that
you wish more people had a great answer to?
```

I walk through one bucket at a time, capturing answers into `aio.cornerstones[].topic`, `.target_queries[]`, and `.notes` per bucket. Estimated time: 20–40 min of the user's time, depending on how thoroughly they want to spec each one.

**Batched version (developer/expert):**

```
4 cornerstones, one per query bucket (A=symptom, B=mechanism, C=comparison,
D=outcome). For each: working title + 3-5 target AI-search queries + any
claims/frameworks/research you want included. Reply in YAML.
```

I save the results to skill config and proceed.

### Step 2: Set up the article-session-vs-integration-session protocol

Cornerstone drafting and integration work conflict if they share a branch. Voice drifts when MDX edits sit in the same context as TypeScript schema work; integration commits get noisy when MDX edits land on the same branch. The fix is to separate them physically — two branches, two worktrees, two session contexts.

I run:

```bash
git checkout -b aio/articles
git push -u origin aio/articles
git checkout main

# Create the worktree (one-time setup)
git worktree add ../<project-slug>-articles aio/articles
```

The user now has two working directories pointing at the same repo, on different branches:
- `<project-slug>/` on `main` — for integration work (TypeScript, infrastructure, deploys)
- `<project-slug>-articles/` on `aio/articles` — for cornerstone drafting (MDX only)

**The protocol going forward**:
- **Article sessions** open Claude Code in the `-articles/` worktree. Context is MDX content, voice, schema markup, claims, research. No TypeScript edits.
- **Integration sessions** open Claude Code in the main worktree. Context is server.js, astro.config.mjs, deploys, dependencies. No MDX edits.
- **Periodic merges** (1–2× per week): article session does `git fetch && git rebase origin/main`, integration session does `git pull && git merge aio/articles` (or `git merge origin/aio/articles` if remote-only). Merges should be trivial — distinct file sets, no overlap.

If the article session and integration session need to make conflicting edits (e.g., the Astro schema component shape changes during integration, breaking a cornerstone's inline `<ArticleSchema>` props), the integration session does the article-side update on its branch and merges. The article session never edits schema component implementations.

This protocol comes from a real observation: when MDX drafting and integration code edits sit in the same session, the model's voice for both starts to blur — schema component PRs accidentally adopt the cornerstone's vocabulary, and cornerstone drafts develop the schema component's procedural cadence. Separating contexts keeps each thread crisp.

### Step 3: Scaffold each cornerstone MDX file

For each of the 4 cornerstones, I create `src/content/blog/<slug>.mdx` with this shape:

```markdown
---
title: "<your-cornerstone-title>"
description: "<your-cornerstone-meta-description>"
slug: <your-cornerstone-slug>
pubDate: 2026-05-21
type: cornerstone
bucket: A
targetQueries:
  - "<query 1>"
  - "<query 2>"
  - "<query 3>"
schemas: ["Article", "FAQPage", "HowTo"]
image: /blog/<your-cornerstone-slug>-og.png
imageAlt: "<your-cornerstone-image-alt>"
tags:
  - "<your-tag-1>"
  - "<your-tag-2>"
readingTime: 12
draft: true
---

import ArticleSchema from "@/components/schema/ArticleSchema.astro";
import FAQPageSchema from "@/components/schema/FAQPageSchema.astro";
import HowToSchema from "@/components/schema/HowToSchema.astro";
import AuthorBio from "@/components/AuthorBio.astro";

<ArticleSchema
  headline={frontmatter.title}
  slug={frontmatter.slug}
  datePublished={frontmatter.pubDate.toISOString()}
  dateModified={(frontmatter.updatedDate ?? frontmatter.pubDate).toISOString()}
  image={`https://mayasconsulting.com${frontmatter.image}`}
/>

## TL;DR

<your 2-3 sentence summary — this is what AI assistants quote when they cite the article>

## <section heading aligned to query 1>

<draft content>

## <section heading aligned to query 2>

<draft content>

### <H3 subsection if needed>

<draft content>

## How-to: <action the reader takes after reading>

<HowToSchema
  name="<your-how-to-name>"
  steps={[
    { name: "<step 1 name>", text: "<step 1 detail>" },
    { name: "<step 2 name>", text: "<step 2 detail>" },
    { name: "<step 3 name>", text: "<step 3 detail>" }
  ]}
/>

1. **<step 1 name>**: <step 1 prose>
2. **<step 2 name>**: <step 2 prose>
3. **<step 3 name>**: <step 3 prose>

## Frequently asked questions

<FAQPageSchema
  items={[
    { question: "<query 1 reframed as question>", answer: "<concise answer>" },
    { question: "<query 2 reframed as question>", answer: "<concise answer>" },
    { question: "<query 3 reframed as question>", answer: "<concise answer>" }
  ]}
/>

**<question 1>**

<answer prose — should match the FAQPageSchema answer text>

**<question 2>**

<answer prose>

**<question 3>**

<answer prose>

## Related reading

- [<cornerstone 2 title>](/blog/<cornerstone-2-slug>/) — <one-sentence why>
- [<cornerstone 3 title>](/blog/<cornerstone-3-slug>/) — <one-sentence why>

<AuthorBio />
```

**Why the structure is opinionated**:

- **TL;DR at top**: AI assistants quote concentrated summaries. A 2–3 sentence TL;DR gives them the cleanest extractable answer.
- **Section headings aligned to target queries**: Google AI Overview and Perplexity extract paragraph-level matches against query terms. Headings that mirror queries make extraction trivial.
- **`<HowToSchema>` paired with a numbered list**: the schema gets cited directly by AI assistants for "how do I X" queries; the numbered list serves human readers. Both must say the same thing — if the schema text and the prose diverge, AI citations look weird.
- **`<FAQPageSchema>` paired with bold-question / answer-paragraph prose**: same pairing logic. The schema is for crawlers; the prose is for humans. AI assistants reward FAQs that look like real FAQs to both.
- **Related reading section**: internal links between cornerstones strengthen topical authority and give crawlers a graph to traverse. See Step 6 for discipline.
- **AuthorBio at the bottom**: the named-author signal reinforces the `Article.author` schema; AI assistants weight authored content over anonymous content.

### Step 4: Suggest research queries

For each cornerstone, I generate a list of research queries the user might run to validate claims:

```
For Cornerstone 1 ("<title>"):
  - "<query 1 — what's the most-cited paper or source on this?>"
  - "<query 2 — what's the standard counter-argument I should address?>"
  - "<query 3 — what's the latest research on this topic in the last 12 months?>"
  - "<query 4 — what do AI search engines currently cite for this query?>"
  - "<query 5 — what comparable resource does the strongest competitor publish?>"
```

The user runs the queries (web, internal docs, books, podcasts, whatever they trust) and brings back claims + citations. I incorporate the ones the user approves. I do NOT invent or paraphrase claims without explicit user approval — cornerstones are too high-leverage for hallucinated citations.

If the user has the `marketing:seo-audit` skill installed, I suggest pairing it at this step to surface query-cluster opportunities the user's topics may have missed.

### Step 5: Draft section openings to spec

For each section, I write the opening 2–4 paragraphs in the configured voice (`aio.cornerstone_voice`). The user reviews and either:
- Approves as-is
- Edits inline (then I continue from their edits)
- Rewrites entirely (then I learn from the rewrite for the next section)

This is iterative. A 3,000-word cornerstone with substantial original content typically takes 4–8 user sessions, with research validation, voice refinement, and citation work happening between sessions. The article-session protocol (Step 2) keeps each session focused.

If the user has `brand-voice:enforce-voice` installed, I run it on each section after drafting to catch voice drift before the user reviews.

### Step 6: Internal-link discipline

Internal links between cornerstones form the site's topical graph. AI assistants traverse this graph during retrieval; well-linked cornerstones get cited more.

Discipline:

- **Every cornerstone links to at least 2 other cornerstones.** Not arbitrarily — the link should make semantic sense (the linked cornerstone genuinely extends or supports the current one).
- **Link text should be the linked article's actual title or a close paraphrase.** Avoid generic "click here" / "learn more" — those carry zero topical signal.
- **Anchor links target H2 sections of the destination article.** Deep-linking gives AI crawlers granular topic associations.
- **Cornerstone-to-product page links are encouraged when natural.** "Want to apply this in practice? <Link to /pricing>" is fine. Cramming product links into every paragraph is not.
- **Cornerstone-to-weekly-post links should set `cornerstoneLink: <cornerstone-slug>` in the weekly post's frontmatter** so the relationship is queryable from the content collection. Useful for the "Related reading" section auto-generation later.

<!--
🚧 LEARNINGS-PENDING: Internal-link patterns
  Open questions:
    - How dense should cornerstone-to-cornerstone links be? (Current guess: 2-4 per cornerstone)
    - Is unidirectional or bidirectional linking better for AI-search citation? (Current guess: bidirectional but with a clear primary direction)
    - How often should cornerstones link to product/pricing vs. other cornerstones?
  Data source: GSC "Top linking sites" + internal-link audit script + PostHog event funnel (cornerstone → product page → conversion).
  Revisit: 30 days after cornerstone 1 publishes. Refine this section with observed patterns from the first published cornerstones.
-->

### Step 7: Validate emitted JSON-LD

After each cornerstone is drafted (still in `draft: true`), I run:

```bash
# In the -articles/ worktree:
npm run build
node -e "
  const fs = require('fs');
  const html = fs.readFileSync('dist/client/blog/<slug>/index.html', 'utf-8');
  const matches = html.matchAll(/<script type=\"application\/ld\+json\">([\s\S]*?)<\/script>/g);
  for (const m of matches) {
    const schema = JSON.parse(m[1]);
    console.log(schema['@type'], '-', schema.headline || schema.name || '(no name)');
  }
"
```

Expected output for a fully-scaffolded cornerstone:

```
Organization - Maya's Consulting
BreadcrumbList - (no name)
Article - <cornerstone title>
FAQPage - (no name)
HowTo - <how-to name>
```

If a schema is missing, I check the MDX for missing `<...Schema>` components. If a schema has wrong field types (e.g., `datePublished` is a Date object instead of a string), I fix the builder argument format.

For the formal pre-launch JSON-LD audit, see Stage 12 (which runs the Google Rich Results Test API + schema-organic.org's validator on every prerendered URL).

### Step 8: Publish protocol

When the user is ready to publish a cornerstone:

1. **User edits frontmatter**: `draft: true → false`. Optionally bumps `updatedDate` to today.
2. **User commits + pushes** on `aio/articles` branch.
3. **Integration session merges** `aio/articles → main`. PR optional; for small teams, fast-forward merge is fine.
4. **Railway auto-deploys** main. The new cornerstone appears in `/blog`, `sitemap-index.xml`, and `rss.xml` automatically.
5. **GSC re-ping**: Stage 15's `gsc_sitemap_submit.py` re-submits the sitemap; Google notices the new URL within hours.
6. **Update the project's content-roadmap doc** (typically `docs/content-roadmap.md` or equivalent — wherever the project tracks "what's planned vs. shipped" for content). Per Principle 9 in [`_internal/reference-operating-principles.md`](_internal/reference-operating-principles.md): don't claim a cornerstone is "published" without the canonical doc reflecting the new state. A future Claude session reading the doc cold should see "Cornerstone X is live (published YYYY-MM-DD)" and not "Cornerstone X is in draft."
7. **Optional**: notify subscribers via Resend (if newsletter sequences are wired up in Stage 5).

The whole publish flow, post-drafting, takes <10 min of the user's time and ~5 min of agent wall-clock.

## Verification (autonomous)

I run these myself after each cornerstone is scaffolded (Step 3) and at end-of-stage:

| Check | What I do | Pass criteria |
|---|---|---|
| Cornerstone files exist | `ls src/content/blog/*.mdx` | 4 cornerstones present |
| All cornerstones have valid frontmatter | `npm run build` exits 0 | No content collection errors |
| Schema components present in MDX | `grep -l 'ArticleSchema' src/content/blog/<slug>.mdx` | Match for each cornerstone |
| Drafts excluded from sitemap | `grep -c '<cornerstone-1-slug>' dist/client/sitemap-index.xml` | 0 (until published) |
| Draft URLs noindex | `grep -l 'noindex' dist/client/blog/<cornerstone-1-slug>/index.html` | Match present |
| Article session worktree set up | `git worktree list` | Shows `aio/articles` worktree |
| Internal links between cornerstones | Inspect `Related reading` sections | At least 2 cornerstone-to-cornerstone links per cornerstone |
| FAQPage schema items match visible Q&A | Manual review during Step 7 | Counts match; text matches |

## Common gotchas I handle automatically

- **MDX expressions in frontmatter**: `pubDate: 2026-05-21` is parsed as a string by Astro's content collection (then coerced to Date by the Zod schema). Don't try `pubDate: new Date()` — Zod accepts the string form. Same for `updatedDate`.
- **Schema component imports in MDX**: each MDX file needs explicit `import ...Schema from "@/components/schema/...Schema.astro"` at the top. MDX doesn't auto-import. If a cornerstone uses 3 schemas, 3 imports are needed.
- **Frontmatter access inside MDX**: `frontmatter.title` works inside MDX expressions but only if the file is rendered as part of a content collection (which `[slug].astro` does). Standalone MDX file rendering doesn't expose frontmatter; use the Astro page wrapping it instead.
- **AuthorBio component**: I create `src/components/AuthorBio.astro` once in Stage 13 (or in this stage if it wasn't done). It pulls author name + suffix + bio from the same config that drives `PersonSchema`, so changing author identity in one place updates everywhere.
- **Working in the wrong worktree**: if the user accidentally edits cornerstone MDX in the integration worktree, the commit lands on `main` and bypasses the article-session protocol. I check `git branch` at the start of any cornerstone-touching session and warn if it doesn't match the worktree's expected branch.

## Self-research instruction

Cornerstone-content best practices for AI-search ranking shift faster than schema standards do. Before this stage:

1. Web-search `AI search ranking factors <current year>` and skim 2–3 recent posts from Perplexity / Google / Semrush / Ahrefs.
2. Web-search `cornerstone article SEO 2026 ChatGPT search` for the latest practitioner observations.
3. Compare against the "internal-link discipline" guidance in Step 6. If the dominant observation is shifting (e.g., "AI assistants now weight backlinks > internal links" or "AI assistants reward 5000+ word cornerstones over 3000-word"), update Step 5/6 and append to the Changelog.

## Outputs

When Stage 14 completes (cornerstones scaffolded, voice agreed, first draft of each in place even if `draft: true`), the user has:

- 4 cornerstone MDX files in `src/content/blog/` with full schema scaffolding (Article + FAQPage + HowTo)
- Article session worktree at `../<project-slug>-articles/` on `aio/articles` branch
- Internal-link graph established between cornerstones
- AuthorBio component present and consuming the same config as `PersonSchema`
- Build + Vitest + sitemap-filter all green
- Cornerstones get prerendered URLs (working previews) but stay out of sitemap / RSS / blog index while `draft: true`
- Clear publish protocol documented for when each cornerstone is ready

Stage 14 produces SCAFFOLDED cornerstones with first-draft prose. Finishing the cornerstones (final voice, validated claims, complete citations) is ongoing editorial work the user owns over weeks. The article-session protocol supports that work without it blocking other stages.

<!--
🚧 LEARNINGS-PENDING: Cornerstone topic selection criteria
  Open question: Is query-bucket alignment (A/B/C/D) actually the right segmentation, or does the data favor a different split (e.g., commercial-intent / informational / navigational)?
  Data source: GSC performance pull per cornerstone URL + manual cross-reference of which cornerstones get cited in Perplexity / ChatGPT Search / Google AI Overview.
  Revisit: 60 days after cornerstone 2 publishes (when there's enough citation data to compare buckets).
  Action when revisited: refine Step 1's bucket framing in this stage, OR replace with whatever segmentation actually predicts citation success.
-->

<!--
🚧 LEARNINGS-PENDING: Cornerstone-to-conversion measurement
  Open question: Which PostHog events meaningfully tie AI-search-driven traffic to signup / purchase outcomes?
  Data source: PostHog cohort analysis — entry referrer = AI assistant or zero-referrer (no UTMs from social/ads/email) → cornerstone page → product page → conversion.
  Revisit: 60 days after first cornerstone publishes.
  Action when revisited: add a "Cornerstone attribution" dashboard to Stage 11's set, document the events to capture in Stage 6.
-->

<!--
🚧 LEARNINGS-PENDING: Refresh cadence
  Open question: How often should cornerstones be updated? (New citations, new sub-sections, new internal links?)
  Data source: GSC "Search appearance" decay curves on cornerstone URLs + manual recheck of AI-citation surfaces.
  Revisit: 90 days after first cornerstone publishes.
  Action when revisited: this becomes the seed for a separate `aio-content-maintenance` skill that handles the ongoing refresh workflow.
-->

## Changelog

| Date | Change |
|---|---|
| 2026-05-21 | Initial. Captures the article-session-vs-integration-session protocol, the 4-cornerstone seed strategy with query-bucket alignment, the schema scaffold (Article + FAQPage + HowTo inline in MDX), the internal-link discipline, the draft-to-publish flip protocol, and the verification workflow. Flags four LEARNINGS-PENDING areas (topic selection criteria, schema combinations, internal-link patterns, cornerstone-to-conversion measurement, refresh cadence) for revisit once the reference implementation's cornerstones accumulate 30–90 days of post-publish ranking and citation data. |
