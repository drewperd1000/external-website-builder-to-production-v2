# Reference: Object Storage on Railway — Buckets vs B2 vs Alternatives

**Last verified: 2026-05-08** against Railway Buckets docs and Backblaze B2 pricing/feature pages. Re-verify if older than 60 days.

This reference is consulted when the user has a storage-of-files question — uploads, media libraries, generated artifacts (e.g., variant screenshots from Stage 11), or anything where bytes need to live somewhere and be retrievable later.

> **🤖 When I read this**: at any stage where the user needs to store files. Most commonly Stage 11 (variant screenshot archive) and any Stage 9 / app-tier work where users upload content. If onboarding flagged "no file storage needed," I skip this entire decision.

## TL;DR

**Default**: Railway Buckets, unless the app needs publicly-accessible URLs that browsers and CDNs can cache.

**Switch to Backblaze B2 (or any S3-compatible alternative — Cloudflare R2, AWS S3, etc.)** when:
- The files need stable public URLs (e.g., a media library where browsers cache by URL across visits, a shared image gallery, a podcast feed, a CDN-fronted asset library)
- The app needs features Railway hasn't shipped yet (versioning, lifecycle rules, server-side encryption, object lock)
- The user is already operating at a scale where storage-cost math overrides operational simplicity

**Default-when-uncertain**: If the public-vs-private question is genuinely unclear, I default to **Backblaze B2** for stability — public-bucket-capable infrastructure forecloses fewer future choices than private-only infrastructure does.

## The two options at a glance

### Option A — Railway Buckets (native to the Railway project)

- **Same invoice, same dashboard, same auth model** as the rest of the user's Railway services.
- **S3-compatible API**. Drop-in for any code already using `@aws-sdk/client-s3` or equivalent. Standard CRUD, listing, multipart upload, presigned URLs, tagging.
- **Pricing as of 2026-05**: $0.015/GB-month storage; free unlimited API operations; **free egress from buckets** (this is the headline advantage).
- **Per-environment isolation**: prod and staging environments each get separate bucket instances with separate credentials automatically.

**Hard limitations** I always surface to the user:
- **Private only.** Public buckets are not supported. All read access requires either presigned URLs or a backend proxy.
- No versioning, no lifecycle rules, no server-side encryption, no object lock.
- Single region per bucket; no built-in geo-replication.
- Bucket egress is free, but **service egress is not**. If the app proxies bucket reads through an Express / Next / Fastify service back to the user, that traffic counts as paid Railway service egress — which defeats the "free egress" advantage. The path that's actually free is direct browser-to-bucket via presigned URL.

### Option B — Backblaze B2 (or any S3-compatible alternative — R2, S3, etc.)

- **External vendor** — separate account, separate billing, separate keys to rotate.
- **S3-compatible API**. B2 specifically requires `forcePathStyle: true` for the AWS SDK; R2 and S3 use the default subdomain style.
- **B2 pricing as of 2026-05**: ~$6/TB-month storage (≈$0.006/GB), ~$0.01/GB egress with the first 3× storage size free per day, very cheap API operations.
- **Public buckets supported**. With `bucketType: allPublic`, every object is fetchable from a stable URL like `https://<bucket>.s3.<region>.backblazeb2.com/<key>`. CDN-friendly out of the box.
- Versioning, lifecycle rules, server-side encryption, object lock all available.
- **B2 quirk**: CORS rules are CLI-only — the web console doesn't expose them. Not a deal-breaker, just something to know. R2 and S3 expose CORS in their dashboards.

## Decision criteria — the four questions

I walk the user through these four. The combination maps cleanly to one of the two options.

### 1. Do the files need public, stable URLs that don't expire?

A "public, stable URL" is one like `https://media.<your-domain>/podcast/episode-42.mp3` that any browser can fetch directly, that doesn't change between visits, and that a CDN can cache.

| Concrete examples | Verdict |
|---|---|
| Podcast feed (RSS points to media files; podcast apps fetch them by stable URL) | **Public** → B2 |
| Public image gallery / portfolio site (browsers cache by URL across visits) | **Public** → B2 |
| Marketing OG images / favicon assets | **Public** → B2 (or just commit to repo) |
| Shared content library played repeatedly across many users (audio tracks, video clips, image-heavy explainers) | **Public** → B2 |
| User uploads private documents to their own account; downloads them via signed link | **Private** → Railway |
| App generates a one-off PDF and emails the user a download link valid for 24h | **Private** → Railway |
| Variant screenshot archive (only the developer + maybe a small team views these) | **Private** → either works; I default to Railway since it's same-invoice |

