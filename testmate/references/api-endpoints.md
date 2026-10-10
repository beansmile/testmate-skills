# TestMate CLI / API Reference

> TestMate（Beansmile 自托管、兼容 Make This Better）沿用上游 CLI `@makethisbetter/cli`
> （二进制 `makethisbetter`）与 MCP `@makethisbetter/mcp`。下文命令与端点与上游一致，
> 仅 base URL 不同：`https://testmate.beansmile.ai/api/v1`。
> TestMate reuses the upstream CLI and MCP unchanged; only the base URL differs.

## Install

```bash
npm install -g @makethisbetter/cli
```

The CLI binary is `makethisbetter` — a precompiled Go binary; the npm package
is a thin installer (Node is only needed to run npm itself). Alternative:
download the binary for your platform from
https://github.com/makethisbetter/cli/releases.

## Config

Saved at `~/.makethisbetter/config.json` with mode `600`:

```json
{
  "api_token": "token_...",
  "api_url": "https://testmate.beansmile.ai/api/v1",
  "account_id": "acct_...",
  "user_email": "dev@example.com"
}
```

不要 `cat` 该文件。用 `makethisbetter info` 查看脱敏状态。

This file contains a bearer token. Do not `cat` it into agent context or logs;
use `makethisbetter info` for masked local status and an authenticated command
to verify the credential.

## Commands

### Login（仅 Token / token only）

```bash
makethisbetter login --token <token> --api-url https://testmate.beansmile.ai/api/v1
makethisbetter login --token "$TESTMATE_API_TOKEN" --api-url https://testmate.beansmile.ai/api/v1
makethisbetter login --account-id acct_xxx
```

TestMate 不支持邮箱 OTP（`--email` / `--send-only` / `--otp` 不可用）。Token 在
`https://testmate.beansmile.ai/settings/tokens` 创建。
TestMate has no email OTP; create tokens in the dashboard and save them with `--token`.

### Feedback list

```bash
makethisbetter feedback list --project acme
makethisbetter feedback list --project acme --status received
makethisbetter feedback list --project acme --label Safari
makethisbetter feedback list --project acme --priority high
makethisbetter feedback list --project acme --sort priority
makethisbetter feedback list --project acme --archived
makethisbetter feedback list --project acme --json
```

Filters: `--status`, `--label`, `--priority`. `--project` is required. AI selects
labels from the system-managed pool; the CLI only reads and filters them.
`--archived` selects the archived collection and cannot be combined with
`--status`.
Sort: `priority`, `created`, `updated`.

### Feedback show

```bash
makethisbetter feedback show <handle/FB-n> --md    # read first: original context and attachment links
makethisbetter feedback show acme/FB-1 --json      # only when structured fields are needed
```

Displays full feedback detail: description, AI analysis, reporter, page URL,
console errors, target element, and timeline. `--md` prints the
server-rendered markdown and is the original-context source of truth. Start
there. Use `--json` only when Markdown cannot answer a question that needs
structured data, such as a target selector or coordinates, individual
annotations/breadcrumbs, raw telemetry, an attachment URL absent from
Markdown, or batch processing. JSON can omit raw attachment addresses; it
also includes Markdown in its `markdown` field.
Show transparently checks the archived collection after an active lookup
misses.

### Project list / show / create / update

```bash
makethisbetter project list [--json]
makethisbetter project show <handle> [--json]
makethisbetter project create <name> --handle <handle> --domain example.com [--ai-context "..."] [--json]
makethisbetter project update <handle> [--name <name>] [--domain <domain>] [--ai-context "..."] [--json]
```

`show` returns `api_key` (the widget `projectKey`), `board_url`, and
`ai_context`; `signing_secret` is included only when the logged-in user is an
account admin. `create` and `update` require account admin (403 otherwise);
`create` returns the new project including `api_key` and `signing_secret`.
`update` sends only the flags you set — unset flags leave the project
unchanged, and `--ai-context ""` clears the field.

Treat project create/show/update output as sensitive for account admins (do not use MCP `project_show` / `project_create`; they leak `signing_secret`). Normal
setup should allowlist the fields it prints and must not expose
`signing_secret` in terminal logs or chat.

### Feedback lifecycle

```bash
makethisbetter feedback pick <handle/FB-n>                        # status -> in_progress
makethisbetter feedback pick <handle/FB-n> --takeover             # replace the current assignee
makethisbetter feedback respond <handle/FB-n> --body-file <path>  # queued notice -> closed(responded)
makethisbetter feedback archive <handle/FB-n>                     # hide one Unclaimed Feedback
makethisbetter feedback restore <handle/FB-n>                     # return one archived Feedback
makethisbetter feedback ready <handle/FB-n> --summary "<what changed>" # status -> pending_release
makethisbetter feedback release --project <handle> --through <deployed-sha> # scoped deployed matches -> closed(shipped)
makethisbetter feedback decline <handle/FB-n>                     # closed(not_planned)
makethisbetter feedback duplicate <handle/FB-n> --canonical <handle/FB-m> # closed(duplicate)
makethisbetter feedback reopen <handle/FB-n>                      # closed -> received
```

Commands accept `--json` for machine-readable output. `ready --summary` is
required and checks the current reachable Git history for an exact Feedback
trailer. `release` requires complete Git history and runs only after production
deployment. `--canonical` must be fully qualified and name Feedback in the same
Project. `pick` claims Feedback for the authenticated user.
It returns `409 Conflict` without changing the Feedback when another team
member is assigned; use `--takeover` only when the user explicitly asks to
replace that assignee. Status values are
`received`, `in_progress`, `pending_release`, `closed`; `shipped`, `responded`,
`not_planned`, and `duplicate` are `close_reason` values on `closed`.

