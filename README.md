# Homelab n8n Workflows

A small collection of n8n workflows used to run day-to-day maintenance on a
Docker-based homelab: patching hosts, updating a Ghost blog, notifying about
available container image updates, and one-click-updating containers from a
web form. All of them report success/failure to Telegram, and share one
central error-handling workflow.

These are exported from a real, in-use n8n instance and then **sanitized**:
credential IDs, chat IDs, hostnames, IPs, and internal file paths have been
replaced with placeholders. You will need to fill those back in for your own
environment — see [Adapting to your environment](#adapting-to-your-environment)
below, and the per-workflow doc for details specific to that flow.

## What's here

| Workflow | Trigger | Purpose |
|---|---|---|
| [Error Handler](docs/error-handler.md) | Error Trigger | Central error handler — sends a Telegram message when any workflow that references it fails. Import this one first. |
| [Update Ghost Blog](docs/update-ghost-blog.md) | Schedule (weekly) | SSHes into the host running Ghost and runs `ghost update`. |
| [Update Ghost Ubuntu](docs/update-ghost-ubuntu.md) | Schedule (daily) | SSHes into the Ghost server and runs `apt update && apt upgrade`. |
| [Update AMP Server](docs/update-amp-server.md) | Schedule (daily) | SSHes into a separate server (password auth) and runs `apt update && apt upgrade`. |
| [DIUN Update Notifier](docs/diun-update-notifier.md) | Webhook | Receives [Diun](https://crazymax.dev/diun/) container-image-update notifications and forwards them to Telegram with a link to trigger an update. |
| [Container Update Form](docs/container-update-form.md) | Form | A web form to pick a Docker Compose service and have n8n SSH in, `docker compose pull && up -d` it, and report the result. |

Workflows that existed on the source instance but were dev/test scaffolding
with no real usage (an in-progress "approval" variant, a default "My workflow"
stub, one never-activated draft) were left out — this repo is only the
workflows that were actually active and running.

## Prerequisites

- An n8n instance (self-hosted or cloud).
- A Telegram bot (via [@BotFather](https://t.me/BotFather)) if you want the
  Telegram notifications — get a bot token and your numeric chat ID (message
  [@userinfobot](https://t.me/userinfobot) or call `getUpdates` on your bot to
  find it).
- SSH access (key or password) from the n8n host to whatever machine(s) a
  given workflow manages.
- For the DIUN workflow: a running [Diun](https://crazymax.dev/diun/) instance
  configured to POST update notifications to this workflow's webhook URL.

## Importing

In the n8n UI: **Workflows → Add workflow → Import from File**, and pick a
JSON file from `workflows/`. Import **Error Handler first** — the other
workflows are meant to point their "Error Workflow" setting at it, and that's
easiest to wire up if it already exists.

Each JSON file is a minimal, importable workflow definition (name, nodes,
connections, settings) — the execution history, ownership/project metadata,
and internal n8n IDs from the source instance have been stripped since none of
that is needed (or wanted) in someone else's instance.

## Adapting to your environment

Every workflow needs some combination of the following after import. See each
workflow's doc for exactly which apply:

1. **Recreate credentials.** Each node with a `REPLACE_ME` credential ID needs
   you to create a new credential of that type in n8n (Telegram API, SSH
   Private Key, or SSH Password) and reassign it on the node. The `name`
   field tells you which real-world account/host each one is for.
2. **Set your Telegram chat ID.** Every `chatId` field is set to the literal
   string `YOUR_TELEGRAM_CHAT_ID` — replace it with your own.
3. **Re-link the Error Workflow setting.** The original `settings.errorWorkflow`
   pointed at a workflow ID from the source instance, which won't exist in
   yours, so it was removed. After importing Error Handler, open each other
   workflow's **Settings** panel and set **Error Workflow** to it manually.
4. **Fix host-specific paths/commands.** SSH `command` fields use placeholders
   like `<path-to-ghost-install>` or `<path-to-compose-files>` — replace with
   the real paths on your target host(s).
5. **Re-enable the workflow.** Imported workflows come in inactive; turn each
   one on once its credentials and settings are correct.

## Repo layout

```
workflows/   sanitized, importable workflow JSON
docs/        one doc per workflow: what it does, node-by-node, how to adapt it
```