### 2. What's the egress-to-storage ratio?

- **Heavy egress** — users repeatedly re-download the same files (media playback in particular, where one file is fetched many times across a userbase): **Railway's free egress wins at scale**; B2's free 3× window helps but caps out at heavy traffic.
- **Mostly storage, light egress** — files written once, read rarely (backups, archives, cold data): **B2 is ~2.5× cheaper on storage alone**.

### 3. Does the app need versioning, lifecycle rules, or server-side encryption?

- **Yes** (e.g., the user wants old versions of files preserved for 30 days; or wants files auto-deleted after 90 days; or has a compliance requirement for at-rest encryption): **B2** (or R2, or S3 — all support this).
- **No**: Railway is fine.

### 4. Will the project cross 50 GB stored or have significant monthly egress soon?

- **Yes**: I run actual numbers; the picture changes at scale, and I'll flag the tradeoff in concrete dollar terms.
- **No**: either works; I pick the one with less operational surface, which is Railway (because it's same-invoice, same dashboard, same auth flow).

## The caching gotcha — the one nuance that decides most cases

This is the single most important concept in the whole decision, and it's not obvious from either vendor's docs:

**With public URLs (B2 / R2 / public S3)**:
- The URL is stable: `https://media.example.com/library/file-42.mp3`
- Browsers HTTP-cache the file by URL
- Subsequent plays / page visits / re-renders use the cached bytes — zero re-download
- CDNs cache the same way at edge nodes

**With presigned URLs (Railway Buckets, or any private S3-compatible bucket)**:
- The URL changes per request: `https://bucket.example.com/library/file-42.mp3?X-Amz-Signature=abc123&X-Amz-Date=...`, then a different signature next time
- Browsers treat each presigned URL as a different resource — every request triggers a full re-download, even of identical bytes
- CDNs are similarly defeated unless query strings are stripped at the CDN layer (which requires custom CDN config and breaks the security model presigning provides)

**Practical impact**:
- For files users access once (private uploads, one-off downloads, password-reset PDFs): the caching gotcha doesn't matter.
- For shared library content played repeatedly across users (a track played by 1,000 listeners, an image rendered on every page-load, a video streamed in a course): every play is a fresh download. Real bandwidth cost, real perceived-load latency, real CDN-bypass.

**Workarounds exist** but each adds operational surface:
- Service Worker that strips signature query params from cache keys
- "Stable wrapper URL → 302 redirect to fresh presigned URL" pattern (you cache the redirect target only)
- Tokenized HMAC URLs validated by your own backend (homegrown public-with-auth)

**The one-line rule**: if browsers and CDNs caching files by stable URL is part of the performance model, the project needs a public bucket — which today means an external vendor (B2, R2, or S3).

## "Let me decide" mode — Claude picks based on saved config

If the user says some variant of "you pick" / "decide for me" / "use your best judgment," I read `<project>/.skill-config.json` and recommend with rationale. The decision logic I apply:

```
1. Read these fields from saved config:
   - project.brand_type / origin.platform (clues about content type)
   - membership.platform (Stage 9 may imply user uploads)
   - variants.archive_approach (Stage 11 may imply screenshot storage)
   - storage.use_case (if onboarding asked the question explicitly)

2. Apply this decision tree:
   - storage.use_case == "none"           → Skip; no bucket needed
   - storage.use_case == "private_uploads" → Railway Buckets
   - storage.use_case == "public_media"    → Backblaze B2
   - storage.use_case == "both"            → Backblaze B2 for everything
                                            (consolidate; running two storage backends
                                             roughly doubles operational surface)
   - storage.use_case == "variant_archive_only" → Railway Buckets
                                                  (low traffic, private, same-invoice convenience)

3. If storage.use_case is unset OR if I can't infer with high confidence:
   default to Backblaze B2.
   Rationale: B2 is public-bucket-capable, so it forecloses fewer future
   choices. If the user later realizes they need public URLs, they don't
   need to migrate. The reverse — Railway → B2 — is a real migration with
   real re-upload time.
```

I report the recommendation with the rationale (one or two sentences) before provisioning anything. The user can override.

## Migration considerations — only if user is already on B2/S3 and asks about Railway

Surface this only when relevant. Most projects start fresh and don't need it.

- **Code change is trivial**: both are S3-compatible; swap endpoint and credentials.
- **Data re-upload is real**: depending on volume, hours to days of one-off transfer time.
- **CORS rules need re-establishing** on the new bucket.
- **Public buckets cannot be migrated 1:1** — see the caching gotcha. They either stay on the original vendor or adopt a workaround.
- **Per-environment isolation is a Railway feature** — if the current setup uses one bucket for prod + staging, the user will need to either accept Railway's separate-buckets-per-env model or replicate it on the original vendor.
- **Billing relationship moves to Railway's invoice** — could matter for accounting / vendor-diversification posture.

**My default recommendation**: for an app already on B2 that's working, don't migrate just to migrate. Re-evaluate when one of these triggers fires:
1. Railway ships public buckets natively (closes the structural blocker for libraries).
2. Egress on the current provider crosses a threshold where consolidating to one invoice outweighs the re-upload + re-config cost.
3. The current provider's missing features become a real gap (rare).

## What I ask the user when the decision isn't already settled

If onboarding didn't capture the storage decision (older saved configs may not have the field), I ask these four questions in plain language:

```
A few questions to figure out the right place to store files for this site:

  1. What kind of files are we talking about?
     (a) Private user uploads — documents, photos, account-scoped content
         that should NOT be browseable by anyone with a link
     (b) Public shared media — a library of files (images, audio, video,
         downloads) where stable URLs that browsers and CDNs can cache
         are part of how the site feels fast
     (c) Both — some private, some public
     (d) None — I just need somewhere to drop variant screenshots from
         Stage 11 and that's it

  2. Are you already on another storage provider (Backblaze, Cloudflare R2,
     AWS S3) for an existing project, or starting fresh?

  3. Do you need any of: file versioning (keep old versions for 30 days),
     lifecycle rules (auto-delete after N days), server-side encryption,
     or object-lock compliance? (Most projects don't.)

  4. (Optional) Do you have actual numbers for storage size and monthly
     egress, or are you greenfield? Greenfield is fine — I'll size it
     for you.
```

The combination of those four answers maps cleanly to one option.

## Saved-config schema additions

When the user picks (or I pick), I save to `<project>/.skill-config.json`:

```json
"storage": {
  "needed": true,
  "use_case": "private_uploads" | "public_media" | "both" | "variant_archive_only" | "none",
  "provider": "railway_buckets" | "backblaze_b2" | "r2" | "s3" | "none",
  "bucket_name": "<auto-generated>",
  "is_public": false,
  "decided_by": "user" | "claude_default" | "claude_inferred"
}
```

`decided_by` is for transparency: when I default to B2 because the user said "you decide" and I had no signal, the config records that, so a future session knows the choice was a Claude-default rather than a user-stated preference. That matters if the user later questions the choice — I can say "I picked this because I had no public-vs-private signal and B2 forecloses fewer options; happy to switch."

## Cross-references

- **[stage-11-variants-and-dashboards.md](../stage-11-variants-and-dashboards.md)** — variant screenshot archive (most common storage need for marketing sites)
- **[stage-9-membership-optional.md](../stage-9-membership-optional.md)** — file uploads tied to subscriber accounts
- **[reference-railway-automation.md](./reference-railway-automation.md)** — Railway MCP / CLI for bucket provisioning (`deploy-template` for Railway Buckets; `set-variables` for the auto-injected `BUCKET_*` env vars)

## Self-research instruction

Re-verify pricing + feature claims every 60 days. Search:
- `Railway Buckets pricing <year>` and `Railway Buckets public buckets` (the most likely thing to change is whether public buckets ship)
- `Backblaze B2 pricing <year>` (egress price tiers shift occasionally)
- `Cloudflare R2 pricing <year>` (R2 is the closest competitor to both — if pricing or features shift, the decision tree may need a third primary option)

Append findings to this file's Changelog.

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Adapted from a separate research session. Captures the Railway Buckets vs Backblaze B2 decision framework, the caching gotcha that makes most public-content cases tip toward B2, the four-question decision walk, the "let me decide" mode where Claude reads saved config and recommends, and the default-to-B2 rule when the public-vs-private question is uncertain (rationale: B2 forecloses fewer future choices than Railway-private does). Audio is referenced only as one of several concrete examples (alongside images, video, documents) — not the primary case. |
