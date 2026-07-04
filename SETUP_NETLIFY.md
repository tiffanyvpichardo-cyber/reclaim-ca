# Reclaim CA — JotForm + DocuSign wiring (items 5 & 6)

The dashboard is a client-side app (localStorage), so a JotForm webhook can't
write to it directly. These two Netlify Functions bridge the gap:

- `jotform-webhook.js` — receives an intake submission, creates a lead, and
  fires the DocuSign contingency agreement. Queues the lead in Netlify Blobs.
- `pending-leads.js` — the dashboard's **Sync Intake** button reads new leads
  from here and clears them.

Flow: `JotForm submit → webhook → (lead queued + DocuSign sent) → you click "Sync Intake" → lead appears in the pipeline at "Agreement Sent".`

## 1. Install dependencies (repo root)

```bash
npm i @netlify/blobs docusign-esign
```

Netlify Blobs needs no setup on Netlify — it's available to functions on any
deploy. Locally, use `netlify dev` so Blobs + functions are emulated.

## 2. Deploy, then point JotForm at the webhook

In JotForm: **Settings → Integrations → WebHooks**, add:

```
https://<your-site>.netlify.app/.netlify/functions/jotform-webhook
```

Submit the form once. Check the function log in Netlify, then open the
dashboard and click **Sync Intake** — your test submission should appear.

### Field mapping
`jotform-webhook.js` matches JotForm answer keys by token (e.g. any key
containing "email"). Look at one real submission's `rawRequest` in the function
log and, if a field didn't map, add its token to `FIELD_TOKENS` at the top of
the file. Nothing needs exact keys unless auto-matching misses.

## 3. DocuSign (optional but that's item 6)

DocuSign is **fail-safe**: with no config, leads still get created — they just
land in "Contacted" instead of "Agreement Sent". To turn it on, set these
environment variables in Netlify (**Site settings → Environment variables**):

| Variable | What it is |
|---|---|
| `DOCUSIGN_INTEGRATION_KEY` | Your app's integration (client) key |
| `DOCUSIGN_USER_ID` | The API user's GUID (the impersonated user) |
| `DOCUSIGN_ACCOUNT_ID` | Your DocuSign account GUID |
| `DOCUSIGN_PRIVATE_KEY` | RSA private key (PEM). Paste with `\n` for newlines, or the raw multiline value |
| `DOCUSIGN_TEMPLATE_ID` | The contingency-agreement template's ID |
| `DOCUSIGN_TEMPLATE_ROLE` | Role name on the template (default `Signer`) |
| `DOCUSIGN_BASE_PATH` | `https://demo.docusign.net/restapi` (demo) or your prod base path |
| `DOCUSIGN_OAUTH_HOST` | `account-d.docusign.com` (demo) or `account.docusign.com` (prod) |

Auth uses **JWT grant**, so one-time: grant consent for the integration key +
user (DocuSign Admin → the consent URL for your app), otherwise the first token
request returns `consent_required`.

The template should have one role (default "Signer"); the function fills that
role with the client's name + email from the intake form.

## 4. Compliance gate reminder

`agreement.py` / `mailer.py` in the Python pipeline hard-block real mail until
`CA_COMPLIANCE_OK=true`. The DocuSign step here has no such gate — so treat
turning DocuSign on as the same decision: only after your attorney has approved
the agreement template. See `CA_COMPLIANCE.md`.
