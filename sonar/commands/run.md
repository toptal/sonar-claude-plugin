---
description: Run a request against the Sonar GraphQL API. Plain English in, results out.
---

# /sonar:run

Explicit entry point for the Sonar plugin. Use this when you want to
be sure the skill triggers — for example when your prompt doesn't
mention "Sonar" but you want a Sonar query, or when you're demoing
the plugin and want a deterministic invocation.

The argument is a natural-language request. Examples:

- `/sonar:run show my projects`
- `/sonar:run list the 5 most recent prompts in project p_abc`
- `/sonar:run what's the mention rate for project p_abc this week?`
- `/sonar:run dry-run delete prompt p_xyz and show me the cascade impact`

Behavior:

- Load the `sonar` skill (which contains the endpoint, auth convention,
  schema, and query cookbook).
- Treat the argument as the user's request.
- Follow the skill's rules — including the dryRun-confirm-commit
  workflow for any mutation.

If `$SONAR_API_KEY` isn't set, the skill will say so and point at
Profile → API Keys.
