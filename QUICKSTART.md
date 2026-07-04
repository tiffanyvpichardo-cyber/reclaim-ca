# Reclaim CA — Quickstart (both halves)

You have two separate projects that hand off to each other:

```
  PYTHON PIPELINE                          DASHBOARD (Vite + Netlify)
  pulls CA SCO data, skip-traces,   →      CRM: pipeline, financials, letters,
  mails letters, exports JSON              AI notes. Reads leads 3 ways:
                                             1. Import JSON  (from the pipeline)
                                             2. Sync Intake  (from JotForm)
                                             3. + New Lead   (manual)
```

They connect through one file: the pipeline writes
`output/exports/reclaim_ca_leads_export.json`, and you load it in the dashboard
with the **Import JSON** button.

---

## Part A — Dashboard (Vite site on Netlify)

The `reclaim_ca_dashboard/` folder is a complete Vite project.

**If you already have your Vite repo:** copy `src/reclaim_ca_app.jsx` into your
`src/`, make sure `src/main.jsx` imports it (`import App from "./reclaim_ca_app.jsx"`),
and copy the `netlify/` folder + `netlify.toml` to the repo root. Then add the
two function deps (below).

**If you want to use this folder as the project:**

```bash
cd reclaim_ca_dashboard
npm install
npm run dev          # local preview at http://localhost:5173
```

Deploy: push the folder to a Git repo and connect it in Netlify (build command
`npm run build`, publish dir `dist` — already set in `netlify.toml`), or run
`netlify deploy --prod`.

Notes:
- The dashboard stores data in your browser under `reclaim_ca_v1`. It's
  per-browser — it isn't shared between devices.
- On first load it shows 4 sample CA leads. Delete them via the pipeline import
  or just ignore them.

## Part B — Python pipeline

```bash
cd reclaim_ca_pipeline
pip install -r requirements.txt
cp .env.example .env          # then fill in your keys + business info
```

Weekly run (Thursdays, after the SCO refresh):

```bash
python pipeline.py --download --band 500+
python pipeline.py --load-file data/sco/<downloaded-file>.zip
python pipeline.py --skip-trace-only
python pipeline.py --letters-only --dry-run     # preview; real send is gated
python pipeline.py --export
```

Then in the dashboard, click **Import JSON** and pick
`reclaim_ca_pipeline/output/exports/reclaim_ca_leads_export.json`.

## Part C — JotForm + DocuSign (optional automation)

See `reclaim_ca_dashboard/SETUP_NETLIFY.md`. Short version: `npm i @netlify/blobs
docusign-esign` (already in this scaffold's package.json), point your JotForm
webhook at the deployed `jotform-webhook` function, set the DocuSign env vars in
Netlify, then use **Sync Intake** in the dashboard to pull new submissions.

## Part D — Before you go live (please read)

- **Fee cap is enforced** at 10% (CCP § 1582) in both the dashboard and the
  agreement/letter generators — no override.
- **Real mail is blocked** until you set `CA_COMPLIANCE_OK=true` in the
  pipeline `.env`. DocuSign has no such gate, so treat enabling it the same way.
- **Have a California attorney review** the agreement + letter language and
  confirm your licensing and the § 1582 timing rules before sending anything.
  Details in `reclaim_ca_pipeline/CA_COMPLIANCE.md`.

---

## What's NOT included / still on you

- Accounts + API keys: BatchData, PostGrid, Anthropic, JotForm, DocuSign.
- The **DocuSign template** itself — you build the contingency-agreement
  template in DocuSign and put its ID in `DOCUSIGN_TEMPLATE_ID`.
- Attorney review (above).
- **Reclaim GA rename** — still needs the `surplus_recovery_app.jsx` file, which
  hasn't been uploaded. That one's a quick rename once it's provided.
- Exact **brand-color match to Reclaim GA** — the palette here is a clean
  cream/gold/navy set; drop in GA's hex values (top of `reclaim_ca_app.jsx`) to
  match precisely.
