---
name: sonar
description: Query and update data in the Sonar platform via its GraphQL
  API. Use whenever the user mentions Sonar, its dashboard, or asks to
  fetch, list, create, update, or delete projects, prompts, topics,
  competitors, runs, mentions, citations, project members, or
  per-prompt/per-competitor stats. Also use for any analytics question
  (mention/citation rates and trends, leaderboards, top cited domains or
  pages, citation distributions such as homepage vs other pages) — one
  `analytics` query answers these. Requires the SONAR_API_KEY
  environment variable.
---

# Sonar GraphQL API

## Endpoint and authentication

- Endpoint defaults to `https://sonar.toptal.com/api/graphql` (production,
  POST only). Override via `$SONAR_API_URL` for local dev
  (`http://localhost:3000/api/graphql`).
- Auth header: `Authorization: Bearer $SONAR_API_KEY`. API keys are
  prefixed `sonar_`; if you see a different prefix it's not a Sonar key.
- Always send `User-Agent: sonar-claude-plugin` so the Sonar team
  can observe plugin-driven traffic.
- If `$SONAR_API_KEY` is unset or empty, STOP and tell the user to
  generate a key on their Sonar profile page (Profile → API Keys) and
  add it to `~/.claude/settings.json` under `env`. Do not attempt
  unauthenticated calls.
- Per-key rate limit: 100 requests/minute. Pace yourself when fanning
  out across many entities; a 429 means back off, do not retry
  immediately.

## How to execute requests

Use Bash with curl. Always send the query as JSON, always use variables
for user-provided values — never interpolate them into the query string:

```bash
curl -s "${SONAR_API_URL:-https://sonar.toptal.com/api/graphql}" \
  -H "Authorization: Bearer $SONAR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "User-Agent: sonar-claude-plugin" \
  -H "X-Sonar-Session-Id: $SONAR_SESSION_ID" \
  -d '{"query": "query($n: Int!) { ... }", "variables": {"n": 5}}'
```

The `X-Sonar-Session-Id` header is required on every non-exempt call once
you have checked in (see "Session check-in / check-out" below). Omit it
only for the exempt operations (`me`, introspection, `checkInSession`).

GraphQL returns HTTP 200 even on errors — always inspect the response
body. There are two error surfaces (see "Result unions" below):
- **Top-level `errors[]` array** — schema validation, malformed cursors,
  unexpected server errors. A 401 means the key is invalid or revoked;
  direct the user to regenerate.
- **Typed errors as union members** — `NotFoundError`, `ForbiddenError`,
  `ValidationError`, `ConflictError`. These appear inside the result
  union for queries and mutations that declare them.

If a call returns `{ "message": "Unexpected error." }` with no further
detail, that's the masked-error response for a server-side problem
(missing DB migration, deployment lag, infrastructure issue). Do NOT
retry — surface the issue to the user and suggest they reach out to
Sonar support.

## First-use scope introspection

On the first Sonar call in a conversation, fetch the viewer's scope so
you know what the user's key can do before attempting anything:

```graphql
{
  me {
    user { id name email role }
    apiKey { id abilities scopeProjects }
  }
}
```

- `apiKey: null` means the request used a session cookie (GraphiQL).
- `apiKey.abilities`:
  - `null` — no abilities scope was set when the key was created. The key
    **inherits the session user's permissions** (an admin still gets
    `manage`). This is the common default for keys created without
    toggling abilities.
  - `[]` (empty list) — the key was **explicitly scoped to zero abilities**
    and will be denied **every operation**, including reads
    (`ForbiddenError`).
  - `["read"]` — read-only key. Reads work; mutations forbidden.
  - `["manage"]` or `["read", "manage"]` — full access, subject to the user's
    own rights and `scopeProjects`.
- `apiKey.scopeProjects` is `[String!]` (project IDs the key is
  restricted to) or `null` for unscoped. Queries against other projects
  return `ForbiddenError`.

## Session check-in / check-out (REQUIRED, enforced)

This is **structurally enforced**, not etiquette. For an API-key caller,
the server rejects every request with `SESSION_REQUIRED` (HTTP 403) until
you have opened a session. The only ungated operations are `me`, schema
introspection (`__schema` / `__type`), and `checkInSession` itself — so
you can do first-use scope introspection, then you must check in before
anything else.

Bracket each distinct task (a coherent goal, however many reads/writes it
takes) with a check-in / check-out pair:

