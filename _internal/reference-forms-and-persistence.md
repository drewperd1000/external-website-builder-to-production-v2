# Reference: Forms — Capture, Persistence, ESP & CRM Integration

**Last verified: 2026-05-08.** ESP/CRM API surfaces and MCP availability shift continuously — this reference codifies the decision pattern, NOT vendor-specific procedures. At integration time, I research the user's specific ESP/CRM live and generate code based on what I find that day. Pinned recipes go stale; the research-first pattern doesn't.

> **🤖 When I read this**: at Stage 5, after Stage 0 has produced `forms-manifest.md`. This doc is the framework I follow when wiring every form discovered in the manifest. Stage 5's stage file delegates the per-form decisions to this reference.

## The four-layer model for every form

Every form submission flows through up to four layers, each independently optional except where the user opts otherwise:

```
[browser]
   │
   ▼
POST /api/<form-name>          (1) endpoint generated from manifest
   │
   ▼
validate against discovered    (2) field schema from manifest
   schema (Zod / native)
   │
   ▼
┌──────────────────────────────┐
│ Layer A — DB capture (default ON unless user opts out)
│   INSERT INTO form_submissions (...) — never lost
└──────────────────────────────┘
   │
   ▼
┌──────────────────────────────┐
│ Layer B — Notification email (default ON if EMAIL_NOTIFY_TO set)
│   Resend → user's inbox so the user sees submissions in real time
└──────────────────────────────┘
   │
   ▼
┌──────────────────────────────┐
│ Layer C — ESP segment sync (only if ESP configured AND form is mapped to a list/segment)
│   user's ESP API → add subscriber to specified segment with tags
└──────────────────────────────┘
   │
   ▼
┌──────────────────────────────┐
│ Layer D — CRM sync (only if CRM configured AND form is mapped to a stage/list)
│   user's CRM API → create contact / add note / advance stage
└──────────────────────────────┘
   │
   ▼
PostHog event: <form_name>_completed   (always; powers Stage 11 funnels)
```

The four layers run in order and degrade gracefully — if Layer C fails, Layers A + B + the PostHog event still succeeded, so no data is lost.

## Defaults I apply

| Layer | Default | Rationale |
|---|---|---|
| **A. DB capture** | **ON** | Form data is the user's most important business signal. Losing it because of a misconfigured ESP is a real cost. PostgreSQL on Railway is cheap (it's the DB Stage 9 also uses if memberships are enabled). |
| **B. Notification email** | ON if `EMAIL_NOTIFY_TO` is set in Stage 5 config | Real-time visibility into submissions matters for early-stage projects. The user reads the email; the email doesn't replace persistence. |
| **C. ESP sync** | ON only if user named an ESP in onboarding AND mapped this specific form to a segment | Auto-syncing to "all subscribers" is rarely right. Users segment for a reason; per-form mapping respects that. |
| **D. CRM sync** | ON only if user named a CRM AND mapped this specific form to a stage/list | Same principle as ESP — segmentation is intentional. |

**The opt-out rule for Layer A**: Users can disable DB capture ONLY IF Layer C and/or Layer D will reliably capture every field of every submission. I confirm by checking that the ESP/CRM sync is mapped for every form in the manifest. If any form lacks an ESP/CRM mapping, DB capture remains forced-on and I tell the user why.

## The research-first pattern (do NOT hardcode vendor recipes)

When the user names their ESP or CRM, I do **live research at integration time**, not at skill-write time. The pattern:

