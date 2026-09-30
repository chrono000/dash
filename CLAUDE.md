# Fail closed, always

Every change must fail closed. When a code path hits something ambiguous, missing, malformed, or unexpected, the correct outcome is to **deny, reject, or stop** — never to let it through.

A user complaining that a feature is blocked is a good outcome. Someone getting unauthorized access, seeing another tenant's data, or performing an action they shouldn't is not. Choose the first every time, even when the second is more convenient or the ambiguity looks harmless.

Concretely:

- **Authorization / permissions** — unknown role, missing scope, absent ownership record, or a check that can't be evaluated → deny. No implicit allow, no "default to the permissive branch."
- **Tenancy** — every read and write is scoped by tenant/owner. A query that can't establish the scope returns nothing; it does not fall back to unscoped.
- **Validation** — reject unrecognized inputs, extra fields, or values outside an explicit allowlist. Validate against what's permitted, not against what's forbidden.
- **Errors** — a thrown or unhandled error in a security-relevant path means the request fails. Never `catch` and continue as if the check passed. Don't return internal error detail to the client.
- **Conditionals** — the `else` / `default` branch of any access, tenancy, or state check is the restrictive one.
- **Dependencies** — if an auth service, token verifier, feature flag store, or rate limiter is unreachable, treat it as a denial, not as an outage to route around.
- **Feature flags / config** — absent or unparseable config means the feature is off and the safe path is taken.
- **Migrations & rollouts** — new restrictions apply immediately; don't grandfather existing records into a permissive state without being asked to.

When failing closed would break something real, say so and let the user decide. Don't quietly loosen it.

# Version bumps

When a change touches a project (`client`, `server`, `staff`, `server/builder-runner`), bump that project's `version` in its `package.json` (and the matching top-level entries in its `package-lock.json`, if the lockfile tracks it). Only bump projects that actually changed.

The bump is measured against `main`, not against the previous commit:

- Compare with `git diff --name-only origin/main...HEAD -- <project>` and `git show origin/main:<project>/package.json`.
- If the project's version on the branch is already higher than on `main`, it has been bumped for this release — leave it alone. Further changes before the next merge to `main` do not bump it again.
- If it still equals `main`'s version, bump it once, choosing the level from what changed in that project since `main`:
  - **New feature** → bump the minor (second) number and reset the patch: `3.5.1` → `3.6.0`.
  - **Minor change or bug fix** only → bump the patch (third) number: `3.5.1` → `3.5.2`.
  - **Major (first) number** → never bump it unless explicitly told to. Major bumps are not part of normal work.
- If the branch already has a patch bump and a new feature lands before the merge to `main`, raise it to a minor bump (`3.5.2` → `3.6.0`). A minor bump already in place covers any later features and fixes.

# Keep the docs in sync with the logic

When a change alters behavior that a Markdown file describes (prices, limits, free allowances, roles and permissions, error messages, setup steps, flows), that file has to be updated too. This covers the AI chat knowledge base in `client/data/knowledge/` (the source for the in-app assistant's answers), `docs/`, and the project READMEs and SKILL/FEATURE_MAP files.

- **Ask before editing them.** Name the affected files and the lines that no longer match, propose the new wording, and wait for approval. Don't change them silently as part of a code change.
- Check for them on every logic change, not only when asked: search the Markdown for the old value or message (`grep -rn '<old value>' --include='*.md'`).
- A knowledge-base edit reaches the assistant only after the index is rebuilt (`cd client && npm run ingest:rag`, after checking with `-- --dry-run`). Say so when proposing the change; don't run it without asking.
