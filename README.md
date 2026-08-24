# Field Passport — demo site (static)

Self-contained static site: farmer demo (8 farm packs, `index.html`),
live-draw page against the v2 API (`live/`), API doc (`API_v2.md`),
proposal one-pager (`field_passport_onepager.*`). ~9 MB, no build step,
all internal paths relative — serves from a domain root or a subpath.

## Deploy on GitHub Pages (rayx.co.uk already runs on Pages)

**Option A — subdomain (recommended)**
1. Push this directory to a new repo in the RayX org (e.g. `field-passport-demo`).
2. Repo Settings -> Pages -> deploy from branch, root.
3. Keep the `CNAME` file (`passport.rayx.co.uk`); set the same custom domain in
   the Pages settings; tick Enforce HTTPS once the cert issues.
4. Namecheap DNS: add `CNAME passport -> <org>.github.io`.

**Option B — path on the main site**
Copy everything into a `demo/` folder of the existing site repo and DELETE
the `CNAME` file. No DNS change; served at rayx.co.uk/demo/.

## After it's live
- The live page calls the AWS API (api gateway, eu-west-2); CORS already
  allows any origin. Optionally tighten `AllowOrigins` to the new domain.
- Rotate the API token and share the new one with demo users only
  (the page takes it via one-time prompt or a `#token=` URL fragment —
  it is never embedded in the page).
- The JASMIN copy can stay as staging; site updates = re-run
  `publish_farm.py` and push.

## Before going public on a company domain — check one licence
`crop_history` in the field packs derives from UKCEH LCCM. Publishing it
under a company domain is a redistribution question — confirm the licence
covers it, or republish the packs with crop_history stripped.
