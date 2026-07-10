# Opportunity to Deal Room (Hypothetical Example)

File: [`workflows/opportunity-to-deal-room.json`](../workflows/opportunity-to-deal-room.json)

**This one's different from the rest of the repo.** Everything else here
automates homelab maintenance; this workflow is a **hypothetical,
genericized example** of a common sales-engineering automation pattern —
included because the error-handling techniques it demonstrates (signature
verification, tolerating partial third-party lookup failures, and
re-synchronizing parallel branches before a final step) are broadly useful
n8n patterns worth having a documented reference for, independent of any
specific company's tooling. It was never wired up against real production
systems — no CRM, chat workspace, or "system of record" API named or implied
here is a real integration; every credential, hostname, and field name is a
placeholder.

## What it does

A Slack slash command (e.g. `/new-opp <link to a CRM Opportunity>`) stands
up a dedicated "deal room" for a sales opportunity:

1. Verifies the request actually came from Slack (HMAC signature check).
2. Pulls the opportunity record from a CRM (Salesforce, in this example) —
   name, account, stage, and the email addresses of the people who should be
   in the room (account exec, sales manager, SE manager, primary/secondary
   SE).
3. Creates a Slack channel named after the account + opportunity.
4. Resolves each role's email to a Slack user ID and invites everyone it
   could resolve, plus the person who ran the slash command — without
   failing the whole run if one or two lookups don't match a Slack account.
5. Creates a Drive folder for the deal, and cross-links it and the CRM
   record into the channel as Slack bookmarks.
6. Notifies some downstream "system of record" that the channel now exists
   for this opportunity (a placeholder for whatever your org would actually
   want to know that — a CRM custom object, a wiki page, an internal
   registry, or nothing at all).
7. DMs the requester a confirmation, including a warning listing anyone who
   couldn't be auto-invited so they can be added by hand.

## Nodes

| Node | Type | What it does |
|---|---|---|
| Slack Slash Command | `webhook` | Receives the slash command payload. Raw Body enabled so the signature check below can hash the exact bytes Slack sent. |
| Verify Slack Signature | `code` | Recomputes Slack's HMAC signature from the raw body + timestamp and compares it (`crypto.timingSafeEqual`) against the `X-Slack-Signature` header, within a 5-minute window. |
| Signature Valid? | `if` | Branches on the result. |
| Notify Signature Failure | `httpRequest` | On failure, posts an ephemeral error back via Slack's `response_url` (the initial webhook ack was already sent, so this is a separate follow-up message, not the HTTP response itself). |
| Parse SFDC Opportunity ID | `set` | Regexes the 15/18-char Salesforce ID out of the opportunity link in the command text. |
| Get Salesforce Opportunity | `httpRequest` | SOQL query for the opportunity's name, account, stage, and owner/role emails via custom lookup relationships. |
| Extract Opportunity Fields | `set` | Pulls the fields out of SOQL's `records[0]` wrapper into flat, named fields. |
| Derive Channel Variables | `code` | Slugifies account + opportunity name into a Slack-safe channel name/topic. |
| Create Slack Channel | `slack` | Creates the channel. |
| Build Email List | `code` | Fans the filtered role emails out into one item per email. |
| Look up a user by email | `slack` | `users.lookupByEmail`, one call per item, `onError: continueRegularOutput` so one bad email doesn't kill the run. |
| Only Successful Lookups | `filter` | Splits lookups into passed (has `id`) vs. failed — **two outputs**, both used below. |
| Aggregate Slack User IDs | `aggregate` | (pass branch) Collects resolved `id`s into one `userIds` array. |
| Invite Users to Channel | `slack` | Invites `userIds` plus the requesting user (deduped), in one call. |
| Recover Failed Email | `set` | (discard branch) `continueRegularOutput` replaces a failed item's `json` entirely with `{error: ...}` — this re-sets `email` from `pairedItem` back to **Build Email List** so there's something to aggregate. |
| Aggregate Failed Lookups | `aggregate` | Collects the recovered `email`s that failed lookup into `unresolvedEmails`. |
| Create Drive Folder | `googleDrive` | Creates a folder for the deal room. |
| Add Bookmark - Drive | `httpRequest` | `bookmarks.add`, linking the Drive folder into the channel. |
| Add Bookmark - SFDC | `httpRequest` | `bookmarks.add`, linking the CRM record into the channel. |
| Register Channel with Deal Registry | `httpRequest` | Placeholder callback to whatever downstream system should know this deal room exists. |
| Sync Before Confirmation | `merge` | Reconverges the invite-path and failed-lookup-path branches before the final message — see below. |
| Send Confirmation | `slack` | DMs the requester a summary, with a warning line if `unresolvedEmails` is non-empty. |

