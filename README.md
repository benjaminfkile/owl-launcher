# wisper-local

One-command local **wisper manager stack**: `wisper.ps1` stands up PostgreSQL,
wisper-api, wisper-web, and wisper-admin in this folder, and tears them all down
again. No Windows services, no scheduled tasks, no firewall rules, no admin
prompt; everything binds `127.0.0.1` and everything on disk lives right here.

Host-side components live in the companion **`host.ps1`** (see "The host stack"
below) and the consumer lives in **`orchestrator.ps1`**; `wisper.ps1` itself is
only the manager/marketplace side.

## Quick start

```powershell
# first time ever (installs everything, then starts):
.\wisper.ps1 -Init <branch>

# every day after that:
.\wisper.ps1            # start
.\wisper.ps1 -Down      # stop
```

The first run clones the three repos on `<branch>`, builds them (`dotnet build
-c Release src\Wisper.Api`, `npm ci` in the two Next apps), downloads portable
PostgreSQL binaries (~350 MB, one time), creates the cluster + the `wisper`
role/database, and starts the stack. Expect it to take a while; later starts
take seconds. Toolchain needs: git, node, npm, .NET 8 SDK (host adds go and
Docker).

When the stack is up, sign in to wisper-web / wisper-admin by pasting the API
key the script prints (also in `state\stack-state.json` under `wckKey`). The key
is a config allow-list entry (`Auth__ApiKeys__<key>__*`) with the `consumer`,
`host` and `admin` scopes, so one key drives every UI and the orchestrator.

## Commands

| Command | What it does |
|---|---|
| `.\wisper.ps1` | Start the stack. Errors with instructions if never installed; tells you if it is already running. |
| `.\wisper.ps1 -Init <branch>` | Cold start: install + build everything from `<branch>`. On an already-installed stack, `-Init <branch>` just behaves like `-Refetch <branch>`. |
| `.\wisper.ps1 -Refetch <branch>` | Stop the stack, `fetch` + `checkout` + fast-forward pull `<branch>` in all three repos, rebuild, restart. Database migrations (DbUp) apply automatically when wisper-api boots. |
| `.\wisper.ps1 -Down` | Stop all services and postgres. Kills full process trees; nothing is left running. |
| `.\wisper.ps1 -Status` | Per-service running/health overview + the API key. |

Port overrides exist as parameters (`-PgPort -ApiPort -WebPort -AdminPort`) but
the postgres port is pinned at install time (see notes below). `-PgVersion`
picks the EDB zip (default `17.5-3`). `-RelayTimeoutMs` (default 3000000 = 50
min) is wisper-api's `Tunnel__RelayRequestTimeoutMs`, the deadline for one
relayed host request including the synchronous lease create; the API's own
default is 120 s, far too short for images that provision for minutes.

## Ports (all loopback-only)

| Service | Address |
|---|---|
| postgres | `127.0.0.1:3005` (db/user `wisper`) |
| wisper-api | `http://127.0.0.1:3006` (health: `/healthz`, also `/api/health`; agent tunnel `ws://127.0.0.1:3006/agent`) |
| wisper-web | `http://localhost:3007` |
| wisper-admin | `http://localhost:3008` |
| wisp (host.ps1) | `http://127.0.0.1:3009` (health: `/healthz`) |
| orchestrator API (orchestrator.ps1) | `http://127.0.0.1:3010` (health: `/api/health`) |
| orchestrator web (orchestrator.ps1) | `http://localhost:4400` |

## Folder layout

```
wisper.ps1        the manager script
host.ps1          the host script (wisp + wisp-agent)
orchestrator.ps1  the consumer script (orchestrator API + web UI)
README.md         this file
repos\            clones of wisper-api, wisper-web, wisper-admin
                  (+ wisp, wisp-agent from host.ps1; + orchestrator from orchestrator.ps1)
pgsql\            portable PostgreSQL binaries (EDB zip, extracted)
pgdata\           the database cluster (your data lives here)
downloads\        the cached PostgreSQL zip
config\           wisp.config.json (written once by host.ps1 -Init)
logs\             <service>.out.log / .err.log per service + postgres.log
state\            stack-state.json (secrets/config, shared by all three scripts)
                  pids.json / host-pids.json / orch-pids.json (while running)
                  orchestrator.sqlite (the orchestrator DB)
```

