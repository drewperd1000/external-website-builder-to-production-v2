# Stage 15: Google Search Console Property + Sitemap Automation

**Last verified: 2026-05-21.** Re-search before this stage if older than 60 days. Search: `Google Search Console API webmasters v3 2026`, `google-auth-oauthlib 2026 release notes`.

> **🤖 Re-grounding (re-read at start of stage)**: This is a Claude-driven skill. I (Claude) run the `gsc_sitemap_submit.py` helper script using the project's `oauth_helper.py` for authentication. The user clicks through Google Search Console's property-verification UI once (no API for that — Google requires human consent to verify ownership) and pastes any TXT verification record into DNS. After that, sitemap submit / list / replace operations are autonomous.

**This stage only runs if `aio.content_hub = true`.** Sites without a content hub don't need GSC integration — there's no organic discovery surface to monitor.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json`. Stage 15 needs:

| Field | Used for | If missing |
|---|---|---|
| `project.site_url` | GSC property identifier | Inherited from onboarding. |
| `aio.gsc_property_type` | URL-prefix vs Domain property | Blocking. Ask: "Do you want to verify a Domain property (covers `https://`, `http://`, all subdomains) or a URL-prefix property (just `https://<your-domain>`)?" Default: Domain (broader coverage). |
| `aio.gsc_owner_email` | Which Google account owns the property | Blocking if not the same Gmail as `oauth_helper.py` is authenticated against. See "Additional-owner pattern" below. |
| `aio.gsc_sitemap_url` | Full URL of the sitemap to submit | Deferrable. Derived from `project.site_url + "/sitemap-index.xml"`. |
| `.shared/secrets/google-oauth-client.json` | OAuth client credentials | Blocking. If missing, see "Prerequisites — OAuth setup" below. |
| `.shared/secrets/google-oauth-token.json` | Cached refresh token | Auto-created on first OAuth consent. |

**Parallel work I can do while waiting**: while the user works through GSC property verification (UI-only step), I prepare the sitemap-submit command and verify the OAuth helper has the GSC scopes configured.

## What I'm doing in this stage

Setting up Google Search Console so the marketing site's sitemap is registered, the property is verified, and ongoing re-pings happen via a scriptable CLI:

1. Verify the OAuth helper at `.shared/scripts/oauth_helper.py` includes the GSC scopes (`webmasters.readonly` + `webmasters`). If not, add them and re-trigger consent.
2. Verify the `gsc_sitemap_submit.py` helper at `.shared/scripts/gsc_sitemap_submit.py` is present and importable. If not, copy from the skill scaffold (see Step 2 below).
3. The user verifies the GSC property in the Google Search Console UI (one-time, no API for this).
4. I submit the sitemap via `python gsc_sitemap_submit.py submit <site> <sitemap-url>`.
5. I verify the submission via `python gsc_sitemap_submit.py list <site>`.
6. I document the re-ping protocol — when to re-submit (every cornerstone publish + every batch of new posts).
7. I optionally schedule a recurring re-ping via the user's task scheduler (default off — manual re-ping at publish time is usually sufficient).

## Why this stage matters (plain-language)

Google Search Console is the single most useful read-write surface for understanding how Google sees the site. It tells you which queries are surfacing the site, which pages are indexed, which have errors, and (post-cornerstone-publish) what fraction of AI Overview citations come from the site. None of that data is available without GSC.

Submitting the sitemap to GSC is the difference between "Google might discover the cornerstones in 1–14 days via crawl" and "Google discovers within hours and queues for indexing." For a freshly-launched marketing site, the gap matters: every day of late discovery is a day of citations going to competitors instead.

The automation matters because cornerstones publish on an ongoing cadence. Re-pinging GSC manually after each publish is the kind of thing that gets forgotten. The script makes it one command.

## Prerequisites

### OAuth setup (one-time, per-machine)

The `oauth_helper.py` script lives at `C:\Users\aperd\Claude_Projects\.shared\scripts\oauth_helper.py` (or equivalent on macOS/Linux). It expects credentials at `.shared/secrets/google-oauth-client.json` and caches a refresh token at `.shared/secrets/google-oauth-token.json`.

If the credentials file doesn't exist, the user needs to:

