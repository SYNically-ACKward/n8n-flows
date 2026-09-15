# Watchtower Update Notifier

File: [`workflows/watchtower-update-notifier.json`](../workflows/watchtower-update-notifier.json)

Receives session reports from [Watchtower](https://github.com/nicholas-fedor/watchtower)
(the maintained `nickfedor/watchtower` fork), which pulls and recreates
containers by itself. Every report is forwarded to Telegram right away.

When the report lists a container that failed for a **transient registry
reason** (rate limit, TLS timeout and similar), the workflow also waits 15
minutes. It then SSHes to the host that sent the report, reruns Watchtower for
only those containers, and posts the result. Without this, a transient failure
sits until the next scheduled Watchtower run, which can be hours away.

This is the successor to [DIUN Update Notifier](diun-update-notifier.md) +
[Container Update Form](container-update-form.md). Diun only *told* you an
image was stale. Watchtower applies the update, so the notification becomes a
receipt rather than a to-do.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Webhook | `webhook` | `POST` endpoint at path `/watchtower`. Watchtower's Shoutrrr `generic://` notifier posts here. |
| Send a text message | `telegram` | Forwards the report text as-is. |
| Find Retryable Failures | `code` | Parses the report. Works out which host sent it from the header line. Collects `❌` containers whose error matches a transient pattern. Emits nothing (ending the branch) if there are none. |
| Wait Before Retry | `wait` | 15 minutes. That gives a drained registry bucket or a network blip time to recover. |
| Which Host? | `if` | Routes to the SSH node for the host that sent the report. |
| Retry via Watchtower (DMZ / Internal) | `ssh` | Runs a targeted one-off of the same Watchtower config: `docker compose run --rm -e WATCHTOWER_NOTIFICATION_URL= watchtower --run-once <containers>`. Up to 3 attempts, 2 minutes apart. |
| Format Retry Result | `code` | Builds a ✅ / ❌ summary from the SSH output. |
| Notify Retry Result | `telegram` | Sends that summary. |

## The report template this expects

The parser reads Watchtower's report rendered with a template like this, set
as `WATCHTOWER_NOTIFICATION_TEMPLATE` with `WATCHTOWER_NOTIFICATION_REPORT=true`:

```gotemplate
{{- if .Report -}}
{{- with .Report -}}
{{- if (or .Updated .Failed) -}}
🐳 Watchtower — internal-host
{{ range .Updated }}
✅ {{ .Name }} — {{ .ImageName }}
   {{ .CurrentImageID.ShortID }} → {{ .LatestImageID.ShortID }}
{{- end }}
{{- range .Failed }}
❌ {{ .Name }} — {{ .ImageName }}
   {{ .State }}: {{ .Error }}
{{- end }}
{{- end -}}
{{- end -}}
{{- end -}}
```

Two things it depends on:

- **The header names the host.** Use `internal-host` on one Watchtower and
  `dmz-host` on the other, or change both the template and the two regexes at
  the top of **Find Retryable Failures** to your own labels.
- **Each failure is two lines**: `❌ <container> — <image>`, then the error on
  the next line.

## Design notes

- **Why a delayed retry and not our own rate limiting.** In the failure that
  prompted this, `lscr.io` / `ghcr.io` returned a 429 (`retry-after 747µs
  allowed 44000 per 1m`). Anonymous GHCR pulls for an org share one token
  bucket with everyone else pulling anonymously, so the quota was drained by
  other people's traffic. Throttling your own checks does not help: Watchtower
  already checks anonymous GHCR images one at a time. Its built-in 429 retry has
  a fixed 30-second budget that cannot be configured. Retrying minutes later
  can.
- **The retry cannot loop.** The one-off run blanks
  `WATCHTOWER_NOTIFICATION_URL`, so it never posts back to this webhook. Its
  result goes to Telegram from n8n instead.
- **Success comes from the session summary, not the exit code.** `--run-once`
  exits 0 even when an update fails. The script reads the `Update session
  completed … failed=N scanned=N` line instead. `scanned=0` means the container
  no longer exists, so it gives up instead of retrying.
- **Only transient errors are retried**: rate limit / `toomanyrequests` / 429,
  TLS handshake timeout, i/o timeout, connection reset, deadline exceeded,
  unexpected EOF. A bad tag or a container-create error would just fail again.
- **The webhook is unauthenticated, and container names reach a shell.**
  Names are therefore checked against Docker's own container-name charset
  (`^[A-Za-z0-9][A-Za-z0-9_.-]*$`), and anything else is dropped. A spoofed
  name such as ``x;rm$(id)`` produces no retry. Don't loosen that check. Also
  consider not exposing the webhook beyond your LAN.
- **The one-off does not conflict with the scheduled Watchtower.** A
  `--run-once` instance leaves the long-running one alone. A 15-minute delay
  also cannot overlap an hourly-or-slower schedule.

## Shoutrrr / Compose gotchas

- **`generic://` posts `Content-Type: text/plain`**, so n8n exposes the report
  as a bare string in `$json.body`, not `$json.body.message`. Both nodes that
  read it handle either shape. Get this wrong and Telegram quietly delivers the
  text `undefined`.
- **Write `$` as `$$` inside the template** if you set it through Docker
  Compose. Compose interpolates `$var` in environment values, which mangles Go
  template variables. Watchtower then falls back to its default template.

## Adapting

1. Create a Telegram credential, assign it to **Send a text message** and
   **Notify Retry Result** (currently `REPLACE_ME`), and set `chatId` on both.
2. Recreate the two SSH Private Key credentials and assign them to the two
   **Retry via Watchtower** nodes. These are the same per-host credentials as
   [Container Update Form](container-update-form.md). The SSH user must be able
   to run `docker compose` without a password (e.g. in the `docker` group).
3. Replace `<path-to-internal-watchtower-compose-dir>` and
   `<path-to-dmz-watchtower-compose-dir>` with the directory holding each
   host's Watchtower `docker-compose.yml`. The service in it must be called
   `watchtower`.
4. **Only one host?** Delete **Which Host?** and **Retry via Watchtower (DMZ)**,
   and wire **Wait Before Retry** straight to the remaining SSH node.
5. Activate the workflow. Point each Watchtower at
   `generic://<your-n8n-domain>/webhook/watchtower`.
6. To test it end to end without waiting for a real failure, POST a fake report
   naming a harmless container:

   ```bash
   printf '%s' $'🐳 Watchtower — internal-host\n\n❌ some-container — some/image:latest\n   Failed: registry rate limited' \
     | curl -X POST -H 'Content-Type: text/plain' --data-binary @- https://<your-n8n-domain>/webhook/watchtower
   ```

   Expect the forwarded message immediately and a `🔁` result about 15 minutes
   later.
