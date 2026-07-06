# Error Handler

File: [`workflows/error-handler.json`](../workflows/error-handler.json)

Central error handler. It doesn't do anything on its own — other workflows
point their **Settings → Error Workflow** at this one, and n8n automatically
runs it (passing in the failed execution's data) whenever one of them errors.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Error Trigger | `errorTrigger` | Entry point; fires when a linked workflow fails. Provides `$json.workflow.name` and `$json.execution.error.message`. |
| Send a text message | `telegram` | Sends `❌ Workflow failed: <name>` with the error message to Telegram. |

## Adapting

1. Create a Telegram credential and reassign it on **Send a text message**
   (currently `REPLACE_ME`).
2. Replace `chatId` (`YOUR_TELEGRAM_CHAT_ID`) with your own chat ID.
3. Import this workflow **before** the others in this repo, then activate it.
4. In every other imported workflow, go to **Settings → Error Workflow** and
   select this workflow — that link isn't preserved across instances/re-imports
   and has to be set by hand each time.

Nothing else needs to change; this workflow has no host-specific data.
