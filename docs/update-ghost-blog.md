# Update Ghost Blog

File: [`workflows/update-ghost-blog.json`](../workflows/update-ghost-blog.json)

Runs `ghost update` (the [Ghost-CLI](https://ghost.org/docs/ghost-cli/) command)
on a schedule to keep a self-hosted Ghost blog current, and reports the result
to Telegram.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Schedule Trigger | `scheduleTrigger` | Cron `0 3 * * 0` — 3am every Sunday. |
| Execute a command | `ssh` | Runs `runuser -l <ghost-linux-user> -c 'cd <path-to-ghost-install> && ghost update' 2>&1` over SSH (private key auth). `runuser -l` is used because Ghost-CLI refuses to run as root. |
| If | `if` | Checks whether `stderr` is non-empty or the exit `code` isn't `0`. |
| Send Success / Send Failure | `telegram` | Reports the last portion of command output either way. |

## Adapting

1. Create an **SSH Private Key** credential pointed at your Ghost host/user,
   and assign it to **Execute a command** (currently `REPLACE_ME`, named
   `SSH Private Key - Ghost Server`).
2. Edit the command on **Execute a command**: replace `<ghost-linux-user>`
   with the OS user Ghost-CLI was installed under, and `<path-to-ghost-install>`
   with the Ghost site directory (e.g. `/var/www/ghost`).
3. Create a Telegram credential and reassign it on **Send Success** and
   **Send Failure**; set `chatId` on both.
4. After importing [Error Handler](error-handler.md), set this workflow's
   **Settings → Error Workflow** to it.
5. Adjust the cron expression on **Schedule Trigger** if weekly-on-Sunday
   doesn't fit your maintenance window.