`state\stack-state.json` keys: `branch`, `pgPort`, `pgSuperPassword`,
`pgWisperPassword`, `wckKey`, `wckUserId`, `wckEmail` (manager);
`hostBranch`, `agentBranch`, `wispAppToken`, `host` (`id`, `name`,
`agentToken`), `hostImages`, `hostImagesPublished` (host); `orchBranch`
(orchestrator). Plaintext; the OS user boundary is the security model.

**Uninstall** = `.\wisper.ps1 -Down` (plus `.\host.ps1 -Down` and
`.\orchestrator.ps1 -Down` if you use them), then delete this folder. Nothing
exists outside it except generic toolchain caches (npm, NuGet, Go) shared with
all your other projects, the Docker images host.ps1 built, and the orchestrator's
`%APPDATA%\orchestrator` (see below).

## Notes & gotchas

- **Branch choice matters.** Active development lands on `grunt` first; as of
  2026-08-16 the GitHub `main` branches of all six repos are in sync with
  `grunt`, but `grunt` moves ahead between merges. You can init/refetch any
  branch, just know what's on it. The current branch is stored in state and
  shown in the startup summary.
- **`-Refetch` refuses to lose work**: it pulls `--ff-only` and stops with an
  error if a repo in `repos\` has diverged or has local edits. Commit/stash or
  reset inside `repos\<name>` and re-run.
- **Postgres port is fixed after install.** initdb bakes the port into
  `pgdata\postgresql.conf`, so the stored value always wins and a conflicting
  `-PgPort` only earns a warning. To actually move it, edit
  `pgdata\postgresql.conf` *and* `pgPort` in `state\stack-state.json`.
- **Ports 3005-3010 and 4400 are common dev-server territory.** If some stray
  dev server grabs one first, the matching health gate times out with a clear
  error, nothing silent.
- **wisper-api runs in `Development`** (`ASPNETCORE_ENVIRONMENT`) with
  `Tunnel__EnableDevEndpoints=true`; the dev lease/shell endpoints are gated on
  both. Fine because nothing is reachable off this machine. Do not rebind it to
  `0.0.0.0` without turning `Tunnel__EnableDevEndpoints` off.
- **The dev wallet is funded on every start.** After wisper-api is healthy the
  script runs an idempotent SQL seed (`state\fund-wallet.sql`, deleted
  afterwards) against the `wisper` DB: creates the `wckUserId` user row, sets
  its `connect_status` to `enabled` (so it may advertise priced images with no
  Stripe), layers a 10% `platform_policy` row over the 0% default that wisper-api migration 0017 seeds (skipped once any non-zero or operator-published policy exists), and credits the
  user wallet with $1,000,000,000.00 via one balanced `topup` transaction
  (idempotency key `seed:local-bootstrap-fund`). So priced leases work with no
  Stripe at all; a failed seed only warns and priced leases 402. Change
  `$FundCents` in the script to alter the amount.
- **Startup order & gates**: postgres (pg_isready, 30 s) -> wisper-api (waits
  on `/healthz`, up to 120 s; first boot after a refetch runs migrations) ->
  wallet seed -> web -> admin (up to 90 s each; Next compiles on first hit).
- **If a start fails partway**, `state\pids.json` is saved after every launch,
  so `.\wisper.ps1 -Down` always cleans up whatever did start.
- **After a reboot** nothing auto-starts (by design). Stale pids are detected
  by process name, so a recycled PID won't confuse `-Status` or `-Down`.

## Troubleshooting

1. `.\wisper.ps1 -Status`: which piece is unhappy?
2. `logs\<service>.err.log` / `logs\<service>.out.log`: the actual error.
3. `logs\postgres.log`: cluster-side problems.
4. Wedged half-started stack: `.\wisper.ps1 -Down` then start again.
5. Nuclear option for one repo: delete `repos\<name>` and run
   `.\wisper.ps1 -Init <branch>`; only the missing repo is re-cloned.

## The host stack (host.ps1)

`host.ps1` brings up the host side on this same machine: **wisp** (the broker,
`http://127.0.0.1:3009`, built as `repos\wisp\wispd.exe`) and **wisp-agent**
(`repos\wisp-agent\wisp-agent.exe`, dials `ws://127.0.0.1:3006/agent` and
bridges leases to the local wisp). It shares this folder: same `repos\` (adds
wisp + wisp-agent), same state file (reuses the wck key), but its own pid file
(`state\host-pids.json`), so the two scripts never interfere.

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

- **Use `grunt` for both repos.** `-AgentBranch <b>` overrides the wisp-agent
  branch, but the only alternative branch, `wisp-agent-ui-grunt` (the agent's
  embedded browser control panel on port 4600), stopped at 2026-07-23 and is
  12 commits behind `grunt`: it lacks the capacity/heartbeat capability, the
  resync-from-wisp-on-connect, the truthful lease lifecycle reporting and the
  GPU pass-through that the current wisper-api expects. There is no control
  panel on `grunt`; ignore the `agent panel http://localhost:4600` line in the
  startup summary unless you deliberately run that older branch.
