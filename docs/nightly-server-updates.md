# Nightly Server Updates (AMP / DMZ / Ghost / Docker Host)

File: [`workflows/nightly-server-updates.json`](../workflows/nightly-server-updates.json)

The consolidated replacement for three formerly-separate workflows (Update
AMP Server, Update Ghost Ubuntu, and a short-lived Update DMZ Server draft).
One schedule now fans out to patch **four** hosts in parallel — three over a
normal synchronous SSH command, plus a fourth (the Docker host that runs n8n
itself) handled specially, since that one can't safely run synchronously.

If you're adapting this for a much smaller setup (one or two hosts, no
self-hosting concerns), it's easier to strip this down to just the "normal"
branch pattern — see step 5 below — rather than keep the self-update
machinery around.

## Why the Docker host branch is different

`n8n` in this setup runs as a container on a Docker host. Patching that host
means `apt upgrade` plus a `docker ... prune` pass — which can restart the
Docker daemon and, with it, the n8n container (and the SSH session n8n is
using to run the command) mid-update. A synchronous SSH node would hang or
error out unpredictably in that case, and n8n itself might not survive to
report the result.

The workaround: fire the update in the background, detached from the SSH
session, and check on it from a **second, later** trigger instead of waiting
on it:

1. `Kickoff Docker Host Update` opens the SSH session, launches the real
   update command wrapped in `nohup setsid bash -c '...' > ~/host-update.log
   2>&1 < /dev/null &`, and returns immediately — before n8n's own container
   is at risk of restarting.
2. `Check Trigger (2:20am)` fires 20 minutes later (assumed to be enough time
   for the update + prune to finish) and `Check Docker Host Update` just
   `cat`s the log file to see what happened.
3. Both paths feed into the same tagging/checklist/notify nodes as the other
   three hosts, so you get one consistent report either way.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Schedule Trigger | `scheduleTrigger` | Cron `0 2 * * *` — 2am daily; fans out to all three synchronous update nodes plus the Docker host kickoff. |
| Update AMP | `ssh` | `apt update/upgrade` + `docker ... prune -f` (containers, networks, images, builder) on the AMP server, password auth. |
| Update DMZ | `ssh` | Same command, on the DMZ host, key auth. |
| Update Ghost | `ssh` | `apt update/upgrade` only (no Docker on this host), on the Ghost server, key auth. |
| Tag AMP / Tag DMZ / Tag Ghost | `code` | Each stamps `hostLabel` (for the notification) and `hasDocker` (whether the checklist below should expect a Docker-cleanup stage). |
| Kickoff Docker Host Update | `ssh` | Fires the background update job on the Docker host and returns immediately — see above. |
| Check Trigger (2:20am) | `scheduleTrigger` | Cron `20 2 * * *` — 20 minutes after the main run. |
| Check Docker Host Update | `ssh` | Reads back `~/host-update.log` from the Docker host. |
| Tag Docker Host | `code` | Parses the `===EXIT_CODE:N===` marker the kickoff command appends to the log, and stamps `hostLabel`/`hasDocker` like the other three. |
| Code in JavaScript | `code` | Builds a per-stage checklist (apt update → apt upgrade → Docker cleanup, the last stage only if `hasDocker`) by pattern-matching `stdout`, and marks `status` success/failure. |
| Check Success | `if` | Branches on `$json.status`. |
| Notify Success / Notify Failure | `telegram` | Reports `hostLabel` plus the checklist (and error detail, on failure). |

## Adapting

1. **Recreate the SSH credentials** — four are expected:
   - `SSH Password - AMP Server` on **Update AMP**
   - `SSH Private Key - DMZ Docker Host` on **Update DMZ**
   - `SSH Private Key - Ghost Server` on **Update Ghost**
   - `SSH Private Key - Internal Docker Host` on both **Kickoff Docker Host
     Update** and **Check Docker Host Update** (same host, same credential,
     used twice)
2. Create a Telegram credential and reassign it on **Notify Success** and
   **Notify Failure**; set `chatId` on both.
3. After importing [Error Handler](error-handler.md), set this workflow's
   **Settings → Error Workflow** to it.
4. Adjust the two cron expressions if 2:00am/2:20am doesn't fit your window
   — the 20-minute gap just needs to be longer than the Docker host's update
   typically takes.
5. **Don't need the self-update branch?** If you're not running n8n on a
   host you also want to patch this way, delete **Kickoff Docker Host
   Update**, **Check Trigger (2:20am)**, **Check Docker Host Update**, and
   **Tag Docker Host**, and just add a normal synchronous `ssh` node (+ its
   own `Tag <Host>` node) wired straight into **Code in JavaScript**, the
   same way AMP/DMZ/Ghost are — one less thing to reason about.
6. If a host doesn't run Docker, set `hasDocker: false` in its `Tag <Host>`
   node so the checklist doesn't expect a cleanup stage that will never run.
7. The success/failure checklist logic in **Code in JavaScript** pattern-matches
   specific strings (`---UPDATE DONE---`, a `"N upgraded, N newly installed"`
   regex) that come from the exact `apt` commands used here — if you change
   the commands, update the `stages` array to match.
