# orca_mcp_guides — Deployment notes

**This repo = "Orca Query Demo" / "Ask Orca" demo (Tom's marketing six-question dashboard).**
Single self-contained `index.html` (Chart.js + the example queries + the dashboard).
It is a *static page* that calls the Orca `/call` HTTP API at runtime — no server-side build,
no secrets in the repo.

---

## 1. Current deploy reality (verified 2026-06-14)

| Fact | Status |
|---|---|
| Git remote | **GitHub only** — `github.com/urbancanary/orca_mcp_guides`. No Gitea remote. |
| On the Hetzner/Coolify box? | **NO.** Not in the `guinness` Gitea org's 11 repos; no `~/Notebooks/hetzner/orca_mcp_guides` working copy. |
| Railway? | Register lists it as **TBD** (project + domain unconfirmed). Treat as *not confirmed deployed anywhere* until a live URL responds. |
| `tom.guinnessgi.x-trillion.com` | DNS resolves to the box (178.105.199.104) but **returns an error page** — nothing is serving this app there. |
| `tom.x-trillion.com` | **Does not resolve** (no DNS record). |

**Consequence:** `git push origin main` publishes to GitHub but **does NOT reach the Hetzner box**.
There is currently no automated path from this repo to any live host. If you "deployed" by pushing,
the code is on GitHub and nowhere else.

---

## 2. How apps actually get onto the Hetzner box (the model this repo is NOT yet wired into)

The box does **not** deploy from GitHub. Authoritative source of truth:
**`mcp_central/guinness-platform/PLATFORM_MODEL.md`** and **`GUINNESS_DELIVERY_RUNBOOK.md`**.
Summary of the pipeline used by the 11 live box apps (athena-html-v3, iris, lexa-mcp, jess-video,
maia-mcp, iona-mcp, georgia, guinness-mcp, presentation-mcp, brian-mcp, orion_v3):

```
GitHub  urbancanary/<app>  (the development copy, what you edit in mcp_central/<app>)
   │
   │  cron on the box: /root/sync_guinness_upstream.sh  (every ~30 min)
   │  pushes GitHub main → Gitea branch `upstream`  (never touches `guinness`)
   ▼
Gitea (ON the box)  gitea.guinness.x-trillion.com  →  org `guinness`  →  repo <app>
   branches (NO `main`, by design):
     upstream  = mirror of GitHub main (auto-synced)
     active    = what WE curate to ship (vendor decision)
     guinness  = what the client has signed off  ← Coolify builds THIS branch
     original  = frozen sanitised first-delivery snapshot
   │
   │  Coolify (http://178.105.199.104:8000) builds the `guinness` branch on push
   ▼
Live at  <app>.guinness.x-trillion.com   (Traefik routes by host, Let's Encrypt TLS)
```

- The local box working copies live at **`~/Notebooks/hetzner/<app>`** (origin = Gitea, NOT GitHub).
  Edits there go on `active`, push to Gitea → Coolify redeploys. See `~/Notebooks/hetzner/README.BOX_TREES.md`.
- For a confidentiality-gated *client delivery* (scrub history, leak-gate, fresh storefront repo),
  follow the full `GUINNESS_DELIVERY_RUNBOOK.md`. That is heavier than a plain box deploy and is only
  needed when Tom/Will get read access to the Gitea repo.

---

## 3. To put THIS app on the box (if that's the intent)

Decide first **which front door** it should use, then provision. Minimum steps (mirrors the
openfigi/iris recipe in the runbook + the Coolify deploy memory):

1. **Pick the host + tenant model.** Likely `ask.guinness.x-trillion.com` (there is already an
   `ask-orca` tile concept in the Orion box shell). NOTE the on-box detection in `index.html`
   (`ON_BOX` regex matches `guinness.x-trillion.com`, **not** `guinness*gi*.x-trillion.com`) — pick
   the host to match, or widen the regex.
2. **Create the Gitea repo** `guinness/orca_mcp_guides` (private) and add a `~/Notebooks/hetzner/orca_mcp_guides`
   working copy tracking it (branch `guinness`).
3. **Wire the sync** (add this repo to `/root/sync_guinness_upstream.sh`) **or** push the curated tree
   to the `guinness` branch manually for a static page this small.
4. **Create a Coolify app** pointing at the Gitea `guinness` branch. This is a **static HTML** app
   (NIXPACKS + `python3 -m http.server`, per the current `Procfile`) — no Dockerfile today; healthcheck
   off (the python-slim no-curl gotcha). Serve on the chosen subdomain.
5. **DNS:** the `*.guinness.x-trillion.com` wildcard already points at the box; just confirm the host
   resolves and Traefik issues a cert.
6. **Runtime API:** the page calls Orca over the gateway. On-box it should use `orca.x-trillion.com`
   (authenticated, sends the `xt_session` cookie); standalone/dev it falls back to the public
   `orca-mcp.x-trillion.com` (header-auth, `credentials:'omit'`). Both paths already exist in `index.html`.

### Confidentiality gotcha if this is ever client-delivered
`index.html` references **`orca-mcp.x-trillion.com`**, which is on the runbook **denylist** (§3.1 —
the internal host, distinct from the allowed `orca.x-trillion.com`). A plain box deploy for internal
use is fine, but a *client-facing* delivery via the storefront pipeline must scrub that host first or
`leak_gate.sh` will (correctly) fail.

---

## 4. Local dev

```bash
cd orca_mcp_guides && python3 -m http.server 8777
# open http://localhost:8777/index.html        → public gateway path (orca-mcp.x-trillion.com, creds omit)
# open http://localhost:8777/index.html?orion=true → on-box path (orca.x-trillion.com, needs a session → 401 at localhost)
```
Append a cache-buster (`?cb=123`) when testing edits — the page is aggressively cached.

---

*Last verified 2026-06-14. If you change where/how this app deploys, update this file in the same commit.*