```
1. WebFetch the vendor's developer-docs URL.
   - Mailchimp: https://mailchimp.com/developer/marketing/api/
   - Klaviyo: https://developers.klaviyo.com/en/docs/welcome
   - ConvertKit: https://developers.convertkit.com/api-reference
   - Beehiiv: https://developers.beehiiv.com/
   - HubSpot: https://developers.hubspot.com/docs/api/overview
   - Salesforce: https://developer.salesforce.com/docs
   - Pipedrive: https://developers.pipedrive.com/docs/api/v1
   - Attio: https://docs.attio.com/rest-api
   - (Any others — I find the docs URL via web search if I don't know it)

2. WebSearch for `<vendor> MCP server <current year>`.
   - More vendors are shipping MCPs each quarter. If one exists, I prefer it over the REST API
     because OAuth + tool-calling is cleaner than managing API keys.

3. WebSearch for `<vendor> Node.js SDK <current year>` or `<vendor> JavaScript client`.
   - If an official SDK exists and is current, I use it.
   - If only community SDKs exist, I evaluate (npm weekly downloads, last-updated date)
     and either use the most-maintained one or fall back to direct fetch() calls.

4. Read the vendor's SUBSCRIBE / CREATE-CONTACT operation specifically.
   - I'm looking for: required fields, optional fields, idempotency semantics,
     rate limits, error response shape, and (critically) how the vendor models
     the user's segments/lists/tags.

5. Read the vendor's TAG / SEGMENT / CUSTOM-FIELD model.
   - This is where ESPs and CRMs differ most. I learn the vocabulary the user
     already uses in their account, then map our form fields → their fields.
```

Only AFTER doing this research do I generate code. The generated code references the user's actual segment IDs / list IDs / pipeline stages — values I either fetch via the API after they grant credentials, or ask them for once during the Stage 5 mapping step.

**What I do NOT do**: copy-paste a generic Mailchimp snippet from a tutorial. Mailchimp's API has been through Marketing v3 → Mandrill → various deprecations; what worked in 2023 isn't what works in 2026. The same is true of every other vendor in this space.

## Stage 5 user-facing flow (the parts that hit the user)

After Stage 0 has produced `forms-manifest.md`, Stage 5 walks the user through:

### Step F1 — DB capture confirmation (asked once, applies to all forms)

```
Default: every form submission is captured into a `form_submissions` table
in your project's PostgreSQL DB on Railway. This costs nothing extra (the
DB you already have for memberships / general use) and means form data is
NEVER lost — even if the ESP / CRM sync hits an outage.

OK to keep DB capture on?
  (a) Yes — capture everything to DB (recommended; default)
  (b) No — I have a CRM that captures everything reliably; skip DB
      (I confirm this only after we've mapped every form to your CRM
       in step F4. Until then, DB capture stays on.)
```

If (a): I provision PostgreSQL (via Stage 9's existing `getDb()` helper if it's already been set up for memberships, or via `mcp__railway__deploy-template { template: "postgres" }` if not), create the `form_submissions` table, and set `DATABASE_URL` on Railway.

### Step F2 — Per-form review

For each form in `forms-manifest.md`, I show the user:

```
Form: newsletter_signup (route: /, fields: email)
  Inferred purpose: newsletter_optin
  Suggested handler:
    - DB capture: YES (form_submissions table)
    - Notification email: YES → hello@<your-domain>
    - ESP sync: <ESP> → which segment?
    - CRM sync: skip (low-intent)
    - PostHog event: newsletter_signup_completed

  Confirm or adjust:
```

Repeat per form. The user typically accepts the suggestions for low-intent forms (newsletter) and customizes high-intent forms (request-demo, get-a-quote — these usually want CRM sync).

### Step F3 — ESP credential + research

If the user has an ESP and we're syncing any form to it:

1. I research the ESP's current API per the pattern above.
2. I tell the user the path to grab their API key from the ESP's dashboard, with click-through directions.
3. The user pastes the key into `<project>/.secrets/<esp>-api-key.txt`. I save it to Railway.
4. I fetch the user's segments/lists via the ESP's API and present them as a pick-list per form.

```
I just connected to your Mailchimp account. Here are the segments you have:
  1. All Subscribers (1,247 contacts)
  2. Prospects (313 contacts)
  3. Customers (94 contacts)
  4. VIP (12 contacts)

For form `newsletter_signup`, which segment should new submissions land in?
  (default: "All Subscribers")
```

I save the per-form segment mapping to `forms.per_form_routing` in skill config.

### Step F4 — CRM credential + research

Same pattern as F3 if a CRM is configured. CRMs typically need additional decisions:
- Which pipeline / object type (HubSpot: contacts vs. deals; Salesforce: leads vs. opportunities)
- Which stage / lifecycle to set on creation
- Which custom fields to populate

I research these via the CRM's API + show the user a structured mapping confirmation per form.

### Step F5 — Generate handlers

For each form, I write the handler in `server/forms/<form-name>.js` (or appropriate path) using the four-layer model. The handler is generated from the manifest's discovered field schema, the user's confirmed routing, and the live-researched ESP/CRM API shape.

