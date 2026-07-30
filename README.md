# wisper-local

One-command local **wisper manager stack** — `wisper.ps1` stands up PostgreSQL,
wisper-api, wisper-web, and wisper-admin in this folder, and tears them all down
again. No Windows services, no scheduled tasks, no firewall rules, no admin
prompt; everything binds `127.0.0.1` and everything on disk lives right here.

Host-side components live in the companion **`host.ps1`** (see "The host stack"
below) — `wisper.ps1` itself is only the manager/marketplace side.

## Quick start

```powershell
# first time ever (installs everything, then starts):
.\wisper.ps1 -Init <branch> -Token ghp_xxxxxxxx

# every day after that:
.\wisper.ps1            # start
.\wisper.ps1 -Down      # stop
```

The first run clones the three repos on `<branch>`, builds them, downloads
portable PostgreSQL binaries (~350 MB, one time), creates the cluster + the
`wisper` role/database, and starts the stack. Expect it to take a while; later
starts take seconds.

When the stack is up, sign in to wisper-web / wisper-admin by pasting the API
key the script prints (also in `state\stack-state.json` under `wckKey`).

## Commands

| Command | What it does |
|---|---|
| `.\wisper.ps1` | Start the stack. Errors with instructions if never installed; tells you if it is already running. |
| `.\wisper.ps1 -Init <branch> -Token ghp_xxx` | Cold start: install + build everything from `<branch>`. On an already-installed stack, `-Init <branch>` just behaves like `-Refetch <branch>`. |
| `.\wisper.ps1 -Refetch <branch>` | Stop the stack, `fetch` + `checkout` + fast-forward pull `<branch>` in all three repos, rebuild, restart. Database migrations (DbUp) apply automatically when wisper-api boots. |
| `.\wisper.ps1 -Down` | Stop all services and postgres. Kills full process trees; nothing is left running. |
| `.\wisper.ps1 -Status` | Per-service running/health overview + the API key. |
| `.\wisper.ps1 -Token ghp_xxx` | Store or rotate the GitHub PAT (then continues with a normal start). |

Port overrides exist as parameters (`-PgPort -ApiPort -WebPort -AdminPort`) but
the postgres port is pinned at install time — see notes below.

## Ports (all loopback-only)

| Service | Address |
|---|---|
| postgres | `127.0.0.1:3005` (db/user `wisper`) |
| wisper-api | `http://127.0.0.1:3006` (health: `/healthz`) |
| wisper-web | `http://localhost:3007` |
| wisper-admin | `http://localhost:3008` |

## The GitHub PAT

- Needed **once**, at `-Init` (the repos are private). Also accepted from
  `$env:GITHUB_PAT` if set when no PAT is stored yet.
- Stored **DPAPI-encrypted** in `state\stack-state.json` (`patDpapi`) — only
  your Windows account on this machine can decrypt it. `-Refetch` reuses it
  silently; plain starts never need git at all.
- It is passed to git as a per-invocation header, so it never appears in
  `.git\config`, remote URLs, or the Windows credential manager.
- Rotate it any time with `.\wisper.ps1 -Token ghp_newtoken`.

## Folder layout

```
wisper.ps1        the script (the only thing you ever run)
README.md         this file
repos\            clones of wisper-api, wisper-web, wisper-admin
pgsql\            portable PostgreSQL binaries (EDB zip, extracted)
pgdata\           the database cluster (your data lives here)
downloads\        the cached PostgreSQL zip
logs\             <service>.out.log / .err.log per service + postgres.log
state\            stack-state.json (secrets/config) + pids.json (while running)
```

**Uninstall** = `.\wisper.ps1 -Down`, then delete this folder. Nothing exists
outside it except generic toolchain caches (npm, NuGet) shared with all your
other projects.

## Notes & gotchas

- **Branch choice matters.** Active development is on `grunt`; the GitHub
  default branches lag far behind (old `main` wisper-api predates API-key auth
  entirely, so the printed key would not work). You can init/refetch any
  branch — just know what's on it. The current branch is stored in state and
  shown in the startup summary.
- **`-Refetch` refuses to lose work**: it pulls `--ff-only` and stops with an
  error if a repo in `repos\` has diverged or has local edits. Commit/stash or
  reset inside `repos\<name>` and re-run.
- **Postgres port is fixed after install.** initdb bakes the port into
  `pgdata\postgresql.conf`, so the stored value always wins and a conflicting
  `-PgPort` only earns a warning. To actually move it, edit
  `pgdata\postgresql.conf` *and* `pgPort` in `state\stack-state.json`.
- **Ports 3005-3008 are common dev-server territory.** If some stray dev
  server grabs one first, the matching health gate times out with a clear
  error — nothing silent.
- **wisper-api runs in `Development`** (required for API-key auth) with dev
  endpoints enabled — fine because nothing is reachable off this machine. Do
  not rebind it to `0.0.0.0` without turning `Tunnel__EnableDevEndpoints` off.
- **Startup order & gates**: postgres → wisper-api (waits on `/healthz`, up to
  120 s — first boot after a refetch runs migrations) → web → admin (up to
  90 s each; Next compiles on first hit).
- **If a start fails partway**, `state\pids.json` is saved after every launch,
  so `.\wisper.ps1 -Down` always cleans up whatever did start.
- **After a reboot** nothing auto-starts (by design). Stale pids are detected
  by process name, so a recycled PID won't confuse `-Status` or `-Down`.

## Troubleshooting

1. `.\wisper.ps1 -Status` — which piece is unhappy?
2. `logs\<service>.err.log` / `logs\<service>.out.log` — the actual error.
3. `logs\postgres.log` — cluster-side problems.
4. Wedged half-started stack: `.\wisper.ps1 -Down` then start again.
5. Nuclear option for one repo: delete `repos\<name>` and run
   `.\wisper.ps1 -Init <branch>` — only the missing repo is re-cloned.

## The host stack (host.ps1)

`host.ps1` brings up the host side on this same machine — **wisp** (the broker,
`http://127.0.0.1:3009`) and **wisp-agent** (tunnels into the local wisper-api;
control panel at `http://localhost:4600`). It shares this folder: same
`repos\` (adds wisp + wisp-agent), same state file (reuses the stored PAT and
the wck key), but its own pid file, so the two scripts never interfere.

```powershell
# first time (manager must have been -Init'd already; Docker should be running):
.\host.ps1 -Init grunt

