# Reference: Cloudflare DNS + Edge Patterns

**Last verified: 2026-05-08.**

This reference documents Cloudflare-specific patterns the skill relies on:

- `/cdn-cgi/trace` for free geo detection (Stage 8)
- Proxy vs DNS-only mode for different services
- CORS / Workers patterns
- Custom domain patterns for redirect-only domains
- Common CF gotchas (Error 1000, etc.)

## Why Cloudflare matters in this stack

- **Geo detection**: `cdn-cgi/trace` is exposed on every CF-fronted domain, free, no API key, returns ISO country code. Used by Stage 8's Standard tier banner.
- **DNS migration**: Cloudflare's DNS is fast to update (low TTL changes propagate within seconds). Used in Stage 4's domain cutover protocol.
- **SSL termination**: Cloudflare auto-provisions SSL on proxied domains. Removes the SSL-cert renewal burden.
- **Redirect rules**: Used by the branded short-URL pattern (Stage 10) for www → apex redirect with query-string preservation.
- **Page Rules / Workers**: not strictly required by this skill but useful for advanced patterns (geo-based routing, A/B testing at edge, etc.)

## Pattern 1: `/cdn-cgi/trace` for geo detection

**Endpoint**: `https://<your-domain>/cdn-cgi/trace`

**Available on**: any domain proxied through Cloudflare (orange cloud in DNS UI). NOT available on DNS-only (gray cloud) domains.

**Output format** (plain text):

```
fl=123x456
h=yourdomain.com
ip=203.0.113.42
ts=1700000000.123
visit_scheme=https
uag=Mozilla/5.0 ...
colo=SJC
sliver=none
http=http/2
loc=US        ← THIS is the country code
tls=TLSv1.3
sni=plaintext
warp=off
gateway=off
rbi=off
kex=X25519
```

**Privacy posture**: `cdn-cgi/trace` returns the visitor's IP. Don't log this server-side without a privacy basis. Reading the `loc=` line from the client (as Stage 8 does) doesn't expose IP to your server, just the visitor's own browser.

**Reliability**: free + no rate limit at consumer scale. Battle-tested in production deployments.

**Fallback**: if not on Cloudflare, alternatives:
- `ipapi.co/json/` — free tier 30k requests/day, returns `country` field
- MaxMind GeoLite2 database — self-hosted, free
- Custom IP database

## Pattern 2: Proxy vs DNS-only mode

Cloudflare's DNS UI has a "proxy status" toggle per record:

- **Proxied (orange cloud)**: traffic routes through CF's edge. SSL, DDoS protection, caching, `cdn-cgi/trace` all available. SSL auto-provisioned by CF.
- **DNS-only (gray cloud)**: CF acts as a pure DNS server. Traffic goes direct to origin. No CF features at the edge. SSL must be provisioned at origin.

### When to proxy

- Your custom domain to Railway (`yourdomain.com` → Railway service): **proxy** for SSL + DDoS protection
- Subdomains for Cloudflare Workers / Pages: proxy required (they ARE Cloudflare)

### When to keep DNS-only

- **Switchy / branded short-link domains** (Stage 10): DNS-only because Switchy provisions its own SSL via Let's Encrypt. Proxying breaks Switchy's SSL flow.
- **MX records for email**: always DNS-only (proxying email mail servers is unsupported)
- **TXT records (SPF, DKIM, DMARC)**: DNS-only by definition (TXT records aren't proxied)
- **Origin pull**: if you have a service that needs to bypass CF (rare), DNS-only that subdomain

### Verification

Use `dig` or `host` from a clean machine:

```bash
dig yourdomain.com +short
# Proxied → CF's IP (104.x.x.x or 172.x.x.x)
# DNS-only → your origin's actual IP
```

## Pattern 3: Cloudflare Error 1000 (the load-bearing reason for vendoring posthog-js dist)

**Symptom**: server-to-server fetch from Railway → `us-assets.i.posthog.com` returns:

```
Error 1000 — DNS points to prohibited IP
```

**Why**: both Railway and PostHog assets sit behind Cloudflare. CF's anti-loop heuristic detects "CF customer fetching from another CF customer" and blocks it as a potential infinite-loop attack vector.

**Fix**: don't try. Vendor the static files locally (Stage 6's `copy-posthog-static.mjs` script).

**Generalization**: any time you're tempted to make a server-side fetch from one CF-fronted domain to another, expect this. Proxy through your own backend OR vendor the assets.

## Pattern 4: Custom domain with redirect-only purpose

If you have a redirect-only domain (e.g., `www.yourdomain.com` → `yourdomain.com`), don't waste a Railway service on it. Use Cloudflare Redirect Rules.

In Cloudflare dashboard:

1. **Rules → Redirect Rules → Create rule**
2. **When incoming requests match**: `Hostname equals www.yourdomain.com`
3. **Then take action**: **Static** redirect, **301**, target URL: `https://yourdomain.com${request.uri.path}${request.uri.query}`
4. **Save and deploy**

Effects:
- `www.yourdomain.com/about?utm=test` → `yourdomain.com/about?utm=test` (path + query preserved)
- 301 (permanent) is right for SEO transfer

