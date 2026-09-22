# Sonar GraphQL — Query Cookbook

Tested, copy-ready operations against the live Sonar GraphQL API. Every example in this file has been round-tripped against a real dev server with the v2 schema.

## Conventions

- All curl examples assume `SONAR_API_KEY` is set per `SKILL.md`. Endpoint defaults to `https://sonar.toptal.com/api/graphql`; override via `$SONAR_API_URL` for staging or localhost.
- Placeholders are wrapped in `<angle-brackets>` (e.g. `<project-id>`).
- Sample responses show the shape and field types; specific values are illustrative.
- Mutations always follow **dryRun → confirm → real call** per `SKILL.md`.

## Cheat sheet

| Pattern | Where |
|---|---|
| Any aggregate (rates, counts, shares, rankings, trends, distributions) | `Query.analytics` — see [Analytics](#analytics--queryanalytics) |
| First-use scope introspection | `me` query |
| Session check-in / check-out | `checkInSession` / `checkOutSession` mutations (REQUIRED, enforced) |
| Result union for queries | `... on Query<Verb>Success { data { … } }` plus error spreads |
| Result union for mutations | `... on Mutation<Verb>Success { data { … } }` plus error spreads |
| Cursor pagination | `edges { node cursor }`, `pageInfo { hasNextPage endCursor }`, `totalCount` |
| Default page size | 50 (max 100); pass `first` to override |
| Required mutation args | `dryRun: Boolean = false` + `intent: String` (except `checkInSession` / `checkOutSession`) |

### Valid `source` values

Many queries accept an optional `source: String` filter (analytics, stats,
`Query.runs`, etc.). The valid set is small and fixed:

| Value | LLM provider |
|---|---|
| `chatgpt` | OpenAI ChatGPT |
| `google_ai_mode` | Google Search's AI Mode |
| `gemini` | Google Gemini |

Omitting the arg means "all sources" (server-side default). Passing an
unknown value will silently match nothing in analytics aggregates — do
not guess values like `"openai"` or `"gpt-4"`; use the literals above.

### Bounded vs paginated — not every list takes `first` / `after`

Naturally bounded lists return a plain `[T!]` and **do not** accept pagination args.
Reflexively reaching for `(first: N)` on these returns "Unknown argument first" — use the naked field.

| Bounded (no pagination) | Paginated (connection) |
|---|---|
| `Query.projects` | `Query.prompts`, `Query.competitors`, `Query.runs`, `Query.users` |
| `Project.topics` | `Project.prompts`, `Project.competitors` |
| `Project.members` | `Prompt.runs`, `Run.mentions`, `Run.citations` |
| `Prompt.topics` | `Query.analytics` (offset cursor, `first` ≤ 1000, 20000-group cap) |

---

## Authentication & introspection

### `me` — viewer + apiKey scope (run first in every conversation)

```graphql
{
  me {
    user { id name email role }
    apiKey { id abilities scopeProjects }
  }
}
```

Sample response:

```json
{
  "data": {
    "me": {
      "user": { "id": "user_abc", "name": "Alice", "email": "alice@example.com", "role": "admin" },
      "apiKey": {
        "id": "key_xyz",
        "abilities": null,
        "scopeProjects": null
      }
    }
  }
}
```

`apiKey: null` → request used a session cookie (GraphiQL). `apiKey.abilities` is now nullable and distinguishes three cases: `null` means no abilities scope was set at creation — the key inherits the user's permissions (an admin user still gets manage); `[]` means the key was explicitly scoped to zero abilities and will be denied every operation; a non-empty list narrows to exactly those abilities (`["read"]` is read-only, `["manage"]` implies `read`). `apiKey.scopeProjects`: list of project IDs the key is restricted to, or `null` for unscoped.

### Cheap smoke test (after install)

```graphql
{ me { user { id name email } } }
```

Returned `email` makes the confirmation unambiguous.

### Schema discovery (introspection)

The schema is not bundled with the plugin — introspect the live
endpoint when this cookbook doesn't cover a field. `SKILL.md` has the
two canonical probes (list operations, describe a type) and the
depth-limit rules for composing your own; use those verbatim rather
than improvising deep introspection queries.

```bash
curl -s "${SONAR_API_URL:-https://sonar.toptal.com/api/graphql}" \
  -H "Authorization: Bearer $SONAR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "User-Agent: sonar-claude-plugin" \
  -d '{"query": "{ __schema { queryType { fields { name description } } mutationType { fields { name description } } } }"}'
```

---

## Session check-in / check-out (REQUIRED, enforced)

An API-key caller must open a session before any non-exempt call — the
server returns `SESSION_REQUIRED` (403) otherwise. Only `me`, introspection,
and `checkInSession` are exempt. Send the returned `id` as the
`X-Sonar-Session-Id` header on every subsequent request. See `SKILL.md` for
the full workflow and `intent`/`report` content rules.

### checkInSession — open a session

```graphql
mutation CheckIn($input: CheckInSessionInput!) {
  checkInSession(input: $input) { id status intent }
}
```

Variables:

```json
{ "input": { "intent": "user asked to audit prompt coverage for topic X" } }
```

Sample response:

```json
{
  "data": {
    "checkInSession": {
      "id": "cmses_abc123",
      "status": "active",
      "intent": "user asked to audit prompt coverage for topic X"
    }
  }
}
```

`checkInSession` / `checkOutSession` do NOT take `dryRun` / `intent` args.
`intent` (check-in) and `report` (check-out) ARE the payload.

### checkOutSession — close a session with a report

```graphql
mutation CheckOut($input: CheckOutSessionInput!) {
  checkOutSession(input: $input) {
    id
    status
    summary {
      totalActions
      successCount
      failureCount
      byOperation { operation resourceType ok count }
    }
  }
}
```

Variables:

```json
{
  "input": {
    "sessionId": "cmses_abc123",
    "report": "Listed prompts for topic X; updated 2 with stale mention rates after confirmation.",
    "deviations": "none"
  }
}
```

Sample response — `summary` is server-computed from the audit trail of
writes made during the session, for comparison against your `report`:

```json
{
  "data": {
    "checkOutSession": {
      "id": "cmses_abc123",
      "status": "checkedOut",
      "summary": {
        "totalActions": 2,
        "successCount": 2,
        "failureCount": 0,
        "byOperation": [
          { "operation": "updatePrompt", "resourceType": "prompt", "ok": true, "count": 2 }
        ]
      }
    }
  }
}
```

`checkOutSession` returns `NotFoundError` for an unknown or foreign session
id, and `ConflictError` if the session was already checked out. After
checkout the session id stops granting access — a new task needs a new
check-in.

---

## Projects

### List all projects (bounded — no pagination)

```graphql
{
  projects {
    id
    name
    slug
    isInternal
    createdAt
  }
}
```

### Fetch a single project (result union)

```graphql
query Project($id: ID!) {
  project(id: $id) {
    __typename
    ... on QueryProjectSuccess {
      data { id name slug isInternal createdAt }
    }
    ... on NotFoundError { message }
    ... on ForbiddenError { message }
  }
}
```

### Project with topics (bounded list)

```graphql
query ProjectTopics($id: ID!) {
  project(id: $id) {
    ... on QueryProjectSuccess {
      data {
        id name
        topics { id name }
      }
    }
  }
}
```

### Project members (nested result union — note the `ProjectMembersSuccess` spread)

```graphql
query ProjectMembers($id: ID!) {
  project(id: $id) {
    ... on QueryProjectSuccess {
      data {
        id
        members {
          __typename
          ... on ProjectMembersSuccess {
            data { user { id name email role } joinedAt }
          }
          ... on ForbiddenError { message }
        }
      }
    }
  }
}
```

Members visibility requires staff/admin; for talents the `members` field surfaces `ForbiddenError` as a union member rather than failing the whole query.

---

## Prompts

### Cursor-paginated prompts in a project

```graphql
query PromptsPage($projectId: ID!, $first: Int, $after: String) {
  prompts(projectId: $projectId, first: $first, after: $after) {
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

Sample response:

```json
{
  "data": {
    "prompts": {
      "__typename": "QueryPromptsSuccess",
      "data": {
        "edges": [
          { "node": { "id": "cmpykrg0y000014vii9oi1k3f", "text": "What's the best CRM?" } }
        ],
        "pageInfo": {
          "hasNextPage": true,
          "endCursor": "eyJjcmVhdGVkQXQiOiIyMDI2LTA1LTIxVDIwOjQ0OjA4LjY0OFoiLCJpZCI6ImNtcGZxMGRoazM2NXcwMXBtNmJ3ZWloNG4ifQ=="
        },
        "totalCount": 25
      }
    }
  }
}
```

To get page 2: pass `pageInfo.endCursor` as `after`.

### Single prompt with relations

```graphql
query PromptDetail($projectId: ID!, $id: ID!) {
  prompt(projectId: $projectId, id: $id) {
    ... on QueryPromptSuccess {
      data {
        id text
        topics { id name }
        latestRun { id status source completedAt }
      }
    }
    ... on NotFoundError { message }
  }
}
```

### Prompt + its runs (nested pagination)

```graphql
query PromptRuns($projectId: ID!, $id: ID!, $first: Int, $after: String) {
  prompt(projectId: $projectId, id: $id) {
    ... on QueryPromptSuccess {
      data {
        id text
        runs(first: $first, after: $after) {
          edges { node { id status source completedAt } cursor }
          pageInfo { hasNextPage endCursor }
          totalCount
        }
      }
    }
  }
}
```

---

## Competitors

### Cursor-paginated competitors in a project

```graphql
query Competitors($projectId: ID!, $first: Int, $after: String) {
  competitors(projectId: $projectId, first: $first, after: $after) {
    __typename
    ... on QueryCompetitorsSuccess {
      data {
        edges {
          node { id name domain isHighlighted }
          cursor
        }
        pageInfo { hasNextPage endCursor }
        totalCount
      }
    }
    ... on NotFoundError { message }
    ... on ForbiddenError { message }
  }
}
```

Sample response (3-row page):

```json
{
  "data": {
    "competitors": {
      "__typename": "QueryCompetitorsSuccess",
      "data": {
        "totalCount": 6,
        "edges": [
          { "node": { "id": "comp_turing", "name": "Turing", "domain": "turing.com", "isHighlighted": false } },
          { "node": { "id": "comp_freelancer", "name": "Freelancer", "domain": "freelancer.com", "isHighlighted": false } }
        ]
      }
    }
  }
}
```

---

## Runs

### Filtered run list

```graphql
query Runs(
  $projectId: ID!
  $promptId: ID
  $source: String
  $dateRange: DateRangeInput
  $first: Int
  $after: String
) {
  runs(
    projectId: $projectId
    promptId: $promptId
    source: $source
    dateRange: $dateRange
    first: $first
    after: $after
  ) {
    __typename
    ... on QueryRunsSuccess {
      data {
        edges {
          node { id status source createdAt completedAt }
          cursor
        }
        pageInfo { hasNextPage endCursor }
        totalCount
      }
    }
    ... on ValidationError { message }
  }
}
```

Filters compose: omit `promptId` to get all prompts; omit `source` to get all sources; omit `dateRange` for all-time (newest-first via cursor). For "runs in the last 7 days," set `dateRange` instead of paging to a cutoff.

### Single run with mentions (specialized cursor — position + id)

```graphql
query RunMentions($id: ID!, $first: Int, $after: String) {
  run(id: $id) {
    ... on QueryRunSuccess {
      data {
        id status source
        mentions(first: $first, after: $after) {
          edges {
            node { id platform position isTarget context }
            cursor
          }
          pageInfo { hasNextPage endCursor }
          totalCount
        }
      }
    }
    ... on NotFoundError { message }
  }
}
```

Sample edge:

```json
{
  "node": {
    "id": "mention_1",
    "platform": "Toptal",
    "position": 1,
    "isTarget": true,
    "context": "Toptal (toptal.com) is a popular option..."
  },
  "cursor": "eyJwb3NpdGlvbiI6MSwiaWQiOiJtZW50aW9uXzEifQ=="
}
```

### Single run with citations (id-only cursor)

```graphql
query RunCitations($id: ID!, $first: Int, $after: String) {
  run(id: $id) {
    ... on QueryRunSuccess {
      data {
        id
        citations(first: $first, after: $after) {
          edges {
            node { id url domain pagePath position isTarget }
            cursor
          }
          pageInfo { hasNextPage endCursor }
          totalCount
        }
      }
    }
  }
}
```

`Run.mentions` / `Run.citations` batch across sibling runs: listing runs with `citations(first: N)` in one query is a single citation read, as long as every run uses the same `first` / `after`. For counts, shares or distributions over citations use `Query.analytics` instead — pulling raw citations is only for inspecting individual URLs.

---

## Analytics — `Query.analytics`

Every aggregated number (overview, trends, leaderboards, domain/page rankings, per-prompt or per-day stats, distributions) comes from **one query**: pick `measures`, optionally `groupBy` dimensions, narrow with `filter`, sort with `orderBy`. It reads pre-aggregated rollups, so a month × every prompt is **one request**, never a fan-out over runs.

**Never reconstruct aggregates by paging `runs` × `Run.citations` / `Run.mentions`.** If a question is "how many / what share / which rank / how does X distribute", it is an `analytics` call. Fall back to runs only when you need the raw answer text or individual citation URLs.

```graphql
query Analytics(
  $projectId: ID!
  $range: DateRangeInput!
  $measures: [AnalyticsMeasure!]!
  $groupBy: [AnalyticsDimension!]
  $filter: AnalyticsFilter
  $orderBy: [AnalyticsOrder!]
  $first: Int
  $after: String
) {
  analytics(
    projectId: $projectId
    dateRange: $range
    measures: $measures
    groupBy: $groupBy
    filter: $filter
    orderBy: $orderBy
    first: $first
    after: $after
  ) {
    __typename
    ... on QueryAnalyticsSuccess {
      data {
        edges {
          node {
            date source domain page pageKind isTarget
            prompt { id text }
            topic { id name }
            competitor { id name }
            answers prompts mentionRate citationRate avgPosition
            citingAnswers citations citedPrompts citationShare
          }
        }
        pageInfo { hasNextPage endCursor }
        totalCount
      }
    }
    ... on ValidationError { message }
    ... on ConflictError { message }
    ... on NotFoundError { message }
    ... on ForbiddenError { message }
  }
}
```

Select only the row fields you need — dimension fields are null unless grouped by, measure fields null unless requested. `prompt`, `topic` and `competitor` are full objects (e.g. `prompt { text topics { name } }`). To compare several projects, alias the field once per project in one request (max 15 aliases).

### Vocabulary

| Dimension (`groupBy`) | Row field | Notes |
|---|---|---|
| `DATE` / `WEEK` / `MONTH` | `date` | At most one. WEEK = ISO week labelled by its Monday; MONTH by its 1st. |
| `SOURCE` | `source` | `chatgpt`, `google_ai_mode`, `gemini` |
| `PROMPT` | `prompt` | |
| `TOPIC` | `topic` | Many-to-many: a prompt in 2 topics counts in both rows. |
| `COMPETITOR` | `competitor` | Switches the rate measures to per-competitor. Not combinable with citation dimensions. |
| `DOMAIN`, `PAGE`, `PAGE_KIND`, `IS_TARGET` | `domain`, `page`, `pageKind`, `isTarget` | Citation dimensions — citation measures only. `PAGE_KIND` is `HOMEPAGE` (path `/`) vs `OTHER`. Citations whose URL has no parseable domain are counted by no citation measure here (the dashboard shows them as `unknown`), so analytics totals can sit just under it. |

| Measure | Meaning |
|---|---|
| `ANSWERS` | Answers in the group. **An answer = one prompt × one source × one day.** |
| `PROMPTS` | Distinct prompts answered. |
| `MENTION_RATE` / `CITATION_RATE` / `AVG_POSITION` | Target brand's mean rates (0–100) / position — or each competitor's when grouped by `COMPETITOR` (answers without that competitor count as 0). 1 decimal. |
| `CITING_ANSWERS` | Distinct answers citing ≥1 URL in the group. |
| `CITATIONS` | Distinct (answer, page) pairs. **Use this for page-level distributions.** |
| `CITED_PROMPTS` | Distinct prompts with a matching citation. |
| `CITATION_SHARE` | `CITING_ANSWERS` as % of all answers in the same date/source/prompt/topic group. `null` when the group has no answers to divide by. |

`filter`: `sources`, `topicIds`, `promptIds`, `includeInactivePrompts` (default false, like the dashboard) apply to everything. `domains`, `domainContains`, `pages`, `pathPrefix`, `pageKind`, `isTarget` narrow **citation measures only** (a share keeps the full answer count as denominator). `domainContains` is case-insensitive; `pathPrefix` and `pages` are not, because paths are case-sensitive. `competitorIds`, `highlightedCompetitors` need `COMPETITOR` in `groupBy`. Resolve topic ids from `Project.topics { id name }` first.

**Which rows come back is decided by `groupBy`, never by `measures` or the citation filters.** Adding a measure fills one more field on the same rows, and narrowing to one page shrinks the counts on those same rows — so you can diff two responses safely. A citation *dimension* does narrow the rows, to the groups the rollup has a row for. A citation *filter* never does: every group with answers stays a row, reporting `0` rather than disappearing, which is what makes a `DATE` trend a complete time series even under `pathPrefix`. So the "which prompts cite this page" drill-down sorts (`orderBy: [{ measure: CITING_ANSWERS }]`) instead of expecting the filter to shorten the list. Prompt-scope filters (`promptIds`, `topicIds`, `sources`, `includeInactivePrompts`) do narrow the rows — they decide which answers exist at all.

An unsupported combination returns `ValidationError` naming what *is* available — read it and adjust rather than guessing. `ConflictError` means the project's citation rollups are still backfilling; retry in a few minutes.

Results are capped at 20000 groups (`ValidationError` beyond: narrow `dateRange` / `filter` or use a coarser grain like `WEEK`). Pages default to 100 rows, max `first: 1000`. Cursors are bound to the exact arguments — resend the same variables with `after` to continue. They are also bound to the data they were cut from: if a run lands mid-scroll, `after` returns a `ValidationError` telling you to restart from the first page, rather than silently skipping or repeating groups. Prefer a coarser grain or a narrower window over a long page walk.

### Recipes (variables for the query above)

**Homepage vs other pages cited, per day, for a month** — the whole distribution in one call:

```json
{ "range": { "from": "2026-08-01", "to": "2026-08-31" },
  "groupBy": ["DATE", "PAGE_KIND"], "measures": ["CITATIONS", "CITING_ANSWERS"] }
```

**Homepage share per cited domain** (competitor domains only):

```json
{ "groupBy": ["DOMAIN", "PAGE_KIND"], "measures": ["CITATIONS"],
  "filter": { "isTarget": false }, "orderBy": [{ "measure": "CITATIONS", "direction": "DESC" }] }
```

**Overview** (headline numbers): `{ "measures": ["MENTION_RATE", "CITATION_RATE", "AVG_POSITION", "ANSWERS", "PROMPTS"] }` — no `groupBy` returns one row.

**Mention / citation trend:** `{ "groupBy": ["DATE"], "measures": ["MENTION_RATE", "CITATION_RATE"] }` — add `"filter": { "topicIds": ["…"] }` for one topic, `"sources": ["chatgpt"]` for one source.

**Competitor leaderboard** (the dashboard's highlighted set):

```json
{ "groupBy": ["COMPETITOR"], "measures": ["MENTION_RATE", "CITATION_RATE", "AVG_POSITION"],
  "filter": { "highlightedCompetitors": true },
  "orderBy": [{ "measure": "MENTION_RATE", "direction": "DESC" }] }
```

Competitors with no stats in the window are omitted (their rates are 0).

**Top cited domains:** `{ "groupBy": ["DOMAIN"], "measures": ["CITATION_SHARE", "CITED_PROMPTS"], "orderBy": [{ "measure": "CITATION_SHARE" }], "first": 10 }`

**Most cited pages:** `{ "groupBy": ["DOMAIN", "PAGE"], "measures": ["CITED_PROMPTS"], "orderBy": [{ "measure": "CITED_PROMPTS" }], "first": 10 }`

**Which prompts cited this page?** `{ "groupBy": ["PROMPT"], "measures": ["CITING_ANSWERS"], "filter": { "domains": ["example.com"], "pages": ["/pricing"] }, "orderBy": [{ "measure": "CITING_ANSWERS" }], "first": 20 }` — every in-scope prompt is a row, so sort and read the top; the rest report `0`.

**Per-prompt, per-day stats** (the former `promptStats`): `{ "groupBy": ["DATE", "SOURCE", "PROMPT"], "measures": ["MENTION_RATE", "CITATION_RATE", "AVG_POSITION"] }` — page with `first: 1000` / `after`. Large projects over long windows can exceed the 20000-group cap; split `dateRange` (e.g. one week per call).

**Per-competitor, per-prompt, per-day stats** (the former `competitorPromptStats`): add `COMPETITOR` to the groupBy above and narrow with `competitorIds`.

**Topic comparison by source:** `{ "groupBy": ["TOPIC", "SOURCE"], "measures": ["MENTION_RATE", "CITATION_SHARE"] }`

---

## Mutations: dryRun → confirm → real call

Mutations accept `dryRun: Boolean = false` and `intent: String` (≤ 500 chars). The workflow is **always**: dryRun preview → user confirmation → real call. See `SKILL.md` for `intent` content rules (generic 1-sentence summary, never the verbatim prompt).

### createPrompt

```graphql
mutation CreatePrompt($input: CreatePromptInput!, $dryRun: Boolean, $intent: String) {
  createPrompt(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationCreatePromptSuccess {
      data { id text createdAt topics { id name } }
    }
    ... on ValidationError { message }
    ... on ConflictError { message }
    ... on NotFoundError { message }
  }
}
```

Variables:

```json
{
  "input": { "projectId": "<project-id>", "text": "What's the best CRM?", "topicIds": [] },
  "dryRun": true,
  "intent": "user asked to add a new prompt"
}
```

DryRun response: `data: { id: "dry-run", text: "..." }` — no row written, no audit row written. Re-run with `dryRun: false` to commit.

### updatePrompt

```graphql
mutation UpdatePrompt($input: UpdatePromptInput!, $dryRun: Boolean, $intent: String) {
  updatePrompt(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationUpdatePromptSuccess { data { id text topics { id name } } }
    ... on ConflictError { message }
    ... on NotFoundError { message }
    ... on ValidationError { message }
  }
}
```

Variables: `input: { id, projectId, text?, topicIds? }`.

### archivePrompt

```graphql
mutation ArchivePrompt($id: ID!, $dryRun: Boolean, $intent: String) {
  archivePrompt(id: $id, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationArchivePromptSuccess { data { id text active } }
    ... on NotFoundError { message }
  }
}
```

Note `archivePrompt` takes `id` directly (not an `input` object). It returns the prompt with `active: false`. Soft-delete; mentions/citations preserved.

### deletePrompt — cascade preview is the whole point

```graphql
mutation DeletePrompt(
  $id: ID!
  $projectId: ID!
  $dryRun: Boolean
  $intent: String
) {
  deletePrompt(id: $id, projectId: $projectId, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationDeletePromptSuccess {
      data {
        id
        cascade { mentions citations stats }
      }
    }
    ... on NotFoundError { message }
  }
}
```

DryRun response shows `cascade.{ mentions, citations, stats }` — the exact row counts that will be deleted. Always present these to the user before committing.

### importPrompts (batch — supports dryRun preview)

```graphql
mutation ImportPrompts($input: ImportPromptsInput!, $dryRun: Boolean, $intent: String) {
  importPrompts(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationImportPromptsSuccess {
      data {
        toCreate
        toUpdate
        toDelete
        validationErrors { row field message }
      }
    }
    ... on ValidationError { message }
    ... on ForbiddenError { message }
    ... on NotFoundError { message }
  }
}
```

Input:

```json
{
  "input": {
    "projectId": "<project-id>",
    "prompts": [
      { "text": "first new" },
      { "text": "second new" },
      { "id": "<existing-prompt-id>", "topicNames": ["new-topic"] },
      { "id": "<expired-prompt-id>", "delete": true }
    ]
  },
  "dryRun": true,
  "intent": "user uploaded a CSV of prompt updates"
}
```

DryRun returns `toCreate / toUpdate / toDelete` counts plus `validationErrors` per-row. **On the commit path (`dryRun: false`), `validationErrors` is always empty** — call dryRun first to get per-row diagnostics.

### createTopic / updateTopic / deleteTopic

```graphql
mutation CreateTopic($input: CreateTopicInput!, $dryRun: Boolean, $intent: String) {
  createTopic(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationCreateTopicSuccess { data { id name } }
    ... on ConflictError { message }
    ... on ValidationError { message }
  }
}

mutation UpdateTopic($input: UpdateTopicInput!, $dryRun: Boolean, $intent: String) {
  updateTopic(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationUpdateTopicSuccess { data { id name } }
    ... on ConflictError { message }
    ... on NotFoundError { message }
    ... on ValidationError { message }
  }
}

mutation DeleteTopic($id: ID!, $projectId: ID!, $dryRun: Boolean, $intent: String) {
  deleteTopic(id: $id, projectId: $projectId, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationDeleteTopicSuccess {
      data { id cascade { promptAssociations } }
    }
    ... on NotFoundError { message }
  }
}
```

`deleteTopic.cascade.promptAssociations` = the number of `(prompt, topic)` join rows that will be removed. Prompts themselves are not deleted.

### createProject / updateProject (admin-gated fields)

```graphql
mutation CreateProject($input: CreateProjectInput!, $dryRun: Boolean, $intent: String) {
  createProject(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationCreateProjectSuccess {
      data { id name slug isInternal createdAt }
    }
    ... on ConflictError { message }
    ... on ValidationError { message }
  }
}

mutation UpdateProject($input: UpdateProjectInput!, $dryRun: Boolean, $intent: String) {
  updateProject(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationUpdateProjectSuccess { data { id name slug } }
    ... on ConflictError { message }
    ... on NotFoundError { message }
    ... on ValidationError { message }
  }
}
```

`CreateProjectInput`: `{ name (req), domain (req), timezone?, region?, cadence?, endDate?, isInternal?, maxPrompts? }`. The last four are admin-only — non-admin requests setting them get `ForbiddenError`.

`UpdateProjectInput`: same fields plus `id`. Project management ability required (staff/admin on the project).

### addProjectMember / removeProjectMember

```graphql
mutation AddProjectMember($input: AddProjectMemberInput!, $dryRun: Boolean, $intent: String) {
  addProjectMember(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationAddProjectMemberSuccess { data { user { id name } joinedAt } }
    ... on ConflictError { message }
    ... on NotFoundError { message }
    ... on ValidationError { message }
  }
}

mutation RemoveProjectMember($input: RemoveProjectMemberInput!, $dryRun: Boolean, $intent: String) {
  removeProjectMember(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationRemoveProjectMemberSuccess { data { user { id name } joinedAt } }
    ... on NotFoundError { message }
    ... on ValidationError { message }
  }
}
```

Both take `input: { projectId, userId }`. Staff/admin only.

**Caveat:** `addProjectMember` dryRun returns synthetic Success even when the user is already a member — the service-level conflict check runs only on the real path. Don't rely on dryRun to detect duplicate-add conflicts.

### updateCompetitorHighlights

```graphql
mutation UpdateHighlights(
  $input: UpdateCompetitorHighlightsInput!
  $dryRun: Boolean
  $intent: String
) {
  updateCompetitorHighlights(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationUpdateCompetitorHighlightsSuccess {
      data {
        id
        competitors { id name domain isHighlighted }
      }
    }
    ... on NotFoundError { message }
    ... on ValidationError { message }
  }
}
```

Input: `{ projectId, highlightedIds: [<id>, ...] }`. **Replaces** the highlight set — competitors not in `highlightedIds` are un-highlighted. Response shows the full competitor list with new highlight state applied.

### mergeCompetitors

```graphql
mutation Merge(
  $input: MergeCompetitorsInput!
  $dryRun: Boolean
  $intent: String
) {
  mergeCompetitors(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationMergeCompetitorsSuccess {
      data {
        id
        target { id name domain }
        impact { mentionsReattributed citationsReattributed statsDeleted }
      }
    }
    ... on ConflictError { message }
    ... on NotFoundError { message }
    ... on ValidationError { message }
  }
}
```

Input: `{ sourceIds: [<id>, ...], targetId: <id> }`. Sources must all belong to the target's project.

DryRun returns accurate impact counts. Real path runs per-source merges in a loop — per-source atomic, **not** whole-batch atomic. A multi-source merge that throws on iteration N leaves earlier iterations committed.

### unmergeCompetitor

```graphql
mutation Unmerge(
  $input: UnmergeCompetitorInput!
  $dryRun: Boolean
  $intent: String
) {
  unmergeCompetitor(input: $input, dryRun: $dryRun, intent: $intent) {
    __typename
    ... on MutationUnmergeCompetitorSuccess {
      data {
        id
        competitor { id name domain }
        impact { mentionsRestored citationsRestored }
      }
    }
    ... on ConflictError { message }
    ... on NotFoundError { message }
    ... on ValidationError { message }
  }
}
```

Input: `{ id }` — the id of the previously-merged (child) competitor to restore. Returns `NotFoundError` if the competitor doesn't exist or wasn't actually merged.

---

## Error handling

### Top-level `errors[]` array

Only used for schema-validation failures (malformed GraphQL, unknown fields) and truly unexpected server errors. A 401 from the API endpoint means the key is invalid or revoked.

A top-level error with `extensions.code: "SESSION_REQUIRED"` (HTTP 403) means you have not checked in (or your session was already checked out). Call `checkInSession` and retry with the `X-Sonar-Session-Id` header set — this is not a permissions failure.

For known business errors, use the result-union members below.

### Typed errors as union members

| Error | When |
|---|---|
| `NotFoundError` | Resource doesn't exist or isn't visible to the principal. Existence is collapsed to NotFound across tenant boundaries to prevent enumeration. |
| `ForbiddenError` | The key's `abilities` lack `manage` for a write op, or `scopeProjects` doesn't include the requested project. |
| `ValidationError` | Input failed validation (Zod schema, malformed cursor, malformed date, unsupported analytics combination, analytics group cap exceeded). |
| `ConflictError` | Uniqueness violation (duplicate name, already-member, already-merged), or `analytics` over citation rollups that are still backfilling. |

Switch on `__typename` to handle them:

```graphql
{
  ... on QueryProjectSuccess { data { id name } }
  ... on NotFoundError { message }
  ... on ForbiddenError { message }
}
```

### Bad cursor → `ValidationError` union member

```graphql
{ prompts(projectId: $p, first: 5, after: "not-a-valid-cursor") {
    __typename
    ... on ValidationError { message }   # → "Malformed cursor."
  }
}
```

Same response if the cursor decodes to a payload with an unparseable date.

### "Unexpected error" with no further detail

This is the masked-error response for anything not in the typed-error union — usually a server-side problem (missing migration, deployment lag, infrastructure issue). Do not retry; surface the issue to the user and suggest they contact Sonar support.

### Query exceeds cost / depth / token budget — `BAD_USER_INPUT`

Some queries (e.g. nesting `citations` under a long `runs` list) can blow the per-request hardening budget. These now come back as a top-level `errors[]` entry with `extensions.code: "BAD_USER_INPUT"` and a message like `Syntax Error: Query Cost limit of 20000 exceeded, found 24550.` (or `… depth limit of 8 exceeded`, `… Aliases limit`, `… Token limit`). The fix is on your side — reduce `first` on the inner list, drop fields you don't actually need, or split the query into multiple round-trips. This is the same code the typed `ValidationError` union uses, so you can treat it as "user-input fixable" rather than "server is broken."

---

## Rate limit

100 requests/minute per API key. A 429 response means back off — don't retry immediately. When fanning out across many entities (e.g. listing runs × mentions), pace the requests or batch where possible.
