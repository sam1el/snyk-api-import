# Test environment — CI test run

This file summarizes the CI test run saved to `logs/ci.log`.

## Run summary (CI)

Note: `logs/ci.log` contains the full CircleCI job output. The CI run shows similar failing suites to local, with a few environment-specific differences.


- Test Suites: 21 failed, 38 passed, 59 total
- Tests:       50 failed, 5 skipped, 7 todo, 220 passed, 282 total
- Snapshots:   11 passed, 11 total
- Time:        649.765 s

The totals above differ from the local `npm` run summary in `logs/npm-test.log`; CI shows fewer failing suites and more passing tests overall, and includes additional service traces and HTTP 500 responses from Snyk API during some import tests.

## High-level failures (CI)

- Snyk API returned HTTP 500s in some import tests (agent/server errors with stack traces in the CI log).
- CI reached staging/test endpoints that returned server errors, whereas local runs often failed earlier with DNS ENOTFOUND.
- ENOENT errors referencing CI paths (e.g., `/home/circleci/project/test/scripts/fixtures/...`) for missing generated logs.

## Representative failing suites (CI)

- `test/lib/org.test.ts`
- `test/scripts/import-projects.test.ts`
- `test/scripts/generate-imported-targets-from-snyk.test.ts`
- `test/system/*` (import, sync, list:imported, orgs:data)
- `test/lib/source-handlers/github/github.test.ts`
- `test/lib/source-handlers/gitlab/*`

## External endpoints observed failing (CI)

- app.dev.snyk.io — flagged as invalid Snyk host in some CI messages
- Snyk API endpoints — HTTP 500 responses recorded during import tests
- gitlab.example / ghe.example — may appear when CI tries to reach configured placeholder hosts in some tests

For exact lines and errorRef IDs from the Snyk agent errors, see `logs/ci.log`.
