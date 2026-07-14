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
| `Project.topics` | `Query.promptStats`, `Query.competitorPromptStats` |
| `Project.members` | `Project.prompts`, `Project.competitors` |
| `Prompt.topics` | `Prompt.runs` |
| `Project.mentionTrend`, `citationTrend`, `leaderboard`, `topCitedDomains`, `mostCitedPages`, `citedPagePrompts` | `Run.mentions`, `Run.citations` |
| `Prompt.stats`, `Competitor.stats` (1000-row cap — use `Query.promptStats` / `competitorPromptStats` for wide ranges) | |

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

### Prompt + per-day stats (capped at 1000 rows — see "1000-row cap" below)

```graphql
query PromptStats($projectId: ID!, $id: ID!, $range: DateRangeInput!) {
  prompt(projectId: $projectId, id: $id) {
    ... on QueryPromptSuccess {
      data {
        id
        stats(dateRange: $range) {
          date source mentionRate citationRate avgPosition
        }
      }
    }
  }
}
```

For wide ranges (>1000 rows) the field throws `BAD_USER_INPUT`. Use `Query.promptStats` (below) instead.

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

### Competitor + per-day stats (capped at 1000 rows)

```graphql
query CompetitorStats(
  $projectId: ID!
  $first: Int
  $range: DateRangeInput!
  $promptId: ID
) {
  competitors(projectId: $projectId, first: $first) {
    ... on QueryCompetitorsSuccess {
      data {
        edges {
          node {
            id name
            stats(dateRange: $range, promptId: $promptId) {
              date promptId source mentionRate citationRate avgPosition
            }
          }
        }
      }
    }
  }
}
```

Same 1000-row cap as `Prompt.stats`. For wide ranges use `Query.competitorPromptStats`.

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

**Known limitation:** `Run.mentions` / `Run.citations` instantiate a fresh DataLoader per Run instance, so listing many runs and asking for mentions in one query fires N service calls (no cross-run batching). Acceptable for one-run-at-a-time navigation; for "list runs × mentions" patterns, paginate runs first then fetch mentions per run separately.

---

## Top-level stat connections (use these for wide date ranges)

### `promptStats`

```graphql
query PromptStats(
  $projectId: ID!
  $range: DateRangeInput!
  $source: String
  $promptId: ID
  $first: Int
  $after: String
) {
  promptStats(
    projectId: $projectId
    dateRange: $range
    source: $source
    promptId: $promptId
    first: $first
    after: $after
  ) {
    __typename
    ... on QueryPromptStatsSuccess {
      data {
        edges {
          node { promptId date source mentionRate citationRate avgPosition }
          cursor
        }
        pageInfo { hasNextPage endCursor }
        totalCount
      }
    }
  }
}
```

Cursor encodes `{ date, promptId }`. Ordered by `date DESC`, `promptId ASC` tiebreak.

### `competitorPromptStats`

```graphql
query CompetitorPromptStats(
  $projectId: ID!
  $range: DateRangeInput!
  $competitorId: ID
  $promptId: ID
  $source: String
  $first: Int
  $after: String
) {
  competitorPromptStats(
    projectId: $projectId
    dateRange: $range
    competitorId: $competitorId
    promptId: $promptId
    source: $source
    first: $first
    after: $after
  ) {
    ... on QueryCompetitorPromptStatsSuccess {
      data {
        edges {
          node {
            competitorId promptId date source
            mentionRate citationRate avgPosition
          }
          cursor
        }
        pageInfo { hasNextPage endCursor }
        totalCount
      }
    }
  }
}
```

Cursor encodes `{ date, promptId, competitorId }`. Ordered by `date DESC`, `promptId ASC`, `competitorId ASC` tiebreak.

---

## Analytics

These fields live on `Project` directly (not result-unioned — errors bubble to top-level `errors[]`). All take a required `dateRange: DateRangeInput!` of `YYYY-MM-DD` calendar strings (inclusive).

All five — `overview`, `mentionTrend`, `citationTrend`, `leaderboard`, `topCitedDomains` — also accept optional `topicId: ID` and `promptId: ID` filters. Resolve topic IDs from `Project.topics { id name }` first, then pass them through. Passing an unknown topic or prompt id (or one that belongs to a different project) returns a top-level `NOT_FOUND` error — fix the id rather than retrying.

### Project overview

```graphql
query Overview($id: ID!, $range: DateRangeInput!, $source: String) {
  project(id: $id) {
    ... on QueryProjectSuccess {
      data {
        overview(dateRange: $range, source: $source) {
          mentionRate
          citationRate
          avgPosition
          activePromptCount
        }
      }
    }
  }
}
```

### Mention / citation trends

```graphql
query Trends($id: ID!, $range: DateRangeInput!, $source: String) {
  project(id: $id) {
    ... on QueryProjectSuccess {
      data {
        mentionTrend(dateRange: $range, source: $source) { date value source }
        citationTrend(dateRange: $range, source: $source) { date value source }
      }
    }
  }
}
```

Both trend fields use the same underlying query — the cookbook is built around `getMentionTrend` returning both rates in one call. `value` is a percentage (0–100).

### Leaderboard (target brand excluded)