**1. At the start of the task**, open a session with your intent:

```graphql
mutation CheckIn($input: CheckInSessionInput!) {
  checkInSession(input: $input) {
    __typename
    ... on MutationCheckInSessionSuccess { data { id status intent } }
    ... on ValidationError { message }
  }
}
```

```json
{ "input": { "intent": "user asked to audit prompt coverage for topic X" } }
```

`intent` follows the same rules as a mutation `intent`: a short, generic
1–2 sentence description of the task (≤ 2000 chars). Never a verbatim
paste of the user's message, never client names or secrets.

The session is at `checkInSession.data.id` (status `ACTIVE`).

**2. For the rest of the task**, send the returned `id` as a header on
**every** request — queries and mutations alike:

```
X-Sonar-Session-Id: <id from checkInSession>
```

**3. At the end of the task** (success, partial, or abandoned), check out
with a report:

```graphql
mutation CheckOut($input: CheckOutSessionInput!) {
  checkOutSession(input: $input) {
    __typename
    ... on MutationCheckOutSessionSuccess {
      data {
        id status
        summary {
          totalActions successCount failureCount
          byOperation { operation resourceType ok count }
        }
      }
    }
    ... on ConflictError { message }
    ... on NotFoundError { message }
    ... on ValidationError { message }
  }
}
```

```json
{
  "input": {
    "sessionId": "<id from checkInSession>",
    "report": "Listed prompts for topic X; updated 2 with stale mention rates after confirmation.",
    "deviations": "none"
  }
}
```

- `report`: what you actually did, in the same generic-summary style as
  `intent`.
- `deviations`: how the work differed from the stated `intent`. If nothing
  deviated, send `"none"` explicitly — don't omit the field.
- `summary` is computed by the server from the audit trail of writes you
  made during the session. You don't fill it in; it exists so reviewers can
  compare your report against reality.

**Rules:**

- Check in ONCE per distinct task, right after first-use scope
  introspection and before your first non-exempt call.
- After checkout the session id stops working — a new task needs a new
  check-in.
- NEVER skip checkout, even if the task failed or you abandoned it midway.
- `checkInSession` / `checkOutSession` do NOT take `dryRun` / `intent`
  args like other mutations.

## Verifying setup

A cheap smoke test confirms key and connectivity (`me` is exempt from the
session gate, so this works before check-in):

```graphql
{ me { user { id name email } } }
```

Run this first whenever the user says they just set things up. The
returned email makes the confirmation unambiguous.

## Result unions (typed errors)

Most top-level queries and every mutation are wrapped in a result union.
Always spread the success case AND the error cases — `__typename`
discriminates them:

```graphql
query Project($id: ID!) {
  project(id: $id) {
    __typename
    ... on QueryProjectSuccess { data { id name slug } }
    ... on NotFoundError { message }
    ... on ForbiddenError { message }
  }
}
```

The response data lives on the Success member's `data` field; the error
message on the error member. Validation errors that prevent the
operation from even parsing (bad cursor, malformed query) still come
back in the top-level `errors[]` array with
`extensions.code: BAD_USER_INPUT`.

## Pagination (Relay-style connections)

List fields use cursor connections:

```graphql
query PromptsPage($projectId: ID!, $after: String) {
  prompts(projectId: $projectId, first: 50, after: $after) {
    __typename
    ... on QueryPromptsSuccess {
      data {
        edges { node { id text } cursor }
        pageInfo { hasNextPage endCursor }
        totalCount
      }
    }
    ... on NotFoundError { message }
    ... on ForbiddenError { message }
    ... on ValidationError { message }
  }
}
```

- `first` defaults to 50, capped at 100.
- `after` is the opaque `endCursor` from the previous page.
- `totalCount` is the full count and is stable across pages.
- Some fields (`Run.mentions`, `Run.citations`) use specialized cursors;
  consult `references/queries.md` for the exact paging shape.

## Analytics: one query, never a fan-out

Every aggregate — rates, counts, shares, rankings, trends, distributions,
per-prompt or per-day stats — comes from `Query.analytics(projectId,
dateRange, measures, groupBy, filter, orderBy)`. It reads pre-aggregated
rollups, so "all citations for every prompt for every day of a month,
split homepage vs other pages" is ONE request
(`groupBy: [DATE, PAGE_KIND], measures: [CITATIONS]`), not one request per
prompt-day.

