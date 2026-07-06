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
| Run Update via SSH | `ssh` | For each internal service: `cd <path-to-compose-files>/<service> && docker compose pull && docker compose up -d`. |
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
8. Selecting several services in one submission fires all of them
   concurrently (n8n doesn't serialize `splitOut` branches) — if your host
   can't handle parallel `docker compose pull`s well, keep selections to one
   service at a time.