1. **Open Terminal and go to** [Google Cloud Console](https://console.cloud.google.com/) → Select a project → APIs & Services → Credentials.
2. **Click "Create Credentials" → "OAuth client ID" → "Desktop app"** → name it (e.g., "Claude Code helper") → Create.
3. **Download the JSON.** Save it to `.shared/secrets/google-oauth-client.json`.
4. **Enable the required APIs** for the project: Google Drive API, Google Sheets API, Google Docs API, Google Search Console API. (Each one has an "Enable" button on its console page.)
5. **Add the user's Gmail to the OAuth consent screen's "Test users" list** (if the project is in "Testing" status, which is the default for personal-use credentials).

Once the JSON is in place, the first invocation of `oauth_helper.get_credentials()` opens a browser, prompts for consent (covering all 5 scopes), and caches the refresh token. Subsequent invocations are silent (auto-refresh).

### GSC scopes in `oauth_helper.py`

I check `oauth_helper.py`'s `SCOPES` list:

```python
SCOPES = [
    "https://www.googleapis.com/auth/drive",                       # existing
    "https://www.googleapis.com/auth/spreadsheets",                # existing
    "https://www.googleapis.com/auth/documents",                   # existing
    "https://www.googleapis.com/auth/webmasters.readonly",         # ← GSC read
    "https://www.googleapis.com/auth/webmasters",                  # ← GSC write (sitemap submit)
]
```

If the two GSC scopes are missing, I add them and delete the cached token (`.shared/secrets/google-oauth-token.json`) so the next call re-prompts for consent with the expanded scope set.

## Automation surface I have available

| Tool | What it does | Required for |
|---|---|---|
| `python <path>/oauth_helper.py` | Trigger initial OAuth consent | First run only |
| `python <path>/gsc_sitemap_submit.py sites` | List verified properties on the authenticated account | Step 3 verification |
| `python <path>/gsc_sitemap_submit.py list <site>` | List sitemaps submitted to a property | Step 5 verification |
| `python <path>/gsc_sitemap_submit.py submit <site> <sitemap-url>` | Submit a sitemap | Step 4 |
| `python <path>/gsc_sitemap_submit.py remove <site> <sitemap-url>` | Remove a sitemap | Cleanup operations |
| `python <path>/gsc_sitemap_submit.py replace <site> <old> <new>` | Atomic-ish remove-then-submit | Rare; e.g., when sitemap URL changes |
| Local scheduled-task MCP (if connected) | Schedule recurring re-ping (Local task) | Optional Step 7 |

## My execution sequence

### Step 1: Verify OAuth helper scopes

```bash
# Check current scopes
grep -A 10 'SCOPES = \[' C:/Users/aperd/Claude_Projects/.shared/scripts/oauth_helper.py
```

If `webmasters.readonly` and `webmasters` are present, skip to Step 2.

If missing, I add them:

```python
SCOPES = [
    "https://www.googleapis.com/auth/drive",
    "https://www.googleapis.com/auth/spreadsheets",
    "https://www.googleapis.com/auth/documents",
    "https://www.googleapis.com/auth/webmasters.readonly",
    "https://www.googleapis.com/auth/webmasters",
]
```

Then I delete the cached token to force re-consent:

```bash
rm C:/Users/aperd/Claude_Projects/.shared/secrets/google-oauth-token.json
```

I run a small Python one-liner to trigger consent and cache the new token:

```bash
python -c "
import sys
sys.path.insert(0, 'C:/Users/aperd/Claude_Projects/.shared/scripts')
from oauth_helper import get_credentials
creds = get_credentials()
print('OK — new token cached with', len(creds.scopes), 'scopes')
"
```

This opens a browser. The user clicks through the consent screen (5 scope checkboxes, all should be pre-checked). Token caches; subsequent runs are silent.

### Step 2: Verify `gsc_sitemap_submit.py` is present

```bash
ls -la C:/Users/aperd/Claude_Projects/.shared/scripts/gsc_sitemap_submit.py
```

If missing, I copy the scaffold from this skill's `examples/gsc_sitemap_submit.py` reference (see Step 2a for the script contents).

#### Step 2a: gsc_sitemap_submit.py reference

The full helper script (paste into `.shared/scripts/gsc_sitemap_submit.py` if missing):

```python
"""
Google Search Console sitemap helper.

Subcommands:
  sites                          — list verified GSC properties
  list <site>                    — list sitemaps for a property
  submit <site> <sitemap-url>    — submit a sitemap
  remove <site> <sitemap-url>    — remove a sitemap
  replace <site> <old> <new>     — remove old + submit new

<site> accepts:
  bare domain (e.g. "mayasconsulting.com")
  sc-domain property (e.g. "sc-domain:mayasconsulting.com")
  URL-prefix property (e.g. "https://mayasconsulting.com/")

<sitemap-url> must be a full URL (e.g. "https://mayasconsulting.com/sitemap-index.xml").
"""

import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent))
from oauth_helper import get_credentials
from googleapiclient.discovery import build


def get_service():
    creds = get_credentials()
    return build("webmasters", "v3", credentials=creds)


def list_verified_sites(service):
    resp = service.sites().list().execute()
    return resp.get("siteEntry", [])


def resolve_site(service, site_arg):
    """Resolve a bare domain to sc-domain or URL-prefix form."""
    if site_arg.startswith("sc-domain:") or site_arg.startswith("http"):
        return site_arg
    # bare domain — look it up
    sites = list_verified_sites(service)
    for site in sites:
        url = site["siteUrl"]
        if url == f"sc-domain:{site_arg}" or url == f"https://{site_arg}/":
            return url
    raise ValueError(
        f"No verified GSC property found for '{site_arg}'. "
        f"Verified properties: {[s['siteUrl'] for s in sites]}"
    )


def cmd_sites():
    service = get_service()
    for site in list_verified_sites(service):
        print(f"  {site['siteUrl']}  (permissionLevel: {site['permissionLevel']})")


def cmd_list(site_arg):
    service = get_service()
    site = resolve_site(service, site_arg)
    resp = service.sitemaps().list(siteUrl=site).execute()
    sitemaps = resp.get("sitemap", [])
    if not sitemaps:
        print(f"No sitemaps submitted for {site}.")
        return
    for sm in sitemaps:
        last_submitted = sm.get("lastSubmitted", "never")
        warnings = sm.get("warnings", 0)
        errors = sm.get("errors", 0)
        print(f"  {sm['path']}")
        print(f"    last submitted: {last_submitted}")
        print(f"    warnings: {warnings}, errors: {errors}")


def cmd_submit(site_arg, sitemap_url):
    service = get_service()
    site = resolve_site(service, site_arg)
    service.sitemaps().submit(siteUrl=site, feedpath=sitemap_url).execute()
    print(f"Submitted {sitemap_url} to {site}")


def cmd_remove(site_arg, sitemap_url):
    service = get_service()
    site = resolve_site(service, site_arg)
    service.sitemaps().delete(siteUrl=site, feedpath=sitemap_url).execute()
    print(f"Removed {sitemap_url} from {site}")


def cmd_replace(site_arg, old_url, new_url):
    cmd_remove(site_arg, old_url)
    cmd_submit(site_arg, new_url)


def main():
    if len(sys.argv) < 2:
        print(__doc__)
        sys.exit(1)
    cmd = sys.argv[1]
    args = sys.argv[2:]
    handlers = {
        "sites": lambda: cmd_sites(),
        "list": lambda: cmd_list(*args),
        "submit": lambda: cmd_submit(*args),
        "remove": lambda: cmd_remove(*args),
        "replace": lambda: cmd_replace(*args),
    }
    handler = handlers.get(cmd)
    if not handler:
        print(f"Unknown command: {cmd}")
        print(__doc__)
        sys.exit(1)
    try:
        handler()
    except TypeError as e:
        print(f"Wrong number of arguments for '{cmd}': {e}")
        print(__doc__)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

This file is intentionally placed at `.shared/scripts/` (outside any individual project repo) so it's reusable across projects. The `oauth_helper.py` it depends on lives in the same directory.

### Step 3: User verifies the GSC property (one-time UI step)

There's no API for verifying property ownership. The user does this in the GSC UI:

```
Please open https://search.google.com/search-console in your browser (logged
in as the Google account that should own this property).