- **What the agent is started with**: `--manager ws://127.0.0.1:3006/agent`,
  `--host-token <wht_ token from state>`, `--wisp http://127.0.0.1:3009`,
  `--wisp-token <wispAppToken>` and `--wisp-create-timeout <-CreateTimeout>`
  (default `50m`, Go duration; the agent's built-in default is 60 s, too short
  for images that provision for minutes). wisp itself gets `WISP_ADDR`,
  `WISP_CONFIG` and `WISP_APP_TOKEN`; the app token now guards `POST/GET
  /contracts` and `POST /events`, and the agent uses it to resync its lease map
  from wisp on every tunnel connect.
- **Docker must be running** to lease (and at `-Init` time to build the base
  image). wisp never pulls images: `-Init` builds `wisp-base` (Linux daemon
  mode) or `wisp-base-windows` (Windows daemon mode) from `repos\wisp\examples\`.
  If you switch daemon modes later, build the other image and update the priced
  list in wisper-web Host tools.
- **First start auto-bootstraps**: registers host `local-host` against the
  manager (`POST /v1/hosts`; its one-time `wht_` agent token is captured into
  the state file), waits up to 60 s for `GET /v1/hosts/mine` to report the host
  `online`, then publishes the advertised images (`PUT /v1/hosts/{id}/images`,
  which 409s `host_offline` until the tunnel is up). Gates before that: wisp
  `/healthz` within 30 s; the manager's `/healthz` must answer or the script
  refuses to start.
- **What the host advertises is yours to edit**: the `hostImages` array in
  `state\stack-state.json`, seeded once with one image matching your Docker
  daemon mode, **priced at 33 cents per minute** (about $19.80/hr, chosen to
  exercise the paid path; the manager script's funded wallet pays for it),
  networks `none`/`open`, `max_ttl_seconds` 3600, enabled. Each entry is
  `{ image_ref, price_cents_per_min, networks, max_ttl_seconds, enabled }`; the
  API also accepts optional `cpus`, `memory_mb`, `gpus`, `max_cpus`,
  `max_memory_mb`, `max_pids`. Set `price_cents_per_min` to 0 for free leases.
  Edit the array and re-run `.\host.ps1`; a changed list is re-published to
  the manager automatically on start, an unchanged list is left alone (so
  tweaks made via wisper-web Host tools survive restarts). Every `image_ref`
  must also be in `config\wisp.config.json` `images.allow` **and** exist in
  the local Docker daemon; wisp never pulls images.
- **`config\wisp.config.json`** is written once at `-Init` and never
  overwritten: `images.allow` = `wisp-base`, `wisp-base-windows`; `limits`
  per-lease caps (`max_cpus`, `max_memory_mb`) and aggregate budget
  (`total_cpus`, `total_memory_mb`) both seeded at half this machine, `0` =
  unlimited elsewhere, `networks` `none`/`open`, and the isolation allow-list
  `limits.isolations` (default `["shared"]`). wisp also understands
  `max_contracts`, `max_gpus`, `gpus_disabled` and `default_isolation` (see
  `repos\wisp\examples\wisp.config.json`); add them by hand if you need them.
  Edit and re-run `.\host.ps1` to apply.
- **If you ever wipe `pgdata\`** (fresh manager DB), the stored host
  registration is orphaned: delete the `host` and `hostImagesPublished`
  entries from `state\stack-state.json` and the next `.\host.ps1` re-registers
  and re-publishes.

## The orchestrator (orchestrator.ps1)

`orchestrator.ps1` runs the third piece: the orchestrator **API**
(`http://127.0.0.1:3010`, health `/api/health`, SQLite in
`state\orchestrator.sqlite`, knex migrations at boot) and its **web UI**
(`http://localhost:4400`, Vite dev server). Same conventions: shared
`repos\`/`state\`/`logs\`, own pid file (`state\orch-pids.json`),
`-Init`/`-Refetch`/`-Down`/`-Status`. Build = `npm ci` at the repo root and in
`web\`, then `npm run build` (the API runs from `dist\index.js` under node).

```powershell
.\orchestrator.ps1 -Init grunt     # first time
.\orchestrator.ps1                 # start   |   -Down stop   |   -Status health
```

It comes up wired to this stack's wisper automatically:
- `WISPER_MODE=v1`, `WISPER_BASE_URL=http://127.0.0.1:3006`, and host selector
  `WISPER_HOST_ID=local-host` (the host `host.ps1` registers; the orchestrator
  matches it against the catalog by id or name).