- NEVER page `runs` × `Run.citations` / `Run.mentions` to count, rank or
  distribute anything, and never sample prompts to work around request
  volume. Use runs only when the user needs raw answer text or individual
  citation URLs.
- `dateRange` is inclusive at both ends, in the project's timezone. The
  latest day can still be filling in — include `ANSWERS` to spot a short day.
- A `ValidationError` from `analytics` names the measures/dimensions that
  are valid for your grouping — fix the request from the message instead of
  guessing.
- Vocabulary, filters and copy-ready recipes: `references/queries.md` →
  "Analytics".

## Mutations: dryRun + intent (REQUIRED)

Mutations only — read operations execute freely. Every mutation accepts
two extra args alongside its `input` (or alongside its plain ID args
for `archivePrompt` / `deletePrompt` / `deleteTopic`):

- **`dryRun: Boolean = false`** — validates + computes impact (cascade
  counts, import plan) without writing. Returns the same Success shape;
  no audit row is written. Use this to preview before committing.
- **`intent: String`** (≤ 500 chars) — a SHORT, GENERIC description of
  WHY the change is happening. Stored in `audit_log.metadata.intent`
  and reviewable by Sonar admins. Treat it like a commit message:
  "user asked to delete expired prompts", NOT a verbatim paste of the
  user's prompt. Never include client names, internal project labels,
  secrets, or anything the user wouldn't want in a compliance review.

**Workflow for any mutation:**

1. Run with `dryRun: true, intent: "<generic 1-sentence summary>"`.
   Show the user the cascade or impact counts ("this will delete
   3 mentions and 2 citations").
2. Confirm with the user.
3. Re-run with `dryRun: false` and the same `intent`.

NEVER skip the dryRun preview, even if the user is being explicit —
confirmation is cheap and the cascade-count visibility is the point.

## Schema and examples

- `references/queries.md` — tested, copy-ready queries and mutations
  for common tasks. Prefer these over composing from scratch.
- For exact type, field, or argument names the cookbook doesn't cover,
  introspect the live schema (next section). Never guess field names.

## Schema discovery (introspection)

The endpoint serves standard GraphQL introspection to authenticated
callers, so what you see is always the deployed schema — there is no
bundled SDL copy that could go stale. Use small targeted probes; the
full `IntrospectionQuery` also works but returns hundreds of KB you
almost never need.

List every available operation:

```graphql
{
  __schema {
    queryType { fields { name description } }
    mutationType { fields { name description } }
  }
}
```

Describe one type — works for object types (`fields`), input types
(`inputFields`), enums (`enumValues`), and result unions
(`possibleTypes`):

```graphql
query($type: String!) {
  __type(name: $type) {
    kind
    name
    description
    fields {
      name
      description
      args {
        name
        defaultValue
        type { kind name ofType { kind name ofType { kind name ofType { kind name } } } }
      }
      type { kind name ofType { kind name ofType { kind name ofType { kind name } } } }
    }
    inputFields {
      name
      defaultValue
      type { kind name ofType { kind name ofType { kind name ofType { kind name } } } }
    }
    enumValues { name description }
    possibleTypes { name }
  }
}
```

Use this probe verbatim. The endpoint enforces a query depth limit of 8
that exempts `__schema`-rooted queries but NOT `__type`-rooted ones —
the shape above sits exactly at the limit, so do not nest it deeper or
wrap the `type` selections in a fragment (fragment spreads count as an
extra level). Three `ofType` levels unwrap every wrapper in this schema
(e.g. `[ID!]!`).

Typical flow for an unfamiliar operation: list operations → describe
the result type → follow `possibleTypes` to the Success member →
describe nested types as needed. Each probe is one cheap request.

## Conduct rules

- If a call returns `SESSION_REQUIRED` (in the top-level `errors[]` array,
  HTTP 403), you skipped check-in or your session was already checked out.
  This is NOT a permissions problem — call `checkInSession`, then retry
  with the `X-Sonar-Session-Id` header set. Don't tell the user their key
  lacks access.
- If a call returns `ForbiddenError`, the user's key lacks that scope —
  say so, do not retry. Check `apiKey.scopeProjects` and `abilities`
  from the first-use introspection to know in advance.
- If the user asks about data in a project not in `apiKey.scopeProjects`,
  tell them their key is scoped and they need to either widen the
  scope or use a different key.
- Never log or display the value of `$SONAR_API_KEY` to the user
  ("your key is …"). Keep it implicit.