Click "Add property" (top-left dropdown).

Choose property type:
  • Domain property — covers https://<domain>, http://<domain>, and all
    subdomains. Verification = adding a TXT record to DNS.
  • URL-prefix property — covers just https://<your-domain>. Verification =
    uploading an HTML file, adding an HTML meta tag, or several other options.

I (Claude) recommend Domain property for broader coverage. For Domain
property verification, you'll get a TXT record value like
"google-site-verification=abc123xyz". Add it to your DNS:

  Type: TXT
  Name: @ (or root)
  Value: google-site-verification=abc123xyz
  TTL: 3600 (or "auto")

If your DNS is on Cloudflare and you granted me a CF API token at onboarding,
I can add the TXT record for you — paste the verification value here and
I'll add it.

After the TXT record is added, click "Verify" in the GSC UI. Verification
typically takes 1–5 minutes for DNS propagation.

Tell me when verification succeeds.
```

If `hosting.cloudflare_token = granted` at onboarding, I add the TXT record via the Cloudflare API when the user pastes the verification value:

```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/<zone-id>/dns_records" \
  -H "Authorization: Bearer $(cat .shared/secrets/cloudflare-api-token.txt)" \
  -H "Content-Type: application/json" \
  --data '{"type":"TXT","name":"@","content":"google-site-verification=<verification-value>","ttl":3600}'