- `WISPER_CREATE_LEASE_TIMEOUT_MS` (parameter `-CreateLeaseTimeoutMs`, default
  3000000 = 50 min; the built-in default is 150 s) bounds the synchronous
  create-lease call. Keep it under a playbook's `ttl_seconds` minus 60.
- On every start the stack's `wck_live_` key is seeded into the orchestrator
  secret store as `WISPER_API_KEY` (via `PUT /api/secrets` with `{key, value}`;
  never logged). The API is gated on `/api/health` within 60 s first; the UI
  within 90 s after.
- The orchestrator boots fine with wisper down (you just can't dispatch);
  the script warns instead of refusing.

Port notes: the orchestrator's native port is **3007**, which this stack gives
to wisper-web, so the API runs on **3010** here via `PORT`. Its web UI's `/api`
proxy target is hardcoded upstream (`web\vite.config.ts` -> 3007), so the script
generates an **untracked** `web\vite.config.local.ts` (UI port 4400 -> API
3010) on every install/refetch and starts Vite with `--config
vite.config.local.ts`; being untracked, it never blocks the `--ff-only` pull.

Data caveat: the orchestrator keeps its **encrypted secret store and
per-dispatch logs in `%APPDATA%\orchestrator`** (master key in the OS
keychain); that's its own design and lives outside this folder. The SQLite DB,
however, is kept here via `ORCH_DB_PATH`.

## Full-stack bring-up order

```powershell
.\wisper.ps1          # 1. manager: postgres + wisper-api + web + admin
.\host.ps1            # 2. host: wisp + wisp-agent (needs manager up)
.\orchestrator.ps1    # 3. consumer: orchestrator API + UI
```

Take them down in any order; each `-Down` touches only its own piece.