## The error-handling patterns this demonstrates

This workflow exists mainly to document three n8n patterns that came up
while building the real thing this is modeled on, each non-obvious enough to
be worth a written note:

1. **Signature verification before doing anything else.** Any workflow
   triggered by an unauthenticated webhook that fans out to real actions
   (channel creation, invites, external API calls) should verify the
   request's authenticity as the very first step and short-circuit on
   failure — see `Verify Slack Signature` → `Signature Valid?`.

2. **Tolerate partial failure in a per-item external lookup instead of
   failing the whole execution.** `Look up a user by email` uses
   `onError: continueRegularOutput` so one unmatched email doesn't abort
   the run. But that setting replaces the failed item's `json` with
   `{error: ...}`, **discarding the original input fields** — if you need
   that data downstream (here, the email address itself, to report which
   ones failed), recover it explicitly via `pairedItem` back to an ancestor
   node (`Recover Failed Email`). Don't assume `continueRegularOutput`
   preserves your input on the error path; check the actual item shape.

3. **Reconverge sibling branches with a Merge node before a step that reads
   from both.** After `Only Successful Lookups` splits into a pass branch
   (→ invite users) and a discard branch (→ collect failed emails), both
   eventually need to have finished before `Send Confirmation` can safely
   read `Aggregate Failed Lookups`'s output. Referencing one branch's node
   via `$('NodeName')` from code running on the *other* sibling branch is
   **not** enough — sibling branches aren't ordered relative to each other,
   so a naive version of this workflow can send the confirmation message
   before the failed-lookups branch has actually produced its output,
   silently dropping the warning. A Merge node (`combineByPosition`,
   `includeUnpaired: true`) with one input per branch only fires once every
   connected input has resolved — including a branch that produced zero
   items, which is what happens on the (normal) all-succeeded path. This is
   the general pattern for "wait for every branch of an IF/Filter split to
   finish" in n8n; a `$()` reference across branches is not a substitute for
   it.

A fourth, smaller gotcha worth knowing about even though this workflow
doesn't hit it: a node fed *only* by a branch that receives zero items in a
given run doesn't execute at all that run — it's not "executed with zero
items." Any `$('NodeName')...` reference to such a node needs an
`.isExecuted` guard (see `Send Confirmation`'s expression) before calling
`.first()` / `.item` on it, or n8n throws `"Node 'X' hasn't been executed"`.

## Adapting

This is a **pattern to build from, not a drop-in workflow** — nothing here
points at real infrastructure.

1. **Recreate credentials.** Every `REPLACE_ME` credential ID needs a real
   credential of that type: Slack (Bot Token — scopes `channels:manage`,
   `chat:write`, `users:read`, `bookmarks:write`), Salesforce OAuth2 (or swap
   for your actual CRM's API), Google Drive OAuth2, and an HTTP Header Auth
   credential for whatever "Deal Registry" ends up being (or delete that
   node if you don't have a downstream system to notify).
2. **Set `SLACK_SIGNING_SECRET`** as an n8n environment variable, from your
   Slack app's Basic Information page.
3. **Fix the CRM query.** `PLACEHOLDER_SFDC_INSTANCE_URL` and the four
   `PLACEHOLDER_*_RELATIONSHIP_NAME` tokens on **Get Salesforce Opportunity**
   are custom lookup-relationship field names specific to *some* Salesforce
   org, not standard fields — replace them with your own org's schema, or
   swap the whole node for a call to a different CRM entirely. The role set
   (account exec / sales manager / SE manager / primary+secondary SE) is
   just an example team shape — adjust to whatever roles your process
   actually needs represented in the room.
4. **Set Drive IDs.** `PLACEHOLDER_SHARED_DRIVE_ID` / `PLACEHOLDER_TEMPLATE_FOLDER_ID`
   on **Create Drive Folder** need to point at a real shared drive/folder,
   or drop the Drive step entirely if you don't need it.
5. **Point "Register Channel with Deal Registry" at something real, or
   delete it.** It's a stand-in for "notify whatever system should know a
   deal room now exists" — a CRM custom object, an internal wiki, a
   database — there's no default assumption about what that should be.
6. **Register the slash command.** Create a Slack app, add a slash command
   (e.g. `/new-opp`) pointing at this workflow's webhook URL, and enable
   **Raw Body** on the Webhook node's options (required for the signature
   check).
7. This workflow has no Error Workflow setting to relink — errors inside
   individual integration calls (Salesforce, Drive, the registry callback)
   aren't caught here and would fail the execution outright; wire in
   [Error Handler](error-handler.md) (or your own) if you build on this for
   real.
