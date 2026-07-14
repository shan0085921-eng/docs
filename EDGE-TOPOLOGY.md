# Edge & Compose Topology (Fuji dev box `verisphere-dev`)

**Status: documents ACTUAL deployed reality, which diverges from what the compose
files imply.** Written 2026-07-11 from live inspection. If you change the edge or the
compose wiring, update this file in the same commit.

## TL;DR

- The live edge is **host nginx** (systemd, outside Docker), NOT Caddy.
- **Caddy is defined in `docker-compose.prod.yml` but never runs.** Do not start it —
  it binds :80/:443, which host nginx already owns (port clash).
- The app is published on **`127.0.0.1:8070`** (loopback only) so host nginx can
  `proxy_pass` to it. Not reachable from the internet directly.
- The **running containers were created from `docker-compose.yml`** (the dev file),
  even though we drive them with the two prod files. This is drift (see below).
- **Operational rule:** never run an *unscoped* `docker compose ... up` — it tries to
  start `caddy` and fails. Always scope to the real services.

## What actually runs

| Component | Reality |
|---|---|
| TLS / edge | **host nginx** (systemd), `server_name test.verisphere.co`, `location /api/ -> http://127.0.0.1:8070` |
| app port | published `127.0.0.1:8070:8070` (loopback) |
| Caddy | **dormant** — defined in `docker-compose.prod.yml`, container never started |
| running app container | `verisphere-app-1`, label `com.docker.compose.project.config_files=/home/ljstrauss/verisphere/docker-compose.yml` |
| services actually run | `postgres`, `app`, `worker`, `vsp-treasury-worker` (+ `frontend`, `grafana` from docker-compose.yml, shown as "orphans" to the prod files) |
| services defined but NOT run | `caddy` |

## The divergence, and why it exists

The repo was authored for a **Caddy-fronted** production topology:
`docker-compose.prod.yml` makes Caddy the edge (binds 80/443 -> app:8070) and gives
the app *no* host port ("reached only via Caddy", base file line ~61).

At some point the edge was moved to **host nginx** (likely for shared cert/vhost
management with other sites on the box), but the compose files were never reconciled:
- `docker-compose.prod.nginx.yml` was added to re-publish the app on loopback for
  nginx — but it does NOT remove/disable the `caddy` service.
- The actually-running containers are from `docker-compose.yml` (dev), not the prod
  files. Driving them with `-f prod.yml -f prod.nginx.yml` mostly works because the
  image and resolved env are the same, but compose considers the running containers
  to belong to a different config (hence "orphan" warnings and recreate churn).

Net: **three compose files are in play, none is cleanly authoritative, and one
defines a service (caddy) that must not start.** This is an audit finding (F-4).

## Operational rules that follow

1. **Recreate the app with BOTH prod files**, or the loopback publish is dropped and
   every `/api/*` 502s (connection refused to 127.0.0.1:8070):
   ```
   docker compose -f docker-compose.prod.yml -f docker-compose.prod.nginx.yml \
     up -d --force-recreate app
   ```
2. **Never run an unscoped `up`** against the prod files — it will try to start
   `caddy` and fail with `address already in use` (nginx owns 80/443). Scope to the
   real services (see `tools/vsp-up.sh`, which names them explicitly).
3. **After a reboot**, `/dev/shm/vsp-resolved.env` is gone (tmpfs). Resolve first:
   ```
   VSP_NETWORK=fuji APP_ENV=dev tools/vsp-env-resolve.sh --write /dev/shm/vsp-resolved.env
   ```
   then `tools/vsp-up.sh` (or `systemctl start verisphere.service`).
4. **Verify the nginx upstream** after any app recreate:
   ```
   docker port verisphere-app-1 | grep 127.0.0.1:8070   # must be present
   curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8070/healthz  # 200
   ```

## The real fix (deferred, tracked)

This should be reconciled before mainnet, one of:
- **(a)** Commit to host-nginx: remove/disable `caddy` from the prod compose (a
  `profiles: [caddy]` guard so it never starts unless explicitly requested), and make
  the running stack originate from one authoritative compose file set.
- **(b)** Commit to Caddy: stop host nginx, run the caddy service, drop the loopback
  override. (Unlikely — host nginx is probably there for good reasons.)

Until then, this doc + the explicit service list in `vsp-up.sh` are the mitigation.

## Canonical service list (what `vsp-up.sh` brings up)
`postgres app worker vsp-treasury-worker`
(`frontend`/`grafana` are managed via `docker-compose.yml`; `caddy` is never started.)
