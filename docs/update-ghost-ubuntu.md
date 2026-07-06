# Update Ghost Ubuntu

File: [`workflows/update-ghost-ubuntu.json`](../workflows/update-ghost-ubuntu.json)

Daily OS-level patching (`apt update && apt upgrade -y`) for the Ubuntu host
that runs Ghost, reporting the last 10 lines of output to Telegram. This is
the OS-patching counterpart to [Update Ghost Blog](update-ghost-blog.md),
which only updates the Ghost application itself — the two are separate
workflows against the same host.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Schedule Trigger | `scheduleTrigger` | Cron `0 2 * * *` — 2am daily. |
| Execute a command | `ssh` | Runs `apt update 2>&1 && echo "---UPDATE DONE---" && apt upgrade -y 2>&1` (private key auth). |
| Code in JavaScript | `code` | Trims `stdout` down to its last 10 lines so the Telegram message stays short. |
| If | `if` | Checks for non-empty `stderr` or non-zero `code`. |
| Send Success / Send Failure | `telegram` | Reports the trimmed output either way. |

## Adapting

1. Create an **SSH Private Key** credential for the target host and assign it
   to **Execute a command** (currently `REPLACE_ME`, named
   `SSH Private Key - Ghost Server`). This can be the same credential used in
   Update Ghost Blog if it's the same host.
2. Create a Telegram credential and reassign it on **Send Success** and
   **Send Failure**; set `chatId` on both.
3. After importing [Error Handler](error-handler.md), set this workflow's
   **Settings → Error Workflow** to it.
4. This workflow runs `apt upgrade -y` unattended and reboots are not
   handled — if a kernel/library update needs a reboot to take effect, that's
   on you to handle separately (e.g. a follow-on check for
   `/var/run/reboot-required`).
