# Zuma demo – weekly Zammad cleanup

File: [`workflows/zuma-zammad-cleanup.json`](../workflows/zuma-zammad-cleanup.json)

A deliberately minimal "run a maintenance script on a schedule" workflow. It
SSHes into the Docker host once a week and runs a cleanup script that removes
the leftovers a demo leaves in a Zammad helpdesk: notes the demo's agent added
to the seeded tickets, and the approval tickets it opened. The script itself
lives in the (private) infra repo, not here.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Sunday 05:00 | `scheduleTrigger` | Cron `0 0 5 * * 0`, with the workflow timezone set explicitly so it doesn't follow the instance default. Picked to stay clear of the nightly patching and backup windows. |
| Run Zammad cleanup | `ssh` | Runs `python3 <path-to>/zuma-zammad-cleanup.py --apply 2>&1` (private key auth). |

## Why there is no success/failure branch

n8n's SSH node does **not** fail on a non-zero exit code, so an `If` on the
result (as in the other workflows) is the usual workaround. Here the script
sends its own Telegram alert on failure and exits non-zero, which keeps the
alerting next to the logic that knows what went wrong, and still works if the
script is run by hand. The node output (one line per deletion plus a summary
line) is kept in the execution log.

## Adapting

1. Create an **SSH Private Key** credential for your host and assign it to
   **Run Zammad cleanup** (currently `REPLACE_ME`).
2. Replace `<path-to>` with wherever your script lives.
3. Set **Settings → Error Workflow** to your [Error Handler](error-handler.md)
   (currently `REPLACE_ME`).