I import all generated handlers into `server.js` at the form-endpoints section (Stage 3 left it stubbed).

### Step F6 — Verification

I run an end-to-end test per form (autonomous via Bash + Preview MCP):
- POST a synthetic submission to each `/api/<form-name>` endpoint
- Confirm row appeared in `form_submissions`
- Confirm notification email landed in the user's inbox (the user confirms this; I can't read the inbox)
- Confirm ESP segment now shows the synthetic subscriber
- Confirm CRM contact / lead was created
- Confirm PostHog event fired with correct `<form_name>_completed` shape
- DELETE the synthetic row + remove the test contact from ESP/CRM after the user confirms everything worked

## The `form_submissions` table schema

Generic enough to handle every form's discovered shape:

```sql
CREATE TABLE IF NOT EXISTS form_submissions (
  id              SERIAL PRIMARY KEY,
  form_name       VARCHAR(64)  NOT NULL,        -- matches forms-manifest.md `name`
  route           VARCHAR(255),                 -- the page the form lived on
  payload         JSONB        NOT NULL,        -- the validated submission data
  user_agent      TEXT,
  ip_hash         CHAR(64),                     -- hashed; never raw IP (privacy)
  utm_source      VARCHAR(64),
  utm_medium      VARCHAR(64),
  utm_campaign    VARCHAR(64),
  affiliate_code  VARCHAR(64),                  -- joins with Stage 10's attribution
  posthog_distinct_id VARCHAR(64),
  esp_synced_at   TIMESTAMP,                    -- null if ESP sync skipped/failed
  crm_synced_at   TIMESTAMP,                    -- null if CRM sync skipped/failed
  created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_form_submissions_form_name ON form_submissions (form_name);
CREATE INDEX idx_form_submissions_created_at ON form_submissions (created_at DESC);
```

Querying example: "show me all newsletter signups from the last 30 days":

```sql
SELECT id, payload->>'email' AS email, utm_source, created_at
FROM form_submissions
WHERE form_name = 'newsletter_signup'
  AND created_at >= NOW() - INTERVAL '30 days'
ORDER BY created_at DESC;
```

## The form-handler factory pattern

Generated handlers all share this shape (the per-form parts get filled in from the manifest + user routing):

```js
// server/forms/<form-name>.js (one file per form)
import { getDb } from "../db.js";              // hoisted from Stage 9
import { sendEmail, EMAIL_NOTIFY_TO } from "../email.js";
import { capturePostHogEvent } from "../posthog-server.js";
import { syncToEsp } from "../integrations/<esp>.js";    // only if ESP configured
import { syncToCrm } from "../integrations/<crm>.js";    // only if CRM configured

const SCHEMA = {
  // populated from forms-manifest.md `fields`
  email:        { type: "email",  required: true,  maxLength: 254 },
  full_name:    { type: "string", required: true,  maxLength: 100 },
  // ... etc.
};

const ROUTING = {
  // populated from skill config `forms.per_form_routing[<form-name>]`
  db_capture: true,
  notify_email: true,
  esp: { segment_id: "abc123", tags: ["lead", "newsletter"] },
  crm: { object: "contact", stage: "subscriber" },
};

export function registerHandler(app) {
  app.post("/api/<form-name>", async (req, res) => {
    // 1. Validate
    const { ok, data, errors } = validate(req.body, SCHEMA);
    if (!ok) return res.status(400).json({ ok: false, errors });

    // 2. Layer A — DB capture (default ON)
    let dbId = null;
    if (ROUTING.db_capture) {
      const db = await getDb();
      if (db) {
        const result = await db.query(
          `INSERT INTO form_submissions (form_name, route, payload, user_agent, ip_hash, ...)
           VALUES (?, ?, ?, ?, ?) RETURNING id`,
          ["<form-name>", "<route>", JSON.stringify(data), req.get("user-agent"), hashIp(req.ip)],
        );
        dbId = result[0]?.id;
      }
    }

    // 3. Layer B — Notification email (default ON)
    if (ROUTING.notify_email && EMAIL_NOTIFY_TO) {
      sendEmail({
        to: EMAIL_NOTIFY_TO,
        subject: `New <form-name> submission: ${data.email || data.full_name || "(anonymous)"}`,
        text: JSON.stringify(data, null, 2),
        replyTo: data.email,
      }).catch(err => console.error("[forms] notify failed:", err));
    }

    // 4. Layer C — ESP sync (only if mapped)
    if (ROUTING.esp) {
      syncToEsp({ ...data, segment_id: ROUTING.esp.segment_id, tags: ROUTING.esp.tags })
        .then(() => markEspSynced(dbId))
        .catch(err => console.error("[forms] ESP sync failed:", err));
    }

    // 5. Layer D — CRM sync (only if mapped)
    if (ROUTING.crm) {
      syncToCrm({ ...data, object: ROUTING.crm.object, stage: ROUTING.crm.stage })
        .then(() => markCrmSynced(dbId))
        .catch(err => console.error("[forms] CRM sync failed:", err));
    }

    // 6. PostHog event
    capturePostHogEvent({
      event: "<form-name>_completed",
      properties: {
        form_route: "<route>",
        utm_source: req.query.utm_source ?? null,
        affiliate_code: req.cookies["<brand-prefix>_affiliate_code"] ?? null,
      },
    });

    return res.json({ ok: true });
  });
}
```

