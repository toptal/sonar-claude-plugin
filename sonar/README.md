# Sonar GraphQL — Claude Code plugin

Query and mutate Sonar data from Claude Code, using your personal API
key. The plugin teaches Claude where the Sonar GraphQL endpoint is,
how to authenticate against it, how to discover the schema through
live introspection, and a cookbook of tested operations. Once
installed, you can just ask:

```
> show my 5 most recent prompts
> what's the mention rate for project p_abc this week?
> dry-run delete prompt p_xyz and show me the cascade
```

## Setup

### 1. Install the plugin

```
/plugin marketplace add toptal/sonar-claude-plugin
/plugin install sonar@sonar
```

Or via the branded URL: `/plugin marketplace add https://sonar.toptal.com/claude/marketplace.json`

### 2. Get an API key

Visit your Sonar profile and generate a key:

→ https://sonar.toptal.com/profile (Profile → API Keys)

The key is shown **once**. Copy it immediately.

### 3. Store the key in Claude Code settings

Edit `~/.claude/settings.json`:

```jsonc
{
  "env": {
    "SONAR_API_KEY": "sonar_xxxxxxxxxxxx"
  }
}
```

### 4. Restart Claude Code and verify

```
> Check my Sonar setup
```

Claude will run a smoke-test query and confirm your identity with your
email address.

## Pointing at staging or local dev

For internal use against staging or a local dev server, set
`SONAR_API_URL` alongside the key:

```jsonc
{
  "env": {
    "SONAR_API_KEY": "sonar_staging_xxxxxxxx",
    "SONAR_API_URL": "https://sonar-staging.toptal.net/api/graphql"
  }
}
```

For a local Next.js dev server: `http://localhost:3000/api/graphql`.

## Updates

Plugin updates ship via the standard Claude Code update flow whenever
the skill content changes. The schema itself is never bundled — Claude
introspects the live endpoint, so schema knowledge cannot drift from
the deployed API.

## Security

- API keys never live in the plugin, the repo, or any shared artifact.
  You generate your own; it sits only on your machine.
- Mutations always preview impact via `dryRun: true` before the real
  call — you confirm before anything writes.
- The plugin issues a generic `intent` summary into the audit log for
  each mutation (e.g. `"user asked to delete expired prompts"`), never
  the verbatim prompt. Treat `intent` like a commit message —
  informative, not sensitive.
- API keys are scoped (`read` vs `manage`) and revocable. If a key is
  compromised, revoke it from the profile page; the plugin will surface
  the resulting 401 cleanly.

## Troubleshooting

**"SONAR_API_KEY is not set."** Add it to `~/.claude/settings.json` per
step 3 above and restart Claude Code.

**Calls return 401.** The key is invalid or revoked. Generate a new
one on the profile page.

**`ForbiddenError` on a project.** Your key's `scopeProjects` doesn't
include that project, or your `abilities` lack `manage` for a write op.
The skill runs a scope-introspection query on first use; ask Claude
"what can my Sonar key do?" to see the current scope.

**The skill doesn't trigger when I ask about my data.** Use the slash
command: `/sonar:run list my recent prompts`. The description-based
trigger can under-fire on ambiguous queries; the slash command is the
reliable fallback.

## License

MIT. See LICENSE at the marketplace repository root.
