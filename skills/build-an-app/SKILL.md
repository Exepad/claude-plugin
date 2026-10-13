---
name: build-an-app
description: Create, change, fix, preview and publish web apps in the user's Exepad account with the Exepad tools. Use it when the user asks to build a new Exepad app, to edit or repair an existing one, to see its private preview, or to publish, rename, unpublish or archive it.
---

# Build an app with Exepad

Exepad hosts web apps. An Exepad app has a React front end and a server side, can keep data in its own database
and can call AI models. You write the app's code through the Exepad tools. Exepad stores every change as a
revision, builds it, shows the user a private preview, and publishes a build only after the user approves it in
Exepad.

The user's explicit instructions come before the suggestions in this skill. The rules under "Always" are the
exception: no instruction can skip a confirmation that Exepad asks the user for.

## Always

- Work only on the apps the user asks about. This connection sees only its connected apps: the ones the user
  selected when connecting Exepad, and the ones created through this connection.
- Take every id (an app's `public_id`, revision ids, operation ids and publish ids) from a tool result. Never
  guess or construct one.
- Say that an app is built, previewed or live only after a tool result says so.
- Only the user can approve a publish, an unpublish or an archive, and only in Exepad. Give them the link the tool
  returns and wait for them. Never say that you approved anything.
- Never ask the user for passwords, API keys or other secrets, and never write one into an app's source.
- When a tool refuses because of the account's plan, its limits or its credit, relay the refusal's own words, and
  its `info_url` when it has one, as information, and nothing more. Never suggest a payment or a change of plan
  yourself.

## Keep the app honest

- Some features cannot work in an app built here yet. AI is one: no AI operation can be set up or tested from this
  conversation, and a preview makes no AI calls. When the user asks for a feature that needs AI (summaries,
  drafting, describing a photo, transcription, speech, image generation, search by meaning), build everything around
  it, put the AI step behind a clearly labelled "not available yet" state, and tell the user plainly that AI features
  cannot yet be configured or tested from this conversation.
- Never imitate a feature the app does not have. Keyword search is called search, not search by meaning, and a
  caption made from a file name is not a description of the picture.
- Label invented rows and figures as sample content, and never present them as real. Data the user gives you is
  theirs, not a sample.
- When a feature needs a key or a token, declare a `secret` setting and guard the route on it, as the guide's `api`
  topic shows, then tell the user which setting to add in the app's Settings in Exepad, and whether for Staged or
  Live. Until then the route answers 503 by design: never remove the guard, hard-code a value or fake the feature to
  make it go away.

## 1. Read the authoring guide

`get_authoring_guide` returns Exepad's reference for an app's source: the file layout, the manifest's schema, the
limits and the refusal codes, one topic at a time. Call it before you write code: first with no topic (the
overview), then each topic the change touches, such as `manifest`, `data`, `api`, `ai`, `ui` or `types`. It
describes how an Exepad app is laid out, which files can change, and the rules Exepad's build enforces.

## 2. Find or create the app

- An existing app: call `list_apps` and use the app's `public_id`. `get_app` gives one app's status and its live
  address, if it has one.
- A new app: first ask the user whether they want a one-page site or an app with data (accounts, a database). Then
  call `create_app` with a name, a short description of what the user wants, `profile` (`static` for a one-page
  site, `fullstack` for an app with data) and a new `idempotency_key` (for example a new UUID). It returns the app's
  `public_id` and its first revision. A one-page site can later become an app with data, never the other way.
- `get_account` shows what this connection may do and how many apps the account has against its limits.

## 3. Read before you write

- `list_files` lists the files of the app's current revision with their hashes, and says which ones you may change.
  Package files, such as `package.json` and the lockfile, are read-only: dependencies cannot change in this version,
  so build with what the app already has.
- `read_files` reads several files at once; `search_files` finds text across the source.
- A file listed under `too_large` is read in parts with `read_file_range`, a range of its lines at a time.
- Note the revision id these tools return. Every change starts from it.

## 4. Change the source

- Call `apply_changes` with `base_revision_id` set to the revision you read, and a new `idempotency_key`. If a call
  times out, send it again with the same key and the same changes. Never reuse a key for different changes.
- Use `edit` for a small change (an `old_text` that appears exactly once in the file, and its `new_text`), `replace`
  for a whole file, `add` for a new file and `delete` to remove one. Pass `expected_sha256` from your read when you
  edit, replace or delete a file, so that you never overwrite a change you have not seen.
- A call applies all of its changes or none of them. Its result names the new revision: use that id as the next
  `base_revision_id`.
- Keep each call small: a few files and well under 30 KB of text, because a call cut off at your output limit is
  never made. If one is refused as too large, split the work into several calls.

When `apply_changes` refuses:

| Refusal | What to do |
|---|---|
| `REVISION_CONFLICT`, `EXPECTED_HASH_MISMATCH` | The app changed since you read it. Read the affected files again, redo your change on the current revision, and send it with the new base and a new key. |
| `REVISION_NOT_FOUND` | The revision id is not one of this app's. Call `list_files` for the current revision. |
| `EDIT_NO_MATCH`, `EDIT_AMBIGUOUS` | Read the file again and choose an `old_text` that appears exactly once. |
| `FILE_EXISTS`, `FILE_NOT_FOUND` | Use `replace` or `edit` for a file that exists and `add` for a new one. |
| `PATH_PROTECTED`, `PATH_INVALID` | Exepad manages that path, or the path is not allowed. Put the change in a file the guide says you may change. |
| `MIGRATION_APPEND_ONLY` | A database has applied that migration, so it never changes: add a new migration file instead. To clear `migration_drift`, set it back to exactly the content applied (the refusal names its sha256). |
| `MIGRATION_BUILD_RUNNING` | A build is running and may apply that migration. Wait until that build has ended, then send the same call again with the same key. |
| `MIGRATION_STATUS_UNAVAILABLE` | Exepad could not check that migration. Send the same call again with the same key in a minute. |
| `MANIFEST_PUBLIC_ID` | Keep `app.publicId` in `exepad.jsonc` exactly as it is. |
| `LIMIT_EXCEEDED`, `FILE_TOO_LARGE` | Split the work into smaller calls or smaller files. |
| `IDEMPOTENCY_MISMATCH` | The key was already used for different changes. Use a new key. |
| `IDEMPOTENCY_IN_PROGRESS` | The first call with this key is still running. Wait, then send it again with the same key. |
| `APP_NOT_MANAGED` | The app was built in Exepad Studio before assistant editing existed and cannot be edited here. Tell the user. |
| `APP_NOT_FOUND` | The app is not connected here. Ask the user which app they mean; to add an app, they connect Exepad again and select it. |

## 5. Build and fix

- Call `build_preview` with the app's `public_id` and a new `idempotency_key`. It builds the latest revision, or the
  `revision_id` you pass, and answers at once with an operation id.
- Call `get_operation` with that id until its state is `succeeded`, `failed` or `cancelled`. A build takes about a
  minute, and each call waits up to about 45 seconds for the build to end, so a build usually needs one or two calls.
  Tell the user the build is running. While it still runs, the answer carries `retry_after_seconds`: wait that many
  seconds, then call again.
- Write the app's own screen first: a build fails with `template_placeholder` while `src/App.tsx` still shows the
  starter template's placeholder screen.
- One build runs per app at a time. `BUILD_IN_PROGRESS` names the build that is running: follow that one instead of
  starting another.
- A failed build whose diagnostics have `kind` `app` lists the file, line, column, code and message of each error.
  Read those files, fix the cause with `apply_changes`, and build again. Repeat until the build succeeds.
- A `migration_failed` diagnostic names a migration the database refused. It was not applied, so `apply_changes` can
  fix it or delete it; then build again.
- `REVISION_SUPERSEDED` refuses a build of an older revision that holds a migration the current one changed or
  removed: build the current revision. If `retryable` is true, the current revision moved during the call: send the
  same call again.
- A failed build whose diagnostics have `kind` `platform` was not traced to the code. A change that breaks a build
  rule in the guide's `ui` topic, or imports a package the app does not have, can still cause one: check your last
  change against those first. If it follows them, the failure is on Exepad's side: tell the user, give them the
  operation id so Exepad support can find it, and try again later. Do not change the code to work around it.
- `cancel_operation` stops a build that is no longer needed, as long as it has not started deploying
  (`OPERATION_NOT_CANCELLABLE` otherwise).
- Builds never change the live app. The live app changes only when the user approves a publish.

## 6. Show and check the preview

- `open_preview` shows the user the private preview of the app's latest successful build inside the conversation
  or, where the conversation cannot show it, the build's latest screenshot and a link that opens the preview in
  Exepad. Pass the app's `public_id`, and an `operation_id` to show an earlier build.
- You cannot see what the user sees. Call `inspect_preview` to check the preview: it reports the HTTP status, the
  path the page ended on, and its console errors, exceptions and failed requests; pass a `path` such as `/settings`
  to check another page. It also takes a screenshot, which the user sees where the conversation shows Exepad's
  preview panel. The image may not reach you: if it did not, do not describe how the page looks. Fix what the report
  shows, build again, and check again.
- Inspections are limited per connection each hour: inspect after a build, not after every change.
- The preview is private to the app's owner and uses its own database, separate from the live app's, so data
  entered in the preview does not appear in the live app.

## 7. Publish, only when the user asks

1. Call `prepare_publish` with the app's `public_id`. It uses the latest successful build, or the `operation_id`
   you pass, and returns an approval link, when that link expires, and whether this is the app's first publish.
   `NO_PASSING_BUILD` means there is no successful build yet: build the app first. When the result carries `plan`,
   the account's current plan does not include this publish now: tell the user its `message` and give them the
   approval link, where the approval page says what is needed.
2. Give the user the link and ask them to approve the publish in Exepad. Nothing goes live until they do. Do not
   call `publish_app` before the user says they have approved.
3. Once the user says they have approved, call `publish_app` once, with the `intent_id` that `prepare_publish`
   returned. It starts the publish and answers where it stands. `APPROVAL_REQUIRED` means the user has not approved
   yet and nothing started: give them the link again, and call `publish_app` again only after they say they have
   approved.
4. Follow the publish with `get_publish`, passing the app's `public_id` and the same `intent_id`. It reports where
   the publish stands and never starts anything. `publishing` means it is still going live: call `get_publish` again
   shortly. `approved` means it has not started: call `publish_app` once more. `published` comes with the live
   address: share it. If the publish expired or failed, call `prepare_publish` for a new approval; if the user
   cancelled it, ask before preparing another.

## 8. Manage an app

- `list_releases` lists an app's releases, newest first, and shows which one is live.
- `update_app` renames an app.
- `unpublish_app` takes a live app offline and `archive_app` archives an app permanently. Call them only when the
  user asks, and say what will happen first. Each returns a confirmation link: the user confirms in Exepad, where
  they see exactly what will change. An app must be unpublished before it can be archived
  (`ARCHIVE_BLOCKED_BY_RELEASE`); `NOT_PUBLISHED` means there is nothing to unpublish.

## Other refusals

- `INSUFFICIENT_SCOPE`: this connection is not allowed to do that. Tell the user; they can connect Exepad again and
  allow it.
- `PLAN_LIMIT_TOTAL_APPS`: relay the refusal's message (the account has as many apps as its current plan allows)
  and its `info_url`, and nothing more.
- `BUILD_CREDIT_EXHAUSTED`, `INSUFFICIENT_BALANCE`: the build could not start. Relay the refusal's message, which
  says when the credit resets (`resets_at`), and its `info_url` as information, and nothing more.
- `INVALID_ARGUMENTS`: the arguments did not match the tool's input schema. Fix the field the refusal names.