```

### Step 4: Submit the sitemap

```bash
python C:/Users/aperd/Claude_Projects/.shared/scripts/gsc_sitemap_submit.py submit \
  mayasconsulting.com \
  https://mayasconsulting.com/sitemap-index.xml
```

Expected output:
```
Submitted https://mayasconsulting.com/sitemap-index.xml to sc-domain:mayasconsulting.com
```

If the resolver can't find the property, that means verification (Step 3) hasn't completed yet or the OAuth helper is authenticated against a different Google account. See "Additional-owner pattern" below.

### Step 5: Verify the submission

```bash
python C:/Users/aperd/Claude_Projects/.shared/scripts/gsc_sitemap_submit.py list \
  mayasconsulting.com
```

Expected output:
```
  https://mayasconsulting.com/sitemap-index.xml
    last submitted: 2026-05-21T17:30:00.000Z
    warnings: 0, errors: 0
```

Warnings >0 are usually drafts accidentally included (unlikely if Stage 13's filter is working) or 404s from URLs that no longer exist. Errors >0 means the sitemap is malformed; I rebuild and resubmit.

### Step 6: Document the re-ping protocol

I write a one-pager to the project repo at `docs/aio-gsc-protocol.md`:

```markdown
# GSC Sitemap Re-Ping Protocol

Run after:
- Every cornerstone publish (draft: true → false, committed + pushed)
- Every batch of weekly-post publishes (>3 posts in a week)
- Every sitemap URL change (e.g., domain migration)

Command:
\`\`\`bash
python C:/Users/aperd/Claude_Projects/.shared/scripts/gsc_sitemap_submit.py submit \\
  mayasconsulting.com \\
  https://mayasconsulting.com/sitemap-index.xml
\`\`\`

Verify:
\`\`\`bash
python C:/Users/aperd/Claude_Projects/.shared/scripts/gsc_sitemap_submit.py list \\
  mayasconsulting.com
\`\`\`

Look for `lastSubmitted` updated to today's date. Google indexes new URLs from
the sitemap within ~24 hours of re-submission (often much faster).
```

### Step 7: (Optional) schedule recurring re-ping

For sites that publish frequently and don't want to remember the manual command, I can set up a daily scheduled task — typically via a Local scheduled-task MCP if the user has one connected (Claude Code's `scheduled-tasks` MCP, or cron on Linux/macOS, or Task Scheduler on Windows). The recipe is:

- **Cron expression**: `0 4 * * *` (4am local time) — early-morning re-pings catch Google's morning crawl window.
- **Command**: `python <path>/gsc_sitemap_submit.py submit <site> <sitemap-url>` then `python <path>/gsc_sitemap_submit.py list <site>` to report the new `last submitted` timestamp.
- **Notify on completion**: usually false. Successful re-pings are routine; only failures matter, and the script exits non-zero on those, which the scheduler can flag separately.

This is opt-in — for most sites, ping-at-publish-time (manual re-run after each cornerstone publish) is sufficient and avoids the noise of daily completion notifications.

## Additional-owner pattern (when the GSC property is on a different Google account)

If the user's GSC property is owned by a different Google account than the one `oauth_helper.py` authenticates as (e.g., the user verified `mayasconsulting.com` on their personal Gmail, but the OAuth helper auths as their work Gmail), the `list` and `submit` calls return "no verified property" errors.

**The fix**: GSC supports multiple owners per property. The user goes to GSC → Settings → Users and permissions → Add user → enters the Gmail the OAuth helper authenticates as → assigns Owner permission. The OAuth-authenticated account now sees the property as if it were the verified owner.

I detect this scenario when `gsc_sitemap_submit.py sites` doesn't list the user's expected property. I prompt:

```
The OAuth helper is authenticated as <oauth-email>, but I don't see
<your-property> in the verified-properties list for that account.

If <your-property> is verified under a different Google account, you can
add <oauth-email> as an Owner so the helper can manage it:

  1. Open https://search.google.com/search-console (logged in as the
     account that owns <your-property>)
  2. Select the <your-property> property
  3. Settings → Users and permissions → Add user
  4. Email: <oauth-email>
  5. Permission: Owner
  6. Add

Then re-run `python gsc_sitemap_submit.py sites` to confirm.
```

## Verification (autonomous)

