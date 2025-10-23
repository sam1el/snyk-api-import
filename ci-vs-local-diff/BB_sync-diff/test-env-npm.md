# Test environment — Local npm test run

This file summarizes the local `npm test` run saved to `logs/npm-test.log`.

## Run summary (local)

- Test Suites: 25 failed, 34 passed, 59 total
- Tests:       60 failed, 5 skipped, 7 todo, 207 passed, 279 total
- Time:        764.685 s

See `logs/npm-test.log` for the full run output.

## High-level failures (local)

- DNS ENOTFOUND to placeholder SCM hosts (e.g., `gitlab.example`, `ghe.example`) causing many system tests to fail.
- ENOENT: missing generated log files under `test/scripts/fixtures/...` (e.g., `org-id-test.failed-imports.log`, `import-job-results.log`).
- Missing parameter errors in Snyk-related flows (e.g., "Missing required parameters: orgId or groupId must be provided").
- Child-process harness issues where `exec()` produced undefined and tests expected a ChildProcess (`.on('exit')` on undefined).

## Representative failing suites (local)

- `test/lib/org.test.ts`
- `test/scripts/import-projects.test.ts`
- `test/scripts/generate-imported-targets-from-snyk.test.ts`
- `test/system/*` (import, sync, list:imported, orgs:data)
- `test/lib/source-handlers/github/github.test.ts`
- `test/lib/source-handlers/gitlab/*`
- `test/lib/source-handlers/bitbucket-cloud/*`

## External endpoints observed failing (local)

- gitlab.example — DNS ENOTFOUND
- ghe.example / ghe.dev.snyk.io — DNS ENOTFOUND or fetch failures

If you want a machine-readable extraction of the exact lines/hosts from `logs/npm-test.log`, I can produce that next.