The integration files (`integrations/mailchimp.js`, `integrations/hubspot.js`, etc.) are generated from the live research at Stage 5 — each one wraps that vendor's specific API/MCP at the time of integration.

## Saved-config schema additions (already added in onboarding.md Section C)

```json
"forms": {
  "discovered": [          // populated by Stage 0 from forms-manifest.md
    {
      "name": "newsletter_signup",
      "route": "/",
      "fields": [{"name": "email", "type": "email", "required": true}],
      "current_action": null,
      "inferred_purpose": "newsletter_optin"
    }
  ],
  "esp_provider": "mailchimp",         // null if user has no ESP
  "esp_credentials_pasted": true,
  "crm_provider": null,                 // null if user has no CRM
  "crm_credentials_pasted": false,
  "db_capture_enabled": true,           // default true; opt-out gated on CRM coverage
  "db_capture_table": "form_submissions",
  "per_form_routing": {
    "newsletter_signup": {
      "db_capture": true,
      "notify_email": true,
      "esp_segment_id": "abc123",
      "esp_tags": ["newsletter"],
      "crm": null
    },
    "request_demo": {
      "db_capture": true,
      "notify_email": true,
      "esp_segment_id": "prospects",
      "esp_tags": ["lead", "demo-request"],
      "crm": { "object": "contact", "stage": "demo-requested" }
    }
  }
}
```

## Cross-references

- **[stage-0-discovery.md](../stage-0-discovery.md)** — produces `forms-manifest.md` that this stage consumes. The manifest's structure is described in Stage 0's "Forms — full structural extraction" section.
- **[stage-3-express-server.md](../stage-3-express-server.md)** — leaves the form-endpoint section stubbed; this stage populates it.
- **[stage-5-email-resend.md](../stage-5-email-resend.md)** — orchestration entry point; reads this reference and walks the user through Steps F1–F6.
- **[stage-9-membership-optional.md](../stage-9-membership-optional.md)** — `getDb()` helper originated here; if Stage 9 is enabled the DB is already provisioned and Stage 5 reuses it.
- **[stage-11-variants-and-dashboards.md](../stage-11-variants-and-dashboards.md)** — every form's `<form_name>_completed` event auto-extends the variant dashboards' conversion funnels.

## Self-research instruction

Re-verify ESP / CRM landscape every 60 days:
- WebSearch `<vendor> MCP server <current year>` for each ESP/CRM the skill commonly encounters
- WebSearch `<vendor> API deprecation <current year>` to catch breaking changes early
- Update the vendor list at the top of this doc when new MCPs ship or major vendors deprecate APIs

Append findings to this file's Changelog.

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. Captures the four-layer model (validate → DB → notify-email → ESP → CRM → PostHog event), the default-DB-capture rule with CRM-coverage opt-out gate, and the research-first pattern for vendor integrations. Audio / image / video forms are handled via the same pattern (the `payload` JSONB column accommodates any field shape; large file uploads route to object storage per `_internal/reference-object-storage.md` instead of being stored in `form_submissions`). |