I run these myself before declaring Stage 15 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| OAuth helper has GSC scopes | `grep webmasters .shared/scripts/oauth_helper.py` | Both scopes present |
| `gsc_sitemap_submit.py` exists | `ls .shared/scripts/gsc_sitemap_submit.py` | Present + executable |
| GSC property verified | `python gsc_sitemap_submit.py sites` | Property listed with `permissionLevel: siteFullUser` or `siteOwner` |
| Sitemap submitted | `python gsc_sitemap_submit.py list <site>` | Sitemap appears with `last submitted` timestamp ≤ 1 hour old |
| Sitemap has no errors | Inspect the `list` output | `errors: 0` |
| Sitemap URL is reachable | `curl -I https://<your-domain>/sitemap-index.xml` | HTTP 200 |
| Sitemap content is valid XML | `curl -s https://<your-domain>/sitemap-index.xml \| xmllint --noout -` | No errors |
| Drafts excluded from sitemap | `curl -s https://<your-domain>/sitemap-index.xml \| grep -c '<draft-slug>'` | 0 matches per draft slug |
| Re-ping protocol doc exists | `ls docs/aio-gsc-protocol.md` | Present |

## Common gotchas I handle automatically

- **`Bad Request` on submit**: usually means the sitemap URL is unreachable from Google's crawlers (e.g., behind Basic Auth on staging, behind a `noindex` header, behind a `robots.txt` Disallow). Stage 12's launch checks include a sitemap-reachability check that catches this before Stage 15 runs.
- **OAuth token has stale scopes**: if I added GSC scopes after the token was first cached, the cached token still represents the old scope set and `gsc_sitemap_submit.py` returns auth errors. The fix is to delete `.shared/secrets/google-oauth-token.json` and re-trigger consent.
- **`sc-domain:` vs URL-prefix property gotcha**: a sitemap submitted under a URL-prefix property (`https://mayasconsulting.com/`) does NOT cover the Domain property (`sc-domain:mayasconsulting.com`). If both property types are verified for the same domain, GSC shows two distinct sitemaps. Submit to whichever property type the user wants to track. The resolver in `gsc_sitemap_submit.py` prefers `sc-domain:` if both are verified.
- **GSC requires the sitemap to live on the same domain as the property**: `https://example.com/sitemap.xml` cannot be submitted to `sc-domain:mayasconsulting.com`. If the user has a CDN-hosted sitemap, ensure the URL matches the property's domain.
- **DNS verification takes 1–5 minutes**: I tell the user not to expect instant verification after adding the TXT record. If it's been >10 minutes and still failing, I check DNS propagation with `dig TXT <domain>` and walk through TXT record troubleshooting.

## Self-research instruction

Before this stage:

1. Web-search `Google Search Console API webmasters v3 deprecation` — Google has indicated long-term plans to move from `webmasters` API to `searchconsole` v1. If a deprecation date is announced and within 6 months, switch `gsc_sitemap_submit.py` to the v1 API and append to Changelog.
2. Web-search `Google Search Console sitemap submission rate limits 2026` — sitemap re-submission has rate limits that have shifted in the past. If new limits are documented, update Step 7's recommended cron frequency.
3. Web-search `google-auth-oauthlib release notes 2026` — auth library breaking changes have happened before. Verify `oauth_helper.py` still works with the latest version.

## Outputs

When Stage 15 completes, the user has:

- GSC property verified for `<project.site_url>` (Domain or URL-prefix per user choice)
- `oauth_helper.py` configured with GSC scopes (in addition to Drive/Sheets/Docs)
- `gsc_sitemap_submit.py` present at `.shared/scripts/` and callable from any project
- Sitemap submitted to the property; `last submitted` timestamp ≤ 1 hour old; zero errors
- `docs/aio-gsc-protocol.md` documenting the re-ping protocol
- Optional: Local scheduled task for daily re-ping (if user opted in)

After Stage 15, every cornerstone publish or draft-flip triggers a re-ping that gets the new URL into Google's index within hours. The PostHog dashboard for "organic search traffic" (Stage 11) will start showing data within 1–7 days for new cornerstones; longer-tail keywords accumulate over weeks.

## Changelog

| Date | Change |
|---|---|
| 2026-05-21 | Initial. Captures the GSC property-verification flow (user UI step, no API available), the sitemap submission via `gsc_sitemap_submit.py` (lives at `.shared/scripts/` for cross-project reuse), the additional-owner pattern for multi-Google-account setups, the OAuth scope-update protocol for adding GSC to an existing `oauth_helper.py`, and the optional Local scheduled task for daily re-ping. Documents the Cloudflare API path for autonomously adding the TXT verification record when the user granted a CF token at onboarding. |
