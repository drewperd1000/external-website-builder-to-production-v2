# AIO Schema Component Library Reference

> **🤖 This file is for Claude only.** It documents the 8 schema components Stage 13 scaffolds: their prop interfaces, JSON-LD output shapes, the Vitest assertion patterns, and the FTC traps to avoid.

## When to read this file

I read this file:
- When scaffolding schema components in [Stage 13](../stage-13-content-hub.md).
- When adding a new schema type beyond the 8 baseline.
- When extending an existing component's prop interface.
- When debugging a Vitest test failure on a schema component.
- When validating that a builder satisfies its expected JSON-LD contract.

## The 8 components

| Component | Type | Props | When to use |
|---|---|---|---|
| `OrganizationSchema` | `Organization` | none (reads from builder constants) | Every page (BaseLayout default) |
| `WebSiteSchema` | `WebSite` | none | Home page, blog index (sites with internal search) |
| `PersonSchema` | `Person` | none (reads from builder constants) | Author bio page, sometimes blog post layout |
| `ArticleSchema` | `Article` | `{ headline, slug, datePublished, dateModified, image }` | Every cornerstone + blog post |
| `FAQPageSchema` | `FAQPage` | `{ items: Array<{ question, answer }> }` | Cornerstones with FAQs, dedicated FAQ pages |
| `HowToSchema` | `HowTo` | `{ name, steps: Array<{ name, text }> }` | Cornerstones with action steps, tutorial posts |
| `ProductSchema` | `Product` | none (reads from builder constants) | Product page, pricing page |
| `BreadcrumbListSchema` | `BreadcrumbList` | `{ crumbs: Array<{ name, url }> }` | Every nested page (BlogPostLayout default) |

The first three (Organization, WebSite, Person) read brand identity from module-scope constants in `schema-builders.ts` — that's where the brand name, URL, author identity, parent org, sameAs URLs live. Single source of truth.

The next four (Article, FAQPage, HowTo, BreadcrumbList) take per-instance props because each instance varies (different article headline, different FAQs, different breadcrumb trail). Product is a special case — it reads from constants because most sites have one product/service, but if your site has multiple products, you'd refactor it to take props.

## Uniform component shape

Every Astro component is a 6-line wrapper around its builder:

```astro
---
import { build<Name>Schema } from "./schema-builders";
const schema = build<Name>Schema(/* args if needed */);
---
<script type="application/ld+json" set:html={JSON.stringify(schema)} />
```

This minimal shape exists because:
- The builder owns all logic. The Astro component is a render shim.
- Testing happens against the builder (a TypeScript function), not the Astro component (which would require an Astro test runner). Vitest can call the builder directly.
- Changes to schema shape happen in one place: the builder. The component never needs editing once it exists.

## Prop interface contracts

### ArticleSchema

```typescript
buildArticleSchema(args: {
  headline: string;       // article title; goes to Article.headline
  slug: string;            // used to construct mainEntityOfPage @id
  datePublished: string;   // ISO 8601 date string ("2026-05-21")
  dateModified: string;    // ISO 8601 date string; same as datePublished if unchanged
  image: string;           // absolute URL of the article's OG image
}): ArticleSchema
```

