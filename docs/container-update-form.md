# Container Update Form

File: [`workflows/container-update-form.json`](../workflows/container-update-form.json)

A self-serve web form: pick one or more Docker Compose services from two
multi-select dropdowns (internal host / DMZ host), and n8n SSHes into the
right host for each and runs `docker compose pull && docker compose up -d`
in that service's directory, reporting each service's result to Telegram
separately. Built to pair with
[DIUN Update Notifier](diun-update-notifier.md), whose alerts link straight
to this form.

This is a rework of an earlier single-dropdown version: instead of one
`Service` field with a `dmz:`-prefix convention to pick the host, there are
now two separate multi-select fields — `Internal Services` and
`DMZ Services` — and you can select any combination of services across both
in one submission.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Update Form | `formTrigger` | Renders a form with two optional multi-select dropdowns: **Internal Services** and **DMZ Services**. |
| Split Internal Services | `splitOut` | Turns a multi-value `Internal Services` selection into one item per service. |
| Split DMZ Services | `splitOut` | Same, for `DMZ Services`. |
| Run Update via SSH | `ssh` | For each internal service: `cd <path-to-compose-files>/<service>`, then `docker compose pull` **retried up to 3× with a 10s backoff**, then `docker compose up -d`. |
| Run Update via SSH (DMZ) | `ssh` | Same, against the DMZ host's compose directory, for each DMZ service. |
| Tag Internal / Tag DMZ | `code` | Stamp `hostLabel` and `serviceName` onto each result for the notification text. |
| Code in JavaScript | `code` | Determines success/failure **by exit code only** — `docker compose pull`/`up` write normal progress output to `stderr` even when they succeed, so checking for non-empty `stderr` (as a naive check would) produces false failures. Also trims output to the last 2000 characters. |
| Check Success | `if` | Branches on `$json.status`. |
| Notify Success / Notify Failure | `telegram` | One message per service, per host, with the trimmed command output. |

## Adapting

1. **Rebuild the dropdowns.** Replace the placeholder options
   (`example-service-1/2/3` under Internal, `dmz-example-service-1/2` under
   DMZ) on the **Update Form** node with the actual subdirectory names under
   your Docker Compose project root(s) — this workflow assumes a
   `<compose-root>/<service-name>/docker-compose.yml` layout per service, on
   each host.
2. **Set your compose-file paths.** On **Run Update via SSH**, replace
   `<path-to-compose-files>` with your internal host's compose project root.
   On **Run Update via SSH (DMZ)**, replace `<path-to-dmz-compose-files>`
   with the DMZ host's.
3. **Recreate the SSH credentials.** Two separate SSH Private Key
   credentials are expected — one per host, assigned to the two SSH nodes
   (currently `REPLACE_ME` / `SSH Private Key - Internal Docker Host` and
   `SSH Private Key - DMZ Docker Host`). These are the same two credentials
   used by the Docker-host branches of
   [Nightly Server Updates](nightly-server-updates.md), if you're using both
   workflows.
4. **Only have one host?** Delete the **DMZ Services** field, **Split DMZ
   Services**, and **Run Update via SSH (DMZ)**, and wire **Split Internal
   Services** straight through to **Tag Internal** → **Code in JavaScript**
   as the only path.
5. Create a Telegram credential and reassign it on **Notify Success** and
   **Notify Failure**; set `chatId` on both.
6. This workflow has no Error Workflow setting to relink — failures inside
   the SSH command itself are reported per-service by **Notify Failure**
   (which always fires on non-zero exit and includes the command output),
   rather than routed through n8n's error-workflow mechanism.
7. The SSH user needs permission to run `docker compose` in the target
   directories without a password prompt (e.g. is in the `docker` group).
8. Selecting several services in one submission processes them **sequentially**
   — the SSH node loops over its input items one at a time, and the DMZ branch
   only starts after the internal branch finishes. (Verified 2026-07-31 from
   execution timing data; an earlier version of this doc wrongly claimed the
   branches fired concurrently.) The only parallelism is *within* a single
   `docker compose pull`, which fetches a stack's images side by side.

## Reliability notes

Pulls intermittently fail with `net/http: TLS handshake timeout` against
Docker Hub / ghcr.io. This is not specific to any one service — observed
hitting `gitea/gitea`, `postgres`, `grafana`, and `paperless-ngx` on
different runs, across both registries. A single failed image aborts that
stack's whole `docker compose pull`, so the update reports as failed.

Two things make this worse than it looks:

- The **SSH node does not error on a non-zero exit code** — it returns the
  exit code as data (`$json.code`). So n8n's own `retryOnFail` never fires
  for a failed `docker compose pull`; the retry has to live *inside* the
  SSH command. That's what the `for attempt in 1 2 3` loop does.
- The more services you select, the more images are pulled, so the odds
  that at least one hits the intermittent timeout compound.

`retryOnFail` is still set on both SSH nodes, but only covers genuine
SSH/connection errors (the node actually throwing).

Host-side, `/etc/docker/daemon.json` sets `max-concurrent-downloads: 2` to
reduce simultaneous layer fetches. Note this may be only partially honored
under Docker's containerd image store (Docker 29 default).
