# DIUN Update Notifier

File: [`workflows/diun-update-notifier.json`](../workflows/diun-update-notifier.json)

Receives webhook notifications from [Diun](https://crazymax.dev/diun/) (a
tool that watches your running containers and tells you when a newer image is
available) and relays them to Telegram, including a link to trigger the
update. It's designed to pair with
[Container Update Form](container-update-form.md) — the link in the message
opens that workflow's form so you can pull/restart the affected service in
one tap.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Webhook | `webhook` | `POST` endpoint at path `/diun`. Diun POSTs its update payload here. |
| Send a text message | `telegram` | Formats `$json.body.hostname`, `.metadata.ctn_names`, `.image`, and `.hub_link` from the Diun payload into a Telegram message, with a link to the update form. |

## Adapting

1. Create a Telegram credential and reassign it on **Send a text message**
   (currently `REPLACE_ME`); set `chatId`.
2. Note the webhook's production URL after activating this workflow
   (n8n shows it on the Webhook node — something like
   `https://<your-n8n-domain>/webhook/diun`), and point Diun's webhook
   notifier at it in Diun's own config.
3. If you're using [Container Update Form](container-update-form.md) too,
   replace the placeholder link in the message text —
   `https://YOUR_N8N_DOMAIN/form/YOUR_CONTAINER_UPDATE_FORM_WEBHOOK_ID` — with
   that workflow's actual form URL (visible on its Form Trigger node once
   activated).
4. If you're not using Container Update Form, just delete that "To Update:"
   line from the message text.
5. This workflow has no Error Workflow setting to relink — it never had one
   in the original (a failed Telegram send here would otherwise itself need
   somewhere to report to, which felt like overkill for a notify-only flow).

The payload fields this workflow reads (`hostname`, `image`, `hub_link`,
`metadata.ctn_names`) come from Diun's webhook notification format — check
Diun's own docs if you're on a version where that schema has changed.