Optional builder additions (not exposed as props because they're brand-wide):
- `author` (from `AUTHOR_NAME` + `AUTHOR_SUFFIX` constants)
- `publisher` (from `BRAND` + `${SITE_URL}/logo.svg`)
- `mainEntityOfPage` (constructed from `${SITE_URL}/blog/${slug}`)

### FAQPageSchema

```typescript
buildFAQPageSchema(items: Array<{
  question: string;  // the question text
  answer: string;    // the answer text — should match prose in the visible Q&A section
}>): FAQPageSchema
```

Critical invariant: the `answer` text in the schema MUST match the prose answer visible on the page. AI assistants compare them; mismatched FAQs look low-quality and degrade citation likelihood.

### HowToSchema

```typescript
buildHowToSchema(args: {
  name: string;                     // overall how-to name
  steps: Array<{
    name: string;                   // step name (becomes HowToStep.name)
    text: string;                   // step description (becomes HowToStep.text)
  }>;
}): HowToSchema
```

Each step gets a `position` field auto-incremented from 1. The builder also auto-types each step as `HowToStep`.

### BreadcrumbListSchema

```typescript
buildBreadcrumbListSchema(crumbs: Array<{
  name: string;  // breadcrumb label
  url: string;   // absolute URL
}>): BreadcrumbListSchema
```

The builder auto-types each crumb as `ListItem` and auto-increments `position` from 1.

## FTC traps to avoid

### ProductSchema must NOT include `aggregateRating` or `review` without real reviews

The schema.org Product type supports `aggregateRating` (numeric rating + reviewCount) and `review` (array of Review objects). Both surface in Google Rich Results as star ratings + review snippets — visually dramatic, demonstrably good for click-through rate.

**They are also one of the FTC's most-actioned misrepresentation categories.** Faked or unverified review schemas are considered "deceptive endorsements" under 16 CFR Part 255. Penalties run into millions of dollars (the Lord & Taylor case, the Sunday Riley case).

The `buildProductSchema()` function I scaffold in Stage 13 deliberately omits these fields. The Vitest test asserts their absence:

```typescript
describe("ProductSchema (FTC trap)", () => {
  it("does NOT include aggregateRating or review (no fake ratings)", () => {
    const schema = buildProductSchema();
    expect(schema.aggregateRating).toBeUndefined();
    expect(schema.review).toBeUndefined();
  });
});
```

**When the user wants to add real reviews** (e.g., they have a real Trustpilot widget, real G2 reviews, real customer testimonials with permission to publish): I help them wire the schema fields from the authentic data source. The data path matters: testimonials must be real, attributed, and verifiable. The schema field is a publication layer over real reviews, not a marketing layer that fabricates them.

### `Person.name` must be a real human

`Person` schema is for real humans, not brand personas. Google + AI search engines disambiguate `Person` entities using LinkedIn, Wikipedia, authoritative biographical sources. A made-up "About the Author" name fails entity verification and the schema loses citation weight.

If the site doesn't have a named author, leave `aio.author_name` unset and skip `PersonSchema`. Don't substitute the brand name into a `Person.name` field — that's a mismatched entity type and AI assistants notice.

### `Organization.sameAs` URLs must be reciprocal

If marketing site lists `sameAs: ["https://app.<your-domain>"]`, the app at `app.<your-domain>` should ALSO list `sameAs: ["https://<your-domain>"]` in its `Organization` schema. Asymmetric `sameAs` is suspicious to entity-disambiguation crawlers — Google's Knowledge Graph specifically uses reciprocity as a trust signal.

Same applies to social profile links: if the schema lists `sameAs: ["https://linkedin.com/in/<handle>"]`, the LinkedIn page should link back to the site. Most do by default; manual `sameAs` lists without backlink verification can accidentally include profiles that don't link back.

## Vitest assertion patterns

The test file Stage 13 scaffolds (`src/tests/schema-components.test.ts`) asserts the contract for each builder. Pattern:

```typescript
import { describe, it, expect } from "vitest";
import { build<Name>Schema } from "../components/schema/schema-builders";

describe("<Name>Schema", () => {
  it("includes <required fields>", () => {
    const schema = build<Name>Schema(/* args */);
    expect(schema["@context"]).toBe("https://schema.org");
    expect(schema["@type"]).toBe("<Type>");
    expect(schema.<field>).toBeTruthy();
  });

  it("handles <edge case>", () => {
    // ...
  });
});
```

What the tests SHOULD cover:
- `@context` and `@type` correctness
- Required-field presence (per schema.org's documented required fields)
- FTC traps (Product aggregateRating/review absence)
- Auto-incremented position fields (BreadcrumbList, HowTo)
- ISO 8601 date format on Article (regex match against `/^\d{4}-\d{2}-\d{2}/`)

What the tests SHOULDN'T cover:
- Exact JSON shape down to whitespace (over-specified, breaks on harmless reformats)
- Brand-name string values (would require updating tests on every rebrand)
- URL prefix correctness (the constants are tested at config-load time by the build itself)

## When to add a new schema type

Beyond the 8 baseline, schema.org has hundreds of types. Worth scaffolding a 9th when:

- AI-search behavior shifts to favor a new type (`ScholarlyArticle` is on the watch list for academic content; `Course` for educational sites; `Recipe` for food blogs; `Event` for events sites; `LocalBusiness` for brick-and-mortar)
- The user has a content type the 8 baseline doesn't cover (`SoftwareApplication` for SaaS, `MusicAlbum` for musicians, `MedicalCondition` / `Drug` for medical content — these need careful schema choice + may trigger Google YMYL scrutiny)

When adding a new type:

1. Add the builder function to `schema-builders.ts` following the existing pattern.
2. Add an Astro component wrapper following the 6-line pattern.
3. Add a Vitest test asserting the contract.
4. Update the `schemas` enum in `src/content/config.ts` (if the new type can be cornerstone-attached).
5. Update this reference doc with the new type's props + use cases.

## Reference: full schema.org type catalog

https://schema.org/docs/full.html

The 8 baseline covers the most-cited types for marketing-site contexts. Other useful types not yet baseline:
- `ScholarlyArticle` — for citations of original research; AI assistants weight this above `Article` for academic queries
- `WebPage` — generic page type; default if no more specific type applies (most pages should declare a more specific type instead)
- `CollectionPage` / `ItemList` — for category pages, indexes
- `ContactPage` / `AboutPage` — typed sub-classes of WebPage
- `Course` / `LearningResource` — for educational content
- `JobPosting` — for careers pages
- `LocalBusiness` (and 100+ sub-types) — for brick-and-mortar

When in doubt, check what Google's "Search Gallery" explicitly supports: https://developers.google.com/search/docs/appearance/structured-data/search-gallery — types that appear there get rich results in standard search AND are reliably extracted by AI search engines.

## Cross-references

- [Stage 13](../stage-13-content-hub.md) — scaffolds the schema component library
- [Stage 14](../stage-14-cornerstones.md) — uses ArticleSchema, FAQPageSchema, HowToSchema inline in MDX
- [Stage 12](../stage-12-launch-checks.md) — pre-launch JSON-LD audit
- [`_internal/claude-invariants.md`](claude-invariants.md) — Invariant #7 (Astro middleware order, which determines which paths get schemas)