## Pattern 5: CORS for the same-origin proxy

If your reverse proxy is on `yourdomain.com/about-...` and your site is also on `yourdomain.com`, no CORS needed (same origin).

If you have a multi-domain setup (e.g., marketing on `yourdomain.com` + app on `app.yourdomain.com` sharing a PostHog session), you need cookies to flow:

- **PostHog distinct_id**: persisted by posthog-js with `persistence: "localStorage+cookie"`. The cookie is set on the apex domain so subdomains see it.
- **Verify**: in DevTools → Application → Cookies, the PostHog distinct_id cookie's `Domain` should be `.yourdomain.com` (with leading dot) — not `yourdomain.com` (host-only).

## Pattern 6: Cloudflare Workers (advanced, not required by this skill)

Cloudflare Workers run JS at the edge. Useful for:
- Edge-side A/B testing (different HTML for different visitors before request reaches origin)
- IP-based redirects (geo-routing)
- Custom auth at the edge
- Edge-cached responses

If you're hitting performance limits with Express + Railway and Workers solves them, the migration path is:
1. Move static asset serving to CF Pages (free)
2. Move API routes to Workers (paid above 100k requests/day)
3. Keep heavy compute on Railway

This skill doesn't require Workers; mention it as a future-optimization avenue.

## Pattern 7: Cloudflare Pages (alternative to Railway for static)

If your site is purely static (no Express server, no webhook handlers, no form endpoints), CF Pages is a viable alternative:

- Free for static hosting
- Auto-deploys from GitHub (same as Railway)
- Built-in geo + Workers integration

**Why this skill uses Railway**: most marketing sites need at least a small Express layer (PostHog reverse proxy, form endpoints). Pages doesn't support custom server logic — you'd offload to Workers, which adds complexity.

If your site genuinely has no server logic, Pages + Workers can work. The skill's stage layout adapts trivially: replace Stage 4 with "deploy to CF Pages", keep Stages 5–11 mostly the same (forms become Workers; PostHog proxy becomes a Worker).

## Pattern 8: SPF record stacking (Resend + Google Workspace, etc.)

Common gotcha when Stage 5 verifies a Resend domain on a domain that already has Google Workspace:

```
# Existing record:
yourdomain.com.  TXT  "v=spf1 include:_spf.google.com ~all"

# Resend wants you to add:
yourdomain.com.  TXT  "v=spf1 include:amazonses.com ~all"

# RFC 7208 prohibits multiple v=spf1 records on the same name.
# Merge into ONE record:
yourdomain.com.  TXT  "v=spf1 include:_spf.google.com include:amazonses.com ~all"
```

Verify post-merge:

```bash
dig TXT yourdomain.com +short
# Should show ONE v=spf1 record with all senders included
```

## Pattern 9: DNS TTL discipline

**Default**: 3600s (1 hour). Most providers default here.

**Pre-cutover**: lower to 60s 24–48h before any DNS change. Lets rollback take effect within minutes.

**Post-cutover**: raise back to 3600s after a successful 7-day window. Reduces DNS query volume + improves caching.

**Exception**: if you have services with extremely strict failover requirements, keep TTL low permanently (300s). Most consumer products don't need this.

## Common Cloudflare gotchas

### Universal SSL provisioning lag

When you add a new domain to Cloudflare (or attach a custom domain to a CF-proxied service), SSL cert provisioning takes 5–60 minutes. During that window, `https://<domain>/` returns SSL errors. Don't panic — wait it out.

### Page Rules deprecation

Cloudflare is migrating from Page Rules to Rules-based products (Redirect Rules, Cache Rules, Origin Rules, etc.). New configurations should use the Rules builder, not legacy Page Rules. As of 2026, both still work but Rules has a richer feature set.

### Email Routing requires DNS-only MX

If you're using CF's free Email Routing (forward `support@yourdomain.com` to Gmail), the MX records MUST be DNS-only (gray cloud). Auto-set by CF's Email Routing UI, but don't change them to proxied.

### `dig` cache pollution

After making DNS changes, `dig` may return stale results from your local resolver's cache. Use `dig @1.1.1.1 yourdomain.com +short` to query Cloudflare's resolver directly, or `dig +trace` to follow the chain from root. Or wait the TTL out.

### Workers KV vs DO storage cost

Cloudflare KV is cheap but eventually consistent (up to 60s lag). Durable Objects are strongly consistent but costly. Pick KV for marketing-site cache; DO for stateful logic (game state, leader boards).

## Self-research instruction

Cloudflare ships features fast. Re-verify before using:
- New Rules engine capabilities
- Workers/Pages pricing changes (they revised the model in 2024, may revise again)
- Email Routing reliability (still in "preview" historically; check status)
- Geo detection alternatives (if `cdn-cgi/trace` ever changes its output format)

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Patterns documented: cdn-cgi/trace geo detection, proxy vs DNS-only decisions, CF Error 1000 explainer (the reason posthog-js dist must be vendored), redirect rules for www → apex, CORS implications for cross-subdomain analytics, SPF record stacking, DNS TTL discipline. |