`respond` requires a UTF-8 body file or `-` for stdin. Agent workflows must use
a real file path containing final user-supplied and confirmed text; Agents must
not autonomously generate and send a response. An omitted subject uses the
Reporter Language and falls back to English when unsupported. Archive only
accepts Unclaimed Feedback. Respond, Archive, and Restore are retry-safe and do
not support bulk operation. Their JSON output exposes state and delivery
metadata but never echoes the response body.

## API Endpoints

Base URL: `https://testmate.beansmile.ai/api/v1`（CLI 从配置中的 `api_url` 读取；widget 必须显式传 `apiUrl`）

### List / show / create projects

```http
GET /api/v1/projects
Authorization: Bearer token_...
```

Returns `[{ "id": "project_...", "name": "...", "domain": null, "feedback_visibility": "...", "feedbacks_count": 0, ... }]`.

```http
GET /api/v1/projects/:id
Authorization: Bearer token_...
```

Adds `api_key` (the widget `projectKey`), `board_url`, `ai_context`,
`enforce_identity_verification`, and — for account admins only —
`signing_secret`.

```http
POST /api/v1/projects
Authorization: Bearer token_...
Content-Type: application/json

{ "project": { "name": "My App", "handle": "my-app", "domain": "example.com" } }
```

Requires account admin (`403` otherwise). Response `201` matches the show
shape including `api_key` and `signing_secret`. `ai_context` may be included
in the create body.

```http
PATCH /api/v1/projects/:handle
Authorization: Bearer token_...
Content-Type: application/json

{ "project": { "ai_context": "B2B invoicing app for accountants..." } }
```

Requires account admin (`403` otherwise). Accepts `name`, `domain`, and
`ai_context`; omitted keys stay unchanged. Response matches the show shape.

### List feedback

```http
GET /api/v1/projects/acme/feedbacks?status=received
Authorization: Bearer token_...
```

Query params: `status`, `label`, `priority`, `limit` (1-200), and `account_id`.
`feedback_type` remains a deprecated read-only alias for `label`. The response
exposes `labels` as an array of AI-managed Project Label names.

### List / show archived feedback

```http
GET /api/v1/projects/acme/archived_feedbacks?label=Bug&priority=low
GET /api/v1/projects/acme/archived_feedbacks/1
Authorization: Bearer token_...
```

The collection accepts `label`, `priority`, and `limit`, but no workflow
`status`. Feedback representations expose `archived_at`. Active show and
archived show are separate API endpoints; CLI/MCP detail perform the fallback.

### Show feedback

```http
GET /api/v1/projects/acme/feedbacks/1
Authorization: Bearer token_...
```

### Update feedback

```http
PATCH /api/v1/projects/acme/feedbacks/1
Authorization: Bearer token_...
Content-Type: application/json

{ "feedback": { "status": "in_progress" } }
```

Accepted fields: `status`, `priority`, `close_reason`, `pr_url`,
`resolution_summary`, `canonical_feedback_id`, `takeover`, and
`ai_structured_summary`. Updating to `in_progress` assigns the Feedback to the
authenticated user; `assignee_id` is not accepted. If another user is already
assigned, the API returns `409 Conflict` and leaves the Feedback unchanged.
Send `takeover: true` with `status: "in_progress"` only to explicitly replace
that assignee. Feedback responses include `assignee` as `{ "id": "user_...",
"name": "..." }` or `null`.
`canonical_feedback_id` uses a fully qualified same-Project reference such as
`acme/FB-1`. Project Labels are read-only through this API. Generic PATCH cannot
write `pending_release`, `closed(shipped)`, or `closed(responded)`. Sending only
`status: received` for closed Feedback invokes reopen (Account Owners/Admins, Active Pro Members, and assigned Team Members).

### Respond and close

```http
POST /api/v1/projects/acme/feedbacks/1/response
Authorization: Bearer token_...
Content-Type: application/json

{ "feedback_response": { "body": "Final user-confirmed text.", "subject": "Optional subject" } }
```

Requires active Feedback in `received` or `in_progress` with a Reporter email.
Claimed Feedback can be responded to directly; `pending_release` and other
closed outcomes are rejected. The optional subject defaults in the Reporter
Language. The response contains the closed Feedback
and persisted delivery metadata; it does not echo the body. Repeating the same
operation after `closed(responded)` returns the existing outcome.

### Archive / restore feedback

```http
POST /api/v1/projects/acme/feedbacks/1/archive
DELETE /api/v1/projects/acme/feedbacks/1/archive
Authorization: Bearer token_...
```

Archive accepts only Unclaimed Feedback. Restore returns archived Feedback to
active views without changing its `received` status. Both are idempotent.

### Mark feedback ready

```http
POST /api/v2/projects/acme/feedbacks/1/readiness
Authorization: Bearer token_...
Content-Type: application/json

{ "feedback_readiness": { "resolution_summary": "Fixed Safari export downloads." } }
```

Requires `in_progress`; changes status to `pending_release` without notifying
the Reporter. Git trailer validation belongs to the CLI.

### Release deployed feedback

```http
POST /api/v2/projects/acme/feedbacks/1/release
Authorization: Bearer token_...
Content-Type: application/json

{
  "feedback_release": {
    "trailer_committed_at": "2026-08-01T12:00:00Z"
  }
}
```

Requires `pending_release` with a saved Resolution Summary. If the trailer time
is not later than the latest reopen event, the API returns `409` and leaves the
Feedback unchanged.

## Error Codes

- `401`: invalid or expired token.
- `404`: feedback not found.
- `409`: Feedback is already assigned, or release evidence predates the latest reopen.
- `422`: invalid parameters.
- `429`: rate limited.
