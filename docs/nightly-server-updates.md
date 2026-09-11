# Nightly Server Updates (AMP / DMZ / Ghost / Technitium / Docker Host)

File: [`workflows/nightly-server-updates.json`](../workflows/nightly-server-updates.json)

The consolidated replacement for three formerly-separate workflows (Update
AMP Server, Update Ghost Ubuntu, and a short-lived Update DMZ Server draft).
One schedule now fans out to patch **five** hosts in parallel — four over a
normal synchronous SSH command, plus a fifth (the Docker host that runs n8n
itself) handled specially, since that one can't safely run synchronously.

If you're adapting this for a much smaller setup (one or two hosts, no
self-hosting concerns), it's easier to strip this down to just the "normal"
branch pattern — see step 5 below — rather than keep the self-update
machinery around.

## Schedule

Both triggers use six-field cron expressions (n8n's cron includes a leading
seconds field):

| Trigger | Expression | Meaning |
|---|---|---|
| Schedule Trigger | `0 0 2 * * 0,2,4` | 02:00 on Sunday, Tuesday, Thursday |
| Check Trigger (2:20am) | `0 20 2 * * 0,2,4` | 02:20 the same mornings |

Three nights a week rather than nightly, deliberately: an unattended
`apt upgrade` that touches the container runtime is disruptive enough that
doing it every night is more risk than it's worth.

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
   hosts, so you get one consistent report either way.

> **Worth knowing before you copy this pattern:** stopping a few dozen
> containers can overrun systemd's default `TimeoutStopSec=90s`, in which case
> the daemon is SIGKILLed mid-shutdown — and `restart: unless-stopped` does
> *not* fire for a container the daemon never finished stopping, so they stay
> exited with nothing to tell you. If you run this against a host with many
> containers, raise `TimeoutStopSec` on `docker.service` first.

## The AMP branch was re-enabled 2026-09-11

It had shipped **disabled and disconnected** while that host was boxed for the
house move: `Update AMP` and `Tag AMP` carried `"disabled": true`, and the
`Schedule Trigger → Update AMP` edge was absent. Both nodes are now enabled and
the trigger edge is reconnected.

That original combination is still worth understanding if you ever pause a host
yourself, because **a disabled n8n node passes its input straight through to its
output.** Disabling the two AMP nodes *alone* would still let the trigger's item
flow into `Code in JavaScript` carrying no `stdout` and no exit code — and that
node computes `failed = exitCode != 0`, which `undefined` satisfies. You'd get a
spurious failure notification every run. Detaching the trigger edge as well is
what actually stops the branch executing.

**Two things had to be fixed before it could be switched back on**, and both are
easy to reintroduce:

1. **`sudo env DEBIAN_FRONTEND=...` did not work on that host.** It had only a
   narrow grant — `sysadmin ALL = NOPASSWD: /usr/bin/apt` — and `env` is not
   `/usr/bin/apt`, so sudo would have prompted for a password and the node would
   have hung indefinitely rather than failing. (The n8n SSH node does not error on
   a non-zero exit, so a hang here is silent.) Resolved by giving the host the same
   `NOPASSWD:ALL` drop-in the other hosts use.

2. **`docker container prune -f` and `docker network prune -f` were removed.**
   AMP runs each game instance as a Docker container (`cubecoders/ampbase:debian`,
   named `AMP_<instance>`), so a *stopped* game server is an ordinary state — and
   the nightly prune would have destroyed its container. The node now prunes images
   and builder cache only, matching the Docker-host branch.

Note this host also runs Ubuntu's own `unattended-upgrades`, so the two overlap.
That is harmless — apt serialises on its own lock — but it means this branch is
not the only thing patching the box.
## Nodes

| Node | Type | What it does |
|---|---|---|
| Schedule Trigger | `scheduleTrigger` | Fans out to the synchronous update nodes plus the Docker host kickoff. |
| Update AMP | `ssh` | `apt update/upgrade` + `docker image/builder prune -f`, password auth. **Re-enabled 2026-09-11**; container/network prune deliberately removed — see above. |
| Update DMZ | `ssh` | Same command, on the DMZ host, key auth. |
| Update Ghost | `ssh` | `apt update/upgrade` only (no Docker on this host), key auth. |
| Update Technitium Host | `ssh` | `apt update/upgrade` only, on the host running DNS. Key auth. |
| Tag AMP / Tag DMZ / Tag Ghost / Tag Technitium | `code` | Each stamps `hostLabel` (for the notification) and `hasDocker` (whether the checklist below should expect a Docker-cleanup stage). |
| Kickoff Docker Host Update | `ssh` | Fires the background update job on the Docker host and returns immediately — see above. |
| Check Trigger (2:20am) | `scheduleTrigger` | 20 minutes after the main run. |
| Check Docker Host Update | `ssh` | Reads back `~/host-update.log` from the Docker host. |
| Tag Docker Host | `code` | Parses the `===EXIT_CODE:N===` marker the kickoff command appends to the log, and stamps `hostLabel`/`hasDocker` like the others. |
| Code in JavaScript | `code` | Builds a per-stage checklist (apt update → apt upgrade → Docker cleanup, the last stage only if `hasDocker`) by pattern-matching `stdout`, and marks `status` success/failure. |
| Check Success | `if` | Branches on `$json.status`. |
| Notify Success / Notify Failure | `telegram` | Reports `hostLabel` plus the checklist (and error detail, on failure). |

Every host gets its own notification — the report node runs per item, so a
five-host run produces five messages rather than one digest.

## Adapting

1. **Recreate the SSH credentials** — five are expected:
   - `SSH Password - AMP Server` on **Update AMP**
   - `SSH Private Key - DMZ Docker Host` on **Update DMZ**
   - `SSH Private Key - Ghost Server` on **Update Ghost**
   - `SSH Private Key - Technitium Host` on **Update Technitium Host**
   - `SSH Private Key - Internal Docker Host` on both **Kickoff Docker Host
     Update** and **Check Docker Host Update** (same host, same credential,
     used twice)
2. Create a Telegram credential and reassign it on **Notify Success** and
   **Notify Failure**; set `chatId` on both.
3. After importing [Error Handler](error-handler.md), set this workflow's
   **Settings → Error Workflow** to it.
4. Adjust the two cron expressions if 02:00/02:20 doesn't fit your window
   — the 20-minute gap just needs to be longer than the Docker host's update
   typically takes.
5. **Don't need the self-update branch?** If you're not running n8n on a
   host you also want to patch this way, delete **Kickoff Docker Host
   Update**, **Check Trigger (2:20am)**, **Check Docker Host Update**, and
   **Tag Docker Host**, and just add a normal synchronous `ssh` node (+ its
   own `Tag <Host>` node) wired straight into **Code in JavaScript**, the
   same way the other hosts are — one less thing to reason about.
6. If a host doesn't run Docker, set `hasDocker: false` in its `Tag <Host>`
   node so the checklist doesn't expect a cleanup stage that will never run.
7. The success/failure checklist logic in **Code in JavaScript** pattern-matches
   specific strings (`---UPDATE DONE---`, a `"N upgraded, N newly installed"`
   regex) that come from the exact `apt` commands used here — if you change
   the commands, update the `stages` array to match.
8. **The SSH node does not error on a non-zero exit code.** It resolves
   successfully with the exit status in `$json.code`, so `retryOnFail` won't
   fire for a failed remote command — which is exactly why the checklist node
   inspects `code`/`stderr` itself rather than relying on node failure.