# daily:
.\host.ps1              # start wisp + agent (requires the manager to be up)
.\host.ps1 -Down        # stop host pieces only; manager keeps running
.\host.ps1 -Status      # includes whether the tunnel shows online
.\host.ps1 -Refetch <branch> [-AgentBranch <b>]
```

Notes specific to the host stack:

- **Single-branch `grunt` works fine**: wisp-agent's `grunt` fully contains the
  secure-lease-isolation work (verified 2026-07-29). The optional
  `-AgentBranch wisp-agent-ui-grunt` only adds the agent's embedded browser
  control panel (port 4600) + two small fixes the script doesn't need. If
  omitted, the agent uses the same branch as `-Init`/`-Refetch`.
- **Docker must be running** to lease (and at `-Init` time to build the base
  image). wisp never pulls images: `-Init` builds `wisp-base` (Linux daemon
  mode) or `wisp-base-windows` (Windows daemon mode) from `repos\wisp\examples\`.
  If you switch daemon modes later, build the other image and update the priced
  list in wisper-web Host tools.
- **First start auto-bootstraps**: registers host `local-host` against the
  manager (its one-time `wht_` agent token is captured into the state file),
  waits for the tunnel to come online, then publishes the advertised images.
- **What the host advertises is yours to edit**: the `hostImages` array in
  `state\stack-state.json` (seeded once with a zero-priced default matching
  your Docker daemon mode, so leases cost $0 with no Stripe). Each entry is
  `{ image_ref, price_cents_per_min, networks, max_ttl_seconds, enabled }`.
  Edit the array and re-run `.\host.ps1` — a changed list is re-published to
  the manager automatically on start; an unchanged list is left alone (so
  tweaks made via wisper-web Host tools survive restarts). Every `image_ref`
  must also be in `config\wisp.config.json` `images.allow` **and** exist in
  the local Docker daemon — wisp never pulls images.
- **If you ever wipe `pgdata\`** (fresh manager DB), the stored host
  registration is orphaned — delete the `host` and `hostImagesPublished`
  entries from `state\stack-state.json` and the next `.\host.ps1` re-registers
  and re-publishes.
- wisp's isolation allow-list is `config\wisp.config.json` (`limits.isolations`,
  default `["shared"]`).

## The orchestrator (orchestrator.ps1)

`orchestrator.ps1` runs the third piece: the orchestrator **API**
(`http://127.0.0.1:3010`, SQLite in `state\orchestrator.sqlite`, migrations at
boot) and its **web UI** (`http://localhost:4400`). Same conventions: shared
`repos\`/`state\`/`logs\`, own pid file, `-Init`/`-Refetch`/`-Down`/`-Status`.

```powershell
.\orchestrator.ps1 -Init grunt     # first time (uses the stored PAT)
.\orchestrator.ps1                 # start   |   -Down stop   |   -Status health
```

It comes up wired to this stack's wisper automatically:
- `WISPER_MODE=v1` against `http://127.0.0.1:3006` as host selector
  `local-host` (the host `host.ps1` registers).
- On every start the stack's `wck_live_` key is seeded into the orchestrator
  secret store as `WISPER_API_KEY` (via `PUT /api/secrets`; never logged).
- The orchestrator boots fine with wisper down (you just can't dispatch);
  the script warns instead of refusing.

Port notes: the orchestrator's native port is **3007**, which this stack gives
to wisper-web — so the API runs on **3010** here. Its web UI's `/api` proxy
target is hardcoded upstream, so the script generates an **untracked**
`web\vite.config.local.ts` (UI port 4400 → API 3010) on every install/refetch;
being untracked, it never blocks the `--ff-only` pull.

Data caveat: the orchestrator keeps its **encrypted secret store and dispatch
logs in `%APPDATA%\orchestrator`** (master key in the OS keychain) — that's its
own design and lives outside this folder. The SQLite DB, however, is kept here
via `ORCH_DB_PATH`.

## Full-stack bring-up order

```powershell
.\wisper.ps1          # 1. manager: postgres + wisper-api + web + admin
.\host.ps1            # 2. host: wisp + wisp-agent (needs manager up)
.\orchestrator.ps1    # 3. consumer: orchestrator API + UI
```

Take them down in any order; each `-Down` touches only its own piece.
# wisper-orchestrator-local