```graphql
query Leaderboard($id: ID!, $range: DateRangeInput!, $source: String) {
  project(id: $id) {
    ... on QueryProjectSuccess {
      data {
        leaderboard(dateRange: $range, source: $source) {
          competitor { id name domain isHighlighted }
          mentionRate
          citationRate
          avgPosition
        }
      }
    }
  }
}
```

`isHighlighted` reflects the persistent flag on the competitor (joined in by the resolver), not a derived metric.

### Top cited domains

```graphql
query TopDomains(
  $id: ID!
  $range: DateRangeInput!
  $source: String
  $limit: Int
) {
  project(id: $id) {
    ... on QueryProjectSuccess {
      data {
        topCitedDomains(dateRange: $range, source: $source, limit: $limit) {
          domain
          citationCount
          share
        }
      }
    }
  }
}
```

`share` is a percentage (0–100) of (date, prompt, source) tuples whose runs cited that domain.

#### Narrowing to a single topic (and/or prompt)

Same call shape with `topicId` (and optionally `promptId`) added. Resolve the topic id from `Project.topics { id name }` first — the field expects an id, not a name. Example: top cited domains for the "Developers" topic.

```graphql
query TopDomainsForTopic($id: ID!, $range: DateRangeInput!, $topicId: ID!) {
  project(id: $id) {
    ... on QueryProjectSuccess {
      data {
        topCitedDomains(dateRange: $range, topicId: $topicId, limit: 10) {
          domain
          citationCount
          share
        }
      }
    }
  }
}
```

The same `topicId` / `promptId` args work on `overview`, `mentionTrend`, `citationTrend`, and `leaderboard`. Use them whenever you have the question "for this topic / this prompt …" instead of paging runs and aggregating client-side.

### Most cited pages

Pages (URL path × domain) ranked by distinct citing prompts. Same call shape as `topCitedDomains` — flat top-N list, no cursor.

```graphql
query MostCitedPages(
  $id: ID!
  $range: DateRangeInput!
  $source: String
  $limit: Int
) {
  project(id: $id) {
    ... on QueryProjectSuccess {
      data {
        mostCitedPages(dateRange: $range, source: $source, limit: $limit) {
          id
          domain
          page
          url
          citedPromptCount
        }
      }
    }
  }
}
```

`id` is `${domain}|${page}` — pages aren't standalone entities, so this composite is the only stable handle. Use it to thread through to the drill-down below.

### Page drill-down — which prompts cited this (domain, page)?

Pass the same `dateRange` / `source` you used in `mostCitedPages` to get the prompts behind a row. Returns full `Prompt` objects, so you can chain `.stats`, `.latestRun`, etc.

```graphql
query CitedPagePrompts(
  $id: ID!
  $domain: String!
  $page: String!
  $range: DateRangeInput
  $source: String
) {
  project(id: $id) {
    ... on QueryProjectSuccess {
      data {
        citedPagePrompts(
          domain: $domain
          page: $page
          dateRange: $range
          source: $source
        ) {
          id
          text
        }
      }
    }
  }
}
```

Typical flow: call `mostCitedPages` first, pick a row, then call `citedPagePrompts` with that row's `domain` + `page` and the **same** `dateRange` / `source` to get a consistent drill-down. Different filters produce different prompts — that's the live behavior, not a bug.

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
| `ValidationError` | Input failed validation (Zod schema, malformed cursor, malformed date, stats-cap exceeded). |
| `ConflictError` | Uniqueness violation (duplicate name, already-member, already-merged). |

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

### The 1000-row cap on `Prompt.stats` / `Competitor.stats`

These convenience fields are capped at 1000 rows for response-size sanity. Wide date ranges throw `ValidationError` in the top-level `errors[]`:

```json
{
  "errors": [
    {
      "message": "Prompt.stats result exceeds the 1000-row cap; narrow dateRange or call Query.promptStats with cursor pagination.",
      "extensions": { "code": "BAD_USER_INPUT" }
    }
  ]
}
```

When you see this, switch to `Query.promptStats` / `Query.competitorPromptStats` (cursor-paginated, no cap).

### "Unexpected error" with no further detail

This is the masked-error response for anything not in the typed-error union — usually a server-side problem (missing migration, deployment lag, infrastructure issue). Do not retry; surface the issue to the user and suggest they contact Sonar support.

### Query exceeds cost / depth / token budget — `BAD_USER_INPUT`

Some queries (e.g. nesting `citations` under a long `runs` list) can blow the per-request hardening budget. These now come back as a top-level `errors[]` entry with `extensions.code: "BAD_USER_INPUT"` and a message like `Syntax Error: Query Cost limit of 20000 exceeded, found 24550.` (or `… depth limit of 8 exceeded`, `… Aliases limit`, `… Token limit`). The fix is on your side — reduce `first` on the inner list, drop fields you don't actually need, or split the query into multiple round-trips. This is the same code the typed `ValidationError` union uses, so you can treat it as "user-input fixable" rather than "server is broken."

---

## Rate limit

100 requests/minute per API key. A 429 response means back off — don't retry immediately. When fanning out across many entities (e.g. listing runs × mentions), pace the requests or batch where possible.
