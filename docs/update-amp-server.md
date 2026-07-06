# Update AMP Server

File: [`workflows/update-amp-server.json`](../workflows/update-amp-server.json)

Same pattern as [Update Ghost Ubuntu](update-ghost-ubuntu.md), against a
different host — daily `apt update && apt upgrade -y`, trimmed output posted
to Telegram. The only structural difference is that this one authenticates
over SSH with a **password** instead of a private key (kept as a second
example of both n8n SSH auth modes).

## Nodes

| Node | Type | What it does |
|---|---|---|
| Schedule Trigger | `scheduleTrigger` | Cron `0 2 * * *` — 2am daily. |
| Execute a command | `ssh` | Runs `sudo apt update 2>&1 && echo "---UPDATE DONE---" && sudo apt upgrade -y 2>&1` (password auth). |
| Code in JavaScript | `code` | Trims `stdout` to its last 10 lines. |
| If | `if` | Checks for non-empty `stderr` or non-zero `code`. |
| Send Success / Send Failure | `telegram` | Reports the trimmed output either way. |

## Adapting

1. Create an **SSH Password** credential for the target host and assign it
   to **Execute a command** (currently `REPLACE_ME`, named
   `SSH Password - AMP Server`).
2. The command uses `sudo` (rather than `runuser`/root SSH like the other two
   update workflows) — make sure the SSH user has passwordless sudo for `apt`,
   or adjust the command to match how you've set up privilege escalation on
   your host.
3. Create a Telegram credential and reassign it on **Send Success** and
   **Send Failure**; set `chatId` on both.
4. After importing [Error Handler](error-handler.md), set this workflow's
   **Settings → Error Workflow** to it.
