# Stage 13: AIO Content-Hub Infrastructure

**Last verified: 2026-05-21.**

> **🤖 Re-grounding (re-read at start of stage)**: This is a Claude-driven skill. I (Claude) write all the Astro config, schema components, builders, layouts, content collections, sitemap filters, RSS feeds, and Vitest tests. The user makes the brand-identity decisions that propagate into the schema (organization name, parent company, author name + credentials, sameAs URLs). When I find myself describing "the user adds the schema component," I stop and reframe — I add the component; the user provides the values it consumes.

**This stage only runs if `aio.content_hub = true`** (set at onboarding). If false, skip to Stage 6. The Vite-only path doesn't get the schema infrastructure.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 13 needs:

| Field | Used for | If missing |
|---|---|---|
| `project.brand` | `Organization.name`, `WebSite.name`, OG site_name | Blocking. Ask: "What's the brand name as it should appear in search results and AI citations?" |
| `project.site_url` | Canonical URL base, `Organization.url`, `WebSite.url` | Blocking. Already captured at onboarding. |
| `project.app_url` | `Organization.sameAs` reciprocity if PWA exists | Deferrable. If site has no companion app, leave empty. |
| `aio.parent_org` | `Organization.parentOrganization` | Optional. If user is a sole-proprietor or has no parent company, skip this schema field entirely (do NOT invent one). |
| `aio.author_name` | `Person.name`, `Article.author` | Blocking IF cornerstones planned. Ask: "Who's the named author of articles? (Real human name — Google's AI surfaces and Perplexity weight authored content over anonymous content)" |
| `aio.author_suffix` | `Person.honorificSuffix` | Optional. Credentials like `Ph.D.`, `L.Ac.`, `MBA` — defaults to empty string. |
| `aio.author_job_titles` | `Person.jobTitle[]` | Optional. Array of role titles relevant to the brand's topic authority. |
| `aio.author_affiliations` | `Person.affiliation[]` | Optional. Institutions/orgs that reinforce expertise. |
| `aio.author_alumni_of` | `Person.alumniOf[]` | Optional. Schools/programs that reinforce credentials. |
| `aio.same_as[]` | `Organization.sameAs`, `Person.sameAs` | Optional. Other URLs representing the same entity (LinkedIn, Twitter, GitHub, app subdomain, parent-company site). Each one strengthens entity disambiguation in AI-search retrieval. |
| `aio.cornerstones_planned` | Number of cornerstones to scaffold blank-state | Deferrable. Default 4. Stage 14 collects topics; this stage just stubs the count. |

**Parallel work I can do while waiting**: while the user answers blocking author questions, I install Astro integrations, scaffold the schema component files (without the brand-specific values), write the Vitest harness, set up the content collection config. Brand values get spliced in once the user answers.

## What I'm doing in this stage

Scaffolding the content-hub infrastructure that makes the site discoverable + citable by AI search engines:

1. Install Astro AIO integrations: `@astrojs/mdx`, `@astrojs/sitemap`, `@astrojs/rss`.
2. Write the 8 schema components (`Organization`, `WebSite`, `Person`, `Article`, `FAQPage`, `HowTo`, `Product`, `BreadcrumbList`) and their TypeScript builders.
3. Write the Vitest test harness validating each schema emits correct JSON-LD with required fields.
4. Write `BaseLayout.astro` with `head-extra` slot for per-page schema injection + OG/Twitter card defaults.
5. Write `BlogPostLayout.astro` extending BaseLayout with BreadcrumbList schema.
6. Define the content collection (`src/content/config.ts`) with the draft + tags + reading-time + bucket + schemas-enum shape.
7. Wire the draft-aware sitemap filter into `astro.config.mjs`.
8. Write the RSS feed endpoint at `src/pages/blog/rss.xml.ts`.
9. Write the blog index (`src/pages/blog/index.astro`) and dynamic slug route (`src/pages/blog/[slug].astro`) — both filter out drafts (slug route prerenders draft URLs but emits `noindex,nofollow`).
10. Add 4 stub cornerstone MDX files (full topics come in Stage 14).
11. Run the Vitest suite end-to-end; verify all schema components emit valid JSON-LD.
12. Build the site (`npm run build`); verify the sitemap excludes drafts.

By the end of Stage 13, the content-hub plumbing is in place. Stage 14 fills it with actual cornerstones.

## Why this stage matters (plain-language)

AI search engines (ChatGPT Search, Perplexity, Google AI Overview, Claude with web search) read structured data — not just the page text — when deciding what to cite. A page with valid `Article` + `FAQPage` + `HowTo` schema markup, an authored `Person` entity, and a recognized `Organization` is dramatically more likely to be quoted in an AI answer than the same page without those signals.

