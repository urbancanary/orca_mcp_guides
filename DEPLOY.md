# orca_mcp_guides — Deployment Bible (current ↔ target)

**This is the source of truth for how the "Ask Orca" demo (Tom's six-question dashboard) deploys.**
It is written as a *split view*: the left of each row is **NOW (verified 2026-06-14)**, the right is
**TARGET (what we're committing to)**. When NOW ≠ TARGET, §3 is the checklist that closes the gap.
**Rule: if you change how/where this app deploys, update this file in the same commit so NOW stays true.**

The app itself is one self-contained static `index.html` (Chart.js + example queries + the dashboard)
that calls the Orca `/call` HTTP API at runtime. No server build, no secrets in the repo.

---

## 1. Split view — NOW vs TARGET

| Dimension | NOW (verified 2026-06-14) | TARGET (the bible) |
|---|---|---|
| **Git remote(s)** | ✅ GitHub `urbancanary/orca_mcp_guides` **+** Gitea `guinness/orca_mcp_guides` (created + pushed 2026-06-14) | GitHub `urbancanary/orca_mcp_guides` (dev/vault) **+** Gitea `guinness/orca_mcp_guides` on the box |
| **Box working copy** | ✅ `~/Notebooks/hetzner/orca_mcp_guides` (seeded from GitHub `5faa1f2`), **pushed to Gitea** | `~/Notebooks/hetzner/orca_mcp_guides`, origin = Gitea, edits on branch `active` |
| **Branch model** | ✅ Gitea has `upstream`/`active`/`guinness`/`original` (no `main`); **`guinness` is the default branch** (Coolify build target) | Box repo: `upstream` / `active` / `guinness` / `original` — **no `main`** (by design); Coolify builds **`guinness`** |
| **Live host** | ✅ `ask.guinness.x-trillion.com` — HTTP 200, real Let's Encrypt cert (deployed 2026-06-15) | `ask.guinness.x-trillion.com` (Traefik host-routed, Let's Encrypt) |
| **Deploy trigger** | push to Gitea `guinness` then **manual Coolify Redeploy** (webhook not yet wired — see #1690) | push to Gitea `guinness` → **Coolify auto-builds + serves** |
| **Build** | ✅ Coolify app `ask-orca` (uuid `d7kq3ygsw6yp38e1e8hptndu`), NIXPACKS + `python3 -m http.server`, `PORT=8080`, healthcheck **off** | Coolify **static app**, NIXPACKS + `python3 -m http.server` (current `Procfile`); healthcheck **off** (python-slim no-curl gotcha) |
| **Runtime gateway** | works standalone via public `orca-mcp.x-trillion.com` (`credentials:'omit'`) | on-box uses `orca.x-trillion.com` (authenticated, sends `xt_session` cookie); standalone falls back to public — **both already coded in `index.html`** |
| **On-box auth** | none (public header-auth path) | `xt_session` cookie via the Orca gateway (same SSO as the other box apps) |
| **In the confidentiality pipeline?** | no | optional — only if Tom/Will get **Gitea read access**; then full `GUINNESS_DELIVERY_RUNBOOK.md` (scrub + leak-gate + storefront) applies |
| **Status** | ✅ **LIVE** at `ask.guinness.x-trillion.com` (200, real cert; `/?orion=true` iframe path 200) | live + discoverable from the Orion box shell ("Ask Orca" tile) |

**The one-line truth:** `git push origin main` here publishes to GitHub and reaches **no live host**.
The box does **not** deploy from GitHub — it deploys from **Gitea on the box**, which this repo is not yet wired into.

---

## 2. TARGET architecture (how it *should* flow)

This mirrors the 11 live box apps (athena-html-v3, iris, lexa-mcp, jess-video, maia-mcp, iona-mcp,
georgia, guinness-mcp, presentation-mcp, brian-mcp, orion_v3). Canonical refs:
`mcp_central/guinness-platform/PLATFORM_MODEL.md` + `GUINNESS_DELIVERY_RUNBOOK.md` +
`~/Notebooks/hetzner/README.BOX_TREES.md`.

```
GitHub  urbancanary/orca_mcp_guides         ← dev copy (mcp_central/orca_mcp_guides, you edit here)
   │
   │  cron on box: /root/sync_guinness_upstream.sh (~30 min)
   │  GitHub main → Gitea branch `upstream`  (never touches `guinness`)
   ▼
Gitea ON THE BOX  gitea.guinness.x-trillion.com → org `guinness` → repo orca_mcp_guides
   upstream = mirror of GitHub main
   active   = what we curate to ship           ← box edits land here (~/Notebooks/hetzner/orca_mcp_guides)
   guinness = client-signed-off, DEFAULT        ← Coolify builds THIS
   original = frozen first-delivery snapshot
   │
   │  Coolify (178.105.199.104:8000) builds `guinness` on push
   ▼
Live   ask.guinness.x-trillion.com   (Traefik routes by host, Let's Encrypt TLS)
   │
   ▼
Orion box shell  →  "Ask Orca" tile iframes  ask.guinness.x-trillion.com?orion=true
```

---

## 3. Gap-closing checklist (NOW → TARGET) — tick these to deploy

> One-time provisioning. Follow in order; each is a USER (Coolify/Gitea UI or box SSH) or AGENT step.
> Detailed recipe + API shapes: `GUINNESS_DELIVERY_RUNBOOK.md` §4–5 and the Coolify deploy memory.

- [ ] **Decide the host.** Confirm `ask.guinness.x-trillion.com` (the `ask-orca` tile concept already
      exists in the Orion box shell). ⚠ `index.html` `ON_BOX` regex matches `guinness.x-trillion.com`
      but **NOT** `guinness*gi*.x-trillion.com` — pick a `*.guinness.x-trillion.com` host **or** widen the regex.
- [x] **Add box working copy** `~/Notebooks/hetzner/orca_mcp_guides` — done 2026-06-14, seeded from
      GitHub `5faa1f2`; branches `upstream`/`active`/`guinness`/`original` (no `main`) all at seed
      `2f8f463`, HEAD on `active`, origin → `gitea.guinness.x-trillion.com/guinness/orca_mcp_guides.git`
      (not yet pushed). Matches the existing box trees (cf. iris, lexa-mcp).
- [x] **Create Gitea repo** `guinness/orca_mcp_guides` (private) + **push all 4 branches** — done
      2026-06-14 (repo id 20, HTTP 201; branches active/guinness/original/upstream pushed; `guinness`
      set as default branch). Done via on-box token (never left the box).
- [ ] **Wire sync** — add this repo to `/root/sync_guinness_upstream.sh` (GitHub→Gitea `upstream`),
      **or** for a page this small just push the curated tree to `guinness` by hand.
- [x] **Create the Coolify app** — done 2026-06-15. App `ask-orca` uuid `d7kq3ygsw6yp38e1e8hptndu`,
      source = Gitea `guinness/orca_mcp_guides` branch `guinness`, NIXPACKS `python3 -m http.server`,
      `PORT=8080`, healthcheck off, FQDN `ask.guinness.x-trillion.com`. Created via on-box Coolify API.
- [x] **DNS/TLS** — confirmed: `ask.guinness.x-trillion.com` → HTTP 200, real Let's Encrypt cert.
- [ ] **Smoke test** in a browser with the box session: page loads (verified 200 + `/?orion=true` 200);
      still TODO — confirm panels actually fill (data calls need a valid `xt_session`; under the no-login
      per-user subdomain Tom has no session, so live Orca calls may 401 — ties to the deferred real-auth work).
- [ ] **Wire the Orion tile** ("Ask Orca" sidebar entry → `ask.guinness.x-trillion.com?orion=true`).
- [ ] **Update §1 NOW column + the CLAUDE.md deploy register row** for orca_mcp_guides.

---

## 4. Invariants — the rules we follow (don't relitigate these)

1. **The box deploys from Gitea, never GitHub.** Pushing to GitHub is necessary (dev/vault) but never sufficient.
2. **No `main` on the box copy** — the `upstream/active/guinness/original` names are a forcing function so
   muscle-memory `git checkout/push main` can't push the wrong tree to the wrong place.
3. **Coolify builds the `guinness` branch.** `active` is staging; `guinness` is live.
4. **Edit the right copy:** app dev → `mcp_central/orca_mcp_guides` (GitHub); box changes →
   `~/Notebooks/hetzner/orca_mcp_guides` (Gitea). Never cross them.
5. **Runtime gateway is host-driven, already coded:** on-box → `orca.x-trillion.com` (+cookie);
   standalone → `orca-mcp.x-trillion.com` (`credentials:'omit'`). Don't hardcode a single base.
6. **Confidentiality (only if Tom/Will get Gitea read access):** `index.html` references
   `orca-mcp.x-trillion.com`, which is on the runbook **denylist** (§3.1 — the internal host, distinct from
   the allowed `orca.x-trillion.com`). Internal box use is fine; a *client-readable* storefront delivery
   must scrub it first or `leak_gate.sh` will (correctly) fail. Never hand Tom internal Gitea/Railway/Worker URLs.

---

## 5. Local dev

```bash
cd orca_mcp_guides && python3 -m http.server 8777
# http://localhost:8777/index.html            → public gateway (orca-mcp.x-trillion.com, creds omit)
# http://localhost:8777/index.html?orion=true → on-box path (orca.x-trillion.com, 401 at localhost — no session)
```
Append `?cb=<n>` when testing edits — the page is aggressively cached.

---

*Bible last verified 2026-06-14. NOW column is only trustworthy if kept in lockstep with reality —
update it in the same commit as any deploy change.*
