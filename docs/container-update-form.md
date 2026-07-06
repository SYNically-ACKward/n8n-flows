# Container Update Form

File: [`workflows/container-update-form.json`](../workflows/container-update-form.json)

A self-serve web form: pick a Docker Compose service from a dropdown, and n8n
SSHes into the right host, runs `docker compose pull && docker compose up -d`
in that service's directory, and posts the result to Telegram. Built to pair
with [DIUN Update Notifier](diun-update-notifier.md), whose alerts link
straight to this form.

This version supports **two Docker hosts** — an "internal" one and a "DMZ"
one — routed by prefixing the DMZ dropdown entries with `dmz:`. If you only
manage one host, see step 4 below to simplify it.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Update Form | `formTrigger` | Renders a form with one required dropdown, **Service**. |
| Route by Host | `if` | Branches on whether `$json.Service` starts with `dmz:`. |
| Run Update via SSH | `ssh` | For non-DMZ services: `cd <path-to-compose-files>/<service> && docker compose pull && docker compose up -d`. |
| Run Update via SSH (DMZ) | `ssh` | For `dmz:`-prefixed services: strips the `dmz:` prefix, then runs the same compose pull/up in the DMZ host's compose directory. |
| Notify Complete | `telegram` | Reports which host/service was updated and the last part of the command output. |

## Adapting

1. **Rebuild the dropdown.** Replace the four placeholder options
   (`example-service-1`, `example-service-2`, `example-service-3`,
   `dmz:example-service`) on the **Update Form** node with the actual
   subdirectory names under your Docker Compose project root(s) — this
   workflow assumes a `<compose-root>/<service-name>/docker-compose.yml`
   layout per service.
2. **Set your compose-file paths.** On **Run Update via SSH**, replace
   `<path-to-compose-files>` with your compose project root. On
   **Run Update via SSH (DMZ)**, replace `<path-to-dmz-compose-files>`
   likewise.
3. **Recreate the SSH credentials.** Two separate SSH Private Key
   credentials are expected — one per host, assigned to the two SSH nodes
   (currently `REPLACE_ME` / `SSH Private Key - Internal Docker Host` and
   `SSH Private Key - DMZ Docker Host`).
4. **Only have one host?** Delete the **Route by Host** and
   **Run Update via SSH (DMZ)** nodes, and connect **Update Form** directly to
   **Run Update via SSH**. Drop the `dmz:`-prefix convention from your
   dropdown values too, since it won't mean anything anymore.
5. Create a Telegram credential and reassign it on **Notify Complete**; set
   `chatId`.
6. This workflow has no Error Workflow setting to relink — failures inside
   the SSH command itself are reported by **Notify Complete**'s message text
   (it always fires, success or failure, and includes the command output),
   rather than routed through n8n's error-workflow mechanism.
7. The SSH user needs permission to run `docker compose` in the target
   directories without a password prompt (e.g. is in the `docker` group).