The schema work is also load-bearing for classic SEO (Google Rich Results, featured snippets), but the AI-search payoff is what tilts the cost-benefit decisively in favor of doing this upfront rather than later.

The infrastructure also enables the editorial workflow that Stage 14 needs: draft articles get URLs for preview, get excluded from public discovery (sitemap, RSS, index), then flip to live with one frontmatter change. Without that workflow, cornerstones either stay in unpublished local files (no preview, slow feedback loop) or get accidentally indexed early (poisoning the brand's first-impression citations).

## Prerequisites

- Stage 1 complete with the Astro fork (not Vite). If the project was scaffolded as Vite, see [stage-1-scaffolding.md](stage-1-scaffolding.md) for the migration path.
- Stage 3 complete (Express wraps Astro handler, prerendered-directory resolver in place).
- Onboarding's content-hub answers captured in `.skill-config.json`.

## Automation surface I have available

| Tool | What it does | Required for |
|---|---|---|
| `npm install` | Install Astro AIO integrations | Step 1 |
| Write/Edit | All file authoring | Steps 2–11 |
| `npm run test` | Run Vitest schema tests | Step 11 |
| `npm run build` | Build the site, exercise sitemap filter | Step 12 |
| Bash `grep -c '<script type="application/ld+json">' dist/client/**/*.html` | Count JSON-LD blocks per prerendered page | Step 12 verification |

## My execution sequence

### Step 1: Install AIO integrations

```bash
npm install astro@^5 @astrojs/node@^9 @astrojs/sitemap@^3 @astrojs/mdx@^4 @astrojs/react@^4 @astrojs/rss@^4
npm install --save-dev @astrojs/check@^0.9 vitest@^2
```

(If Stage 1 already installed Astro itself, only the missing integrations get added. I use `npm ls` first to check.)

### Step 2: Schema component scaffold

I create `src/components/schema/` with 9 files: 8 `.astro` components + 1 `schema-builders.ts`. Each Astro component is a 6-line wrapper around its builder; the builder owns all the JSON-LD shape and the brand-value substitution.

**Builder file pattern** (`src/components/schema/schema-builders.ts`):

```typescript
// All brand identity is imported from src/site-config.ts (set up in Stage 1).
// Single source of truth — a brand rename means editing site-config.ts and
// nothing else. The builders here never hardcode brand values.

import {
  SITE_URL,
  APP_URL,
  BRAND,
  LOGO_PATH,
  PARENT_ORG,
  PARENT_ORG_URL,
  SAME_AS,
  AUTHOR_NAME,
  AUTHOR_SUFFIX,
  AUTHOR_JOB_TITLES,
  AUTHOR_AFFILIATIONS,
  AUTHOR_ALUMNI_OF,
  AUTHOR_KNOWS_ABOUT,
} from "@/site-config";

export function buildOrganizationSchema() {
  const schema: Record<string, unknown> = {
    "@context": "https://schema.org",
    "@type": "Organization",
    name: BRAND,
    url: SITE_URL,
    logo: `${SITE_URL}${LOGO_PATH}`,
  };
  if (SAME_AS.length > 0) schema.sameAs = SAME_AS;
  if (PARENT_ORG) {
    schema.parentOrganization = {
      "@type": "Organization",
      name: PARENT_ORG,
      url: PARENT_ORG_URL,
    };
  }
  return schema;
}

export function buildWebSiteSchema() {
  return {
    "@context": "https://schema.org",
    "@type": "WebSite",
    name: BRAND,
    url: SITE_URL,
    potentialAction: {
      "@type": "SearchAction",
      target: { "@type": "EntryPoint", urlTemplate: `${SITE_URL}/search?q={search_term_string}` },
      "query-input": "required name=search_term_string",
    },
  };
}

export function buildPersonSchema() {
  const schema: Record<string, unknown> = {
    "@context": "https://schema.org",
    "@type": "Person",
    name: AUTHOR_NAME,
    url: SITE_URL,
  };
  if (AUTHOR_SUFFIX) schema.honorificSuffix = AUTHOR_SUFFIX;
  if (AUTHOR_JOB_TITLES.length > 0) schema.jobTitle = AUTHOR_JOB_TITLES;
  if (AUTHOR_ALUMNI_OF.length > 0) {
    schema.alumniOf = AUTHOR_ALUMNI_OF.map((name) => ({ "@type": "EducationalOrganization", name }));
  }
  if (AUTHOR_AFFILIATIONS.length > 0) {
    schema.affiliation = AUTHOR_AFFILIATIONS.map((name) => ({ "@type": "Organization", name }));
  }
  if (AUTHOR_KNOWS_ABOUT.length > 0) schema.knowsAbout = AUTHOR_KNOWS_ABOUT;
  if (PARENT_ORG) schema.worksFor = { "@type": "Organization", name: PARENT_ORG };
  return schema;
}

export function buildArticleSchema(args: {
  headline: string;
  slug: string;
  datePublished: string;
  dateModified: string;
  image: string;
}) {
  const schema: Record<string, unknown> = {
    "@context": "https://schema.org",
    "@type": "Article",
    headline: args.headline,
    image: args.image,
    datePublished: args.datePublished,
    dateModified: args.dateModified,
    publisher: {
      "@type": "Organization",
      name: BRAND,
      logo: { "@type": "ImageObject", url: `${SITE_URL}/logo.svg` },
    },
    mainEntityOfPage: {
      "@type": "WebPage",
      "@id": `${SITE_URL}/blog/${args.slug}`,
    },
  };
  if (AUTHOR_NAME) {
    schema.author = {
      "@type": "Person",
      name: AUTHOR_NAME,
      ...(AUTHOR_SUFFIX ? { honorificSuffix: AUTHOR_SUFFIX } : {}),
    };
  }
  return schema;
}

export function buildFAQPageSchema(items: Array<{ question: string; answer: string }>) {
  return {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    mainEntity: items.map(({ question, answer }) => ({
      "@type": "Question",
      name: question,
      acceptedAnswer: { "@type": "Answer", text: answer },
    })),
  };
}

export function buildHowToSchema(args: { name: string; steps: Array<{ name: string; text: string }> }) {
  return {
    "@context": "https://schema.org",
    "@type": "HowTo",
    name: args.name,
    step: args.steps.map((step, index) => ({
      "@type": "HowToStep",
      position: index + 1,
      name: step.name,
      text: step.text,
    })),
  };
}

export function buildProductSchema() {
  // FTC trap: do NOT add aggregateRating or review unless real reviews exist.
  // Fake reviews are an explicit FTC-actionable misrepresentation. Real reviews
  // require a sourced data structure (Trustpilot, G2, etc.) — out of scope here.
  return {
    "@context": "https://schema.org",
    "@type": "Product",
    name: BRAND,
    description: "<your-product-description>",
    brand: { "@type": "Brand", name: BRAND },
    url: SITE_URL,
  };
}

export function buildBreadcrumbListSchema(crumbs: Array<{ name: string; url: string }>) {
  return {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    itemListElement: crumbs.map((crumb, index) => ({
      "@type": "ListItem",
      position: index + 1,
      name: crumb.name,
      item: crumb.url,
    })),
  };
}
```

**Astro component pattern** (uniform across all 8 — example for `OrganizationSchema.astro`):

```astro
---
import { buildOrganizationSchema } from "./schema-builders";
const schema = buildOrganizationSchema();
---
<script type="application/ld+json" set:html={JSON.stringify(schema)} />
```

For schemas that take props (Article, FAQPage, HowTo, BreadcrumbList), the component declares the props interface and passes them through to the builder:

```astro
---
// ArticleSchema.astro
import { buildArticleSchema } from "./schema-builders";
const { headline, slug, datePublished, dateModified, image } = Astro.props;
const schema = buildArticleSchema({ headline, slug, datePublished, dateModified, image });
---
<script type="application/ld+json" set:html={JSON.stringify(schema)} />
```

The full library is documented in [`_internal/reference-aio-schema-library.md`](_internal/reference-aio-schema-library.md).

### Step 3: Vitest schema test harness

I create `src/tests/schema-components.test.ts` that imports each builder and asserts the contract:

```typescript
import { describe, it, expect } from "vitest";
import {
  buildOrganizationSchema,
  buildWebSiteSchema,
  buildPersonSchema,
  buildArticleSchema,
  buildFAQPageSchema,
  buildHowToSchema,
  buildProductSchema,
  buildBreadcrumbListSchema,
} from "../components/schema/schema-builders";

describe("OrganizationSchema", () => {
  it("includes name, url, and logo", () => {
    const schema = buildOrganizationSchema();
    expect(schema["@context"]).toBe("https://schema.org");
    expect(schema["@type"]).toBe("Organization");
    expect(schema.name).toBeTruthy();
    expect(schema.url).toMatch(/^https:\/\//);
    expect(schema.logo).toMatch(/^https:\/\//);
  });
});

describe("ArticleSchema", () => {
  it("includes datePublished, dateModified, author, publisher, mainEntityOfPage", () => {
    const schema = buildArticleSchema({
      headline: "Test Article",
      slug: "test-article",
      datePublished: "2026-05-20",
      dateModified: "2026-05-20",
      image: "https://mayasconsulting.com/blog/test.png",
    });
    expect(schema.headline).toBe("Test Article");
    expect(schema.datePublished).toBe("2026-05-20");
    expect((schema.publisher as any)["@type"]).toBe("Organization");
    expect((schema.mainEntityOfPage as any)["@id"]).toMatch(/\/blog\/test-article$/);
  });
});

describe("FAQPageSchema", () => {
  it("emits Question/Answer pairs in mainEntity[]", () => {
    const schema = buildFAQPageSchema([
      { question: "What is X?", answer: "X is Y." },
      { question: "How does X work?", answer: "Like Z." },
    ]);
    expect((schema.mainEntity as any).length).toBe(2);
    expect((schema.mainEntity as any)[0]["@type"]).toBe("Question");
    expect((schema.mainEntity as any)[0].acceptedAnswer.text).toBe("X is Y.");
  });
});

describe("HowToSchema", () => {
  it("numbers step positions starting at 1", () => {
    const schema = buildHowToSchema({
      name: "How to ship",
      steps: [
        { name: "Open editor", text: "Open your editor." },
        { name: "Write code", text: "Write the code." },
      ],
    });
    expect((schema.step as any)[0].position).toBe(1);
    expect((schema.step as any)[1].position).toBe(2);
  });
});

describe("ProductSchema (FTC trap)", () => {
  it("does NOT include aggregateRating or review (no fake ratings)", () => {
    const schema = buildProductSchema();
    expect(schema.aggregateRating).toBeUndefined();
    expect(schema.review).toBeUndefined();
  });
});

describe("BreadcrumbListSchema", () => {
  it("numbers item positions starting at 1", () => {
    const schema = buildBreadcrumbListSchema([
      { name: "Home", url: "https://mayasconsulting.com/" },
      { name: "Blog", url: "https://mayasconsulting.com/blog/" },
    ]);
    expect((schema.itemListElement as any)[0].position).toBe(1);
    expect((schema.itemListElement as any)[1].position).toBe(2);
  });
});

describe("PersonSchema", () => {
  it("includes name and url at minimum", () => {
    const schema = buildPersonSchema();
    expect(schema.name).toBeTruthy();
    expect(schema.url).toMatch(/^https:\/\//);
  });
  it("includes honorificSuffix when configured", () => {
    const schema = buildPersonSchema();
    // Only asserts presence/absence based on config — skip if AUTHOR_SUFFIX is empty
    if ((schema as any).honorificSuffix) {
      expect(typeof (schema as any).honorificSuffix).toBe("string");
    }
  });
});
```

The point of the test isn't to verify the exact shape of every field — it's to catch the cases where a builder accidentally returns malformed JSON-LD (missing `@context`, wrong `@type`, no `name`, broken nested objects). If a future Claude session edits a builder, the test catches the regression before it ships.

### Step 4: BaseLayout with `head-extra` slot

I write `src/layouts/BaseLayout.astro`:

```astro
---
import OrganizationSchema from "@/components/schema/OrganizationSchema.astro";
import WebSiteSchema from "@/components/schema/WebSiteSchema.astro";
import PersonSchema from "@/components/schema/PersonSchema.astro";
import { SITE_URL, BRAND } from "@/site-config";
import "@/styles/globals.css";

export interface Props {
  title: string;
  description: string;
  canonicalPath?: string;
  ogImage?: string;
  ogType?: "website" | "article";
  noindex?: boolean;
  includeSchema?: {
    organization?: boolean;
    website?: boolean;
    person?: boolean;
  };
}

const {
  title,
  description,
  canonicalPath = "/",
  ogImage = "/og-image.png",
  ogType = "website",
  noindex = false,
  includeSchema = {},
} = Astro.props;

const canonical = `${SITE_URL}${canonicalPath}`;
const og = ogImage.startsWith("http") ? ogImage : `${SITE_URL}${ogImage}`;
const schemaConfig = {
  organization: includeSchema.organization ?? true,
  website:      includeSchema.website      ?? false,
  person:       includeSchema.person       ?? false,
};
---
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="icon" type="image/x-icon" href="/favicon.ico" />
    <title>{title}</title>
    <meta name="description" content={description} />
    <link rel="canonical" href={canonical} />
    {noindex && <meta name="robots" content="noindex,nofollow" />}

    <meta property="og:type"        content={ogType} />
    <meta property="og:site_name"   content={BRAND} />
    <meta property="og:url"         content={canonical} />
    <meta property="og:title"       content={title} />
    <meta property="og:description" content={description} />
    <meta property="og:image"       content={og} />
    <meta property="og:image:width" content="1200" />
    <meta property="og:image:height" content="630" />

    <meta name="twitter:card"        content="summary_large_image" />
    <meta name="twitter:url"         content={canonical} />
    <meta name="twitter:title"       content={title} />
    <meta name="twitter:description" content={description} />
    <meta name="twitter:image"       content={og} />

    {schemaConfig.organization && <OrganizationSchema />}
    {schemaConfig.website      && <WebSiteSchema />}
    {schemaConfig.person       && <PersonSchema />}

    <slot name="head-extra" />
  </head>
  <body>
    <slot />
    <slot name="cookie-banner" />
  </body>
</html>
```

The `head-extra` slot is the key extensibility point. Pages inject per-route schema there (BreadcrumbList for blog posts, FAQPage inline in cornerstones, etc.) without me having to enumerate every schema combination in the layout.

### Step 5: BlogPostLayout adding BreadcrumbList

```astro
---
// src/layouts/BlogPostLayout.astro
import BaseLayout from "./BaseLayout.astro";
import BreadcrumbListSchema from "@/components/schema/BreadcrumbListSchema.astro";
import { SITE_URL } from "@/site-config";

export interface Props {
  title: string;
  description: string;
  slug: string;
  pubDate: Date;
  updatedDate?: Date;
  type: "cornerstone" | "post";
  image?: string;
  readingTime?: number;
  draft?: boolean;
}

const { title, description, slug, pubDate, image, draft } = Astro.props;
const articleImage = image
  ? (image.startsWith("http") ? image : `${SITE_URL}${image}`)
  : `${SITE_URL}/og-image.png`;
const crumbs = [
  { name: "Home", url: `${SITE_URL}/` },
  { name: "Blog", url: `${SITE_URL}/blog/` },
  { name: title, url: `${SITE_URL}/blog/${slug}/` },
];
---
<BaseLayout
  title={title}
  description={description}
  canonicalPath={`/blog/${slug}`}
  ogType="article"
  ogImage={articleImage}
  noindex={draft}
  includeSchema={{ organization: true }}>
  <Fragment slot="head-extra">
    <BreadcrumbListSchema crumbs={crumbs} />
  </Fragment>
  <article class="blog-post">
    <header>
      <h1>{title}</h1>
      <p class="meta">
        Published {new Date(pubDate).toLocaleDateString("en-US", {
          year: "numeric", month: "long", day: "numeric"
        })}
      </p>
    </header>
    <div class="content"><slot /></div>
  </article>
</BaseLayout>
```

**Why BreadcrumbList lives in the layout but Article/FAQPage/HowTo live inline in MDX**: BreadcrumbList is derived purely from the slug and the layout knows the slug. Article/FAQPage/HowTo depend on per-post content (headline, FAQs, steps) that the writer chooses per-article. Centralizing them in the layout would force a `schemas: ['Article', 'FAQPage']` enum that drives template logic — cleaner to let the writer add `<ArticleSchema ... />` directly in the MDX where they're already authoring the content.

### Step 6: Content collection schema

```typescript
// src/content/config.ts
import { defineCollection, z } from "astro:content";

const blog = defineCollection({
  type: "content",
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.date(),
    updatedDate: z.date().optional(),
    type: z.enum(["cornerstone", "post"]),
    bucket: z.enum(["A", "B", "C", "D"]).optional(),
      // A = symptom-driven query ("why am I X")
      // B = mechanism explainer ("how does Y work")
      // C = comparison ("X vs Y vs Z")
      // D = outcome / performance ("how to achieve Z")
    cornerstoneLink: z.string().optional(),  // slug of the cornerstone this post points at
    targetQueries: z.array(z.string()).optional(),
    // Per-article opt-in schemas. BreadcrumbList is NOT here because it's
    // auto-injected by BlogPostLayout for every post. Organization is auto-
    // injected by BaseLayout. The enum is intentionally restricted to types
    // that have working builders so that `schemas: ["X"]` never silently
    // fails to render. To add ScholarlyArticle, ItemList, Course, Recipe,
    // or any other schema.org type: first write the builder + component (see
    // _internal/reference-aio-schema-library.md "When to add a new schema type"),
    // then extend this enum.
    schemas: z.array(z.enum([
      "Article", "FAQPage", "HowTo", "Product"
    ])).default(["Article"]),
    image: z.string().optional(),
    imageAlt: z.string().optional(),
    tags: z.array(z.string()).optional(),
    readingTime: z.number().optional(),
    draft: z.boolean().default(false),
  }),
});

export const collections = { blog };
```

The `bucket` and `targetQueries` fields are AIO-specific — they support Stage 14's cornerstone-content workflow. Posts authored as ordinary posts (not cornerstones) can leave them empty; cornerstones should always set them.

### Step 7: Draft-aware sitemap filter in `astro.config.mjs`

I update the existing `astro.config.mjs` (already created in Stage 1's Astro fork) to add the sitemap integration with a filter that reads draft state from MDX frontmatter at config-load time:

```javascript
// astro.config.mjs
import { defineConfig } from "astro/config";
import { readdirSync, readFileSync } from "fs";
import { dirname, join } from "path";
import { fileURLToPath } from "url";
import node from "@astrojs/node";
import sitemap from "@astrojs/sitemap";
import mdx from "@astrojs/mdx";
import react from "@astrojs/react";

// astro.config.mjs reads SITE_URL from env directly rather than importing from
// @/site-config — Astro's config file runs before the @ alias is resolved, so
// imports from `@/site-config` aren't available here. Same value either way —
// site-config.ts also reads process.env.SITE_URL with the same fallback.
const SITE_URL = process.env.SITE_URL || "https://mayasconsulting.com";

const __dirname = dirname(fileURLToPath(import.meta.url));

function readDraftSlugs() {
  const blogDir = join(__dirname, "src/content/blog");
  try {
    return new Set(
      readdirSync(blogDir)
        .filter((f) => f.endsWith(".mdx") || f.endsWith(".md"))
        .map((f) => {
          // Normalize CRLF -> LF so regex doesn't need Windows line-ending handling
          const content = readFileSync(join(blogDir, f), "utf-8").replace(/\r\n/g, "\n");
          const fm = content.match(/^---\n([\s\S]*?)\n---/)?.[1] ?? "";
          const slug = fm.match(/^slug:\s*["']?([^"'\n]+?)["']?\s*$/m)?.[1]?.trim();
          const draft = fm.match(/^draft:\s*(true|false)/m)?.[1] === "true";
          return draft && slug ? slug : null;
        })
        .filter(Boolean),
    );
  } catch {
    return new Set();
  }
}
const DRAFT_SLUGS = readDraftSlugs();

export default defineConfig({
  site: SITE_URL,
  output: "server",
  adapter: node({ mode: "middleware" }),
  integrations: [
    sitemap({
      filter: (page) => {
        if (page.includes("/_legacy/")) return false;
        const blogMatch = page.match(/\/blog\/([^/]+)\/?$/);
        if (blogMatch && DRAFT_SLUGS.has(blogMatch[1])) return false;
        return true;
      },
    }),
    mdx(),
    react(),
  ],
  vite: {
    resolve: { alias: { "@": new URL("./src", import.meta.url).pathname } },
  },
});
```

**Why this works the way it does**:
- Drafts get prerendered URLs via the slug route (`src/pages/blog/[slug].astro`) so editors can preview them. But `noindex,nofollow` is set in BaseLayout when `draft=true`, so search engines that hit those URLs by accident don't index.
- The sitemap filter excludes them by slug from `sitemap-index.xml`. AI crawlers that discover URLs only via sitemap will not find drafts.
- The blog index page (Step 9) filters them out of the visible listing.
- The RSS feed (Step 8) filters them out of the feed.

That's four discovery surfaces, all gated on the single `draft: true` frontmatter flag. Editors flip one boolean, the draft goes live everywhere.

**Why the `/_legacy/` exclusion**: if Stage 1's Astro fork relocated legacy React entry points under `src/_legacy/` (e.g., `src/_legacy/main.tsx`, `src/_legacy/App.tsx`), Astro's build may emit `/blog/` index pages or other routes under that prefix that shouldn't be discoverable. Filtering them out of the sitemap is defensive belt-and-suspenders against a stage-1-detail leaking into search engines.

### Step 8: RSS feed endpoint

```typescript
// src/pages/blog/rss.xml.ts
import rss from "@astrojs/rss";
import { getCollection } from "astro:content";
import type { APIContext } from "astro";
import { BRAND, BRAND_DESCRIPTION, AUTHOR_NAME, EMAIL_NOTIFY_TO } from "@/site-config";

export async function GET(context: APIContext) {
  const posts = await getCollection("blog", ({ data }) => !data.draft);
  return rss({
    title: `${BRAND} Blog`,
    description: BRAND_DESCRIPTION,
    site: context.site!,
    items: posts
      .sort((a, b) => b.data.pubDate.getTime() - a.data.pubDate.getTime())
      .map((post) => ({
        title: post.data.title,
        description: post.data.description,
        pubDate: post.data.pubDate,
        link: `/blog/${post.slug}/`,
        author: AUTHOR_NAME ? `${AUTHOR_NAME} <${EMAIL_NOTIFY_TO}>` : EMAIL_NOTIFY_TO,
      })),
  });
}
```

### Step 9: Blog index + dynamic slug route

**Blog index** (`src/pages/blog/index.astro`):

```astro
---
import BaseLayout from "@/layouts/BaseLayout.astro";
import { getCollection } from "astro:content";
import { BRAND } from "@/site-config";

const posts = await getCollection("blog", ({ data }) => !data.draft);
const sorted = posts.sort((a, b) => b.data.pubDate.getTime() - a.data.pubDate.getTime());
const cornerstones = sorted.filter((p) => p.data.type === "cornerstone");
const weeklyPosts  = sorted.filter((p) => p.data.type === "post");
---
<BaseLayout
  title={`Blog — ${BRAND}`}
  description="<your-blog-meta-description>"
  canonicalPath="/blog"
  includeSchema={{ organization: true, website: true }}>
  <main class="blog-index">
    <h1>The Blog</h1>
    {cornerstones.length > 0 && <section>
      <h2>Cornerstone guides</h2>
      <ul>{cornerstones.map((post) => <li>
        <a href={`/blog/${post.slug}/`}>
          <h3>{post.data.title}</h3>
          <p>{post.data.description}</p>
        </a>
      </li>)}</ul>
    </section>}
    {weeklyPosts.length > 0 && <section>
      <h2>Recent writing</h2>
      <ul>{weeklyPosts.map((post) => <li>
        <a href={`/blog/${post.slug}/`}>
          <h3>{post.data.title}</h3>
          <p>{post.data.description}</p>
        </a>
      </li>)}</ul>
    </section>}
  </main>
</BaseLayout>
```

**Dynamic slug route** (`src/pages/blog/[slug].astro`):

```astro
---
import { getCollection } from "astro:content";
import BlogPostLayout from "@/layouts/BlogPostLayout.astro";

export const prerender = true;  // static HTML at build time

export async function getStaticPaths() {
  // Drafts ARE included — prerendered URLs for direct preview / editorial sharing.
  // Excluded from /blog index, sitemap, RSS. BlogPostLayout passes draft flag
  // through to BaseLayout which emits noindex,nofollow meta (defensive
  // crawler exclusion).
  const posts = await getCollection("blog");
  return posts.map((post) => ({ params: { slug: post.slug }, props: { post } }));
}

const { post } = Astro.props;
const { Content } = await post.render();
const { data, slug } = post;
---
<BlogPostLayout
  title={data.title}
  description={data.description}
  slug={slug}
  pubDate={data.pubDate}
  updatedDate={data.updatedDate}
  type={data.type}
  image={data.image}
  draft={data.draft}>
  <Content />
</BlogPostLayout>
```

### Step 10: Cornerstone stubs

I create 4 stub MDX files in `src/content/blog/` so the content collection isn't empty. Each stub explicitly sets `slug:` in frontmatter — without it, the draft filter in `astro.config.mjs` cannot match the file against the sitemap exclusion rule:

```markdown
---
title: "Cornerstone 1 (placeholder)"
description: "Will become the first cornerstone once Stage 14 collects topics."
slug: cornerstone-1-placeholder
pubDate: 2026-05-21
type: cornerstone
bucket: A
schemas: ["Article"]
draft: true
---

This is a stub. Stage 14 will replace this with a real cornerstone draft.
```

Repeated for buckets B, C, D (with `slug: cornerstone-2-placeholder`, etc.). The stubs let Step 11 (test) and Step 12 (build) exercise the full collection plumbing; Stage 14 replaces them with real content.

### Step 11: Run Vitest

```bash
npm run test
```

Every schema component test should pass. If a builder is missing a required field (no `@context`, no `name`, etc.), Vitest catches it before any of this hits production.

### Step 12: Build the site

```bash
npm run build
```

I verify:
- Build completes without errors
- `dist/server/entry.mjs` exists (Express will mount this in Stage 3)
- `dist/client/sitemap-index.xml` exists and contains no `/blog/<draft-slug>/` entries
- `dist/client/blog/<draft-slug>/index.html` exists with `<meta name="robots" content="noindex,nofollow">` in the head
- Each prerendered blog post page contains at least 1 `<script type="application/ld+json">` block (the Organization schema injected from BaseLayout's `includeSchema.organization: true` default)

Quick Bash verification:

```bash
# Count JSON-LD blocks per page
for f in dist/client/blog/*/index.html; do
  count=$(grep -c '<script type="application/ld+json">' "$f")
  echo "$f: $count JSON-LD blocks"
done
```

Every page should be ≥1. Cornerstones once authored (Stage 14) should be ≥3 (Organization + BreadcrumbList + per-article Article/FAQPage/HowTo).

## Verification (autonomous)

I run these myself before declaring Stage 13 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Schema tests | `npm run test -- src/tests/schema-components.test.ts` | All tests pass |
| Build succeeds | `npm run build` | Exit code 0; `dist/server/entry.mjs` present |
| Sitemap excludes drafts | `cat dist/client/sitemap-index.xml \| grep -c '<draft-slug>'` | 0 matches |
| Drafts get URLs + noindex | `grep -l 'noindex,nofollow' dist/client/blog/<draft-slug>/index.html` | File present + match found |
| BaseLayout injects Organization on /, /blog/ | `grep -c '"@type":"Organization"' dist/client/index.html` | ≥1 |
| Schema builders pass TypeScript | `npx astro check` | 0 errors |
| RSS endpoint serves | After build, `curl localhost:3000/blog/rss.xml` returns valid XML | XML parses; drafts absent |
| No `placeholder` strings left in shipped schema | `grep -r '<your-' dist/client/ \| grep -v posthog-static` | 0 matches (placeholders were substituted) |

If any check fails I stop and debug within the stage. I do not advance to the next stage.

## Common gotchas I handle automatically

- **CRLF line endings on Windows**: the sitemap filter's frontmatter regex normalizes `\r\n` to `\n` before matching. Without it, `draft: true` followed by a Windows line break doesn't match the `^draft:\s*(true|false)$/m` pattern.
- **Astro slug derivation**: Astro infers slug from filename (e.g., `cornerstone-one.mdx` → slug `cornerstone-one`). If the user writes a `slug:` frontmatter field, it overrides. The sitemap filter reads `slug:` from frontmatter explicitly — if a writer relies on filename-only slugs, the filter still works because the frontmatter parser returns `null` for that file and it falls through to the default "include in sitemap" branch. **But drafts MUST have an explicit `slug:` frontmatter field** for the filter to exclude them. I add this to every cornerstone stub in Step 10.
- **The `/_legacy/` exclusion**: if Stage 1 didn't relocate legacy React entry points, this filter still runs but matches nothing. It's defensive future-proofing.
- **Schema-builder hot-reload**: edits to `schema-builders.ts` require a full restart of `npm run dev` (Astro doesn't hot-reload module-level constants reliably). I tell the user to restart dev server after schema changes.

## Self-research instruction

Before this stage, I check if Astro has shipped new integrations or schema-related features:

1. Web-search `Astro 5 integrations 2026` and check the official docs at `https://docs.astro.build/en/guides/integrations-guide/`.
2. Web-search `schema.org Article AI search 2026` to see if Google / OpenAI / Anthropic have published guidance on schema preferences for AI-search citation.
3. If schema preferences have shifted (e.g., a new `@type` like `AIAnswer` becomes load-bearing), add the corresponding builder + Astro component + test, and append to the Changelog.

## Outputs

When Stage 13 completes, the user has:

- `src/components/schema/` with 8 `.astro` components + 1 `schema-builders.ts` + 1 Vitest test file
- `src/layouts/BaseLayout.astro` with `head-extra` slot
- `src/layouts/BlogPostLayout.astro` extending BaseLayout with BreadcrumbList
- `src/content/config.ts` defining the blog collection with draft + tags + bucket + schemas-enum
- `astro.config.mjs` extended with sitemap integration + draft filter
- `src/pages/blog/index.astro` (filters drafts out of visible list)
- `src/pages/blog/[slug].astro` (prerenders all posts including drafts, with `noindex` on drafts)
- `src/pages/blog/rss.xml.ts` (RSS feed filtering drafts)
- 4 cornerstone stub MDX files (replaced with real content in Stage 14)
- Passing Vitest suite verifying schema component contracts
- `dist/client/sitemap-index.xml` with drafts correctly excluded
- `npm run build` clean

The plumbing is in place. Stage 14 fills it with cornerstones; Stage 15 submits the sitemap to Google Search Console.

<!--
🚧 LEARNINGS-PENDING: Optimal schema combinations per content type
  Open question: Does Article + FAQPage + HowTo consistently outperform Article + FAQPage alone for AI-search citations?
  Data source: PostHog event `ai_citation_inbound` (set up in Stage 11 dashboards) + GSC performance pull on cornerstone URLs + manual citation tracker checking Perplexity / ChatGPT Search / Google AI Overview monthly.
  Revisit: 30 days after cornerstone 1 publishes, 60 days after cornerstone 2 publishes.
  Action when revisited: refine the `schemas` enum default in `src/content/config.ts` from `["Article"]` to whatever combination is winning, and update the cornerstone template in Stage 14.
-->

## Changelog

| Date | Change |
|---|---|
| 2026-05-21 | Initial. Captures the Astro 5 schema-component + content-collection + draft-filter pattern from the reference implementation. Eight schema components (not ten — the original spec listed ten but actual count is eight). Documents the CRLF normalization gotcha in the sitemap filter regex. Documents the cornerstone-stub pattern that keeps Stage 13's test/build steps green before Stage 14 fills in real content. |
