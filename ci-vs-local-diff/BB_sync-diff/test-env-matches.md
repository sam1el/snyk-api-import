# Test environment — Matches & differences (npm vs CI)

This document lists what matched between the local `npm` run (`logs/npm-test.log`) and the CI run (`logs/ci.log`), and highlights key differences.

## Totals (npm vs CI)

- npm (local) — Test Suites: 25 failed, 34 passed, 59 total
- npm (local) — Tests:       60 failed, 5 skipped, 7 todo, 207 passed, 279 total
- CI — Test Suites: 21 failed, 38 passed, 59 total
- CI — Tests:       50 failed, 5 skipped, 7 todo, 220 passed, 282 total

The runs have different top-line totals; CI shows fewer failing suites and more passing tests compared with the local run. The failing suites still overlap significantly, but CI logs also include service-side HTTP 500 errors not visible when local runs fail early at DNS resolution.

## Matches (common failures)

- Many of the same failing suites appear in both runs: `test/lib/org.test.ts`, `test/scripts/import-projects.test.ts`, `test/scripts/generate-imported-targets-from-snyk.test.ts`, `test/system/*`, `test/lib/source-handlers/github/*`, `test/lib/source-handlers/gitlab/*`, etc.
- Common error types: ENOENT for missing test-generated logs, missing required parameters in Snyk import flows, and assertion mismatches (expected objects vs received empty objects).

## Differences (where runs diverge)

- Local run often shows DNS ENOTFOUND for placeholder hosts (e.g., `gitlab.example`, `ghe.example`) — these prevented tests from reaching staging services.
- CI sometimes reaches services and records HTTP 500 (Snyk agent) responses with stack traces; these are service-side errors not visible in local DNS failures.
- Absolute paths differ between runs (local uses `/Users/jbrimager/...`, CI uses `/home/circleci/project/...`) which affects ENOENT error messages and where generated logs are expected.

## Recommended quick actions (based on matches/differences)

1. Add Jest mocks for placeholder hosts (`gitlab.example`, `ghe.example`) so local runs don't fail on DNS and will exercise code paths similar to CI.
2. For Snyk-api-related tests, either mock the Snyk client responses or capture/forward the CI errorRef IDs to Snyk infra for root-cause investigation.
3. Normalize test expectations that rely on absolute paths or use configurable temp directories to avoid ENOENT mismatches between local and CI.

If you'd like, I can now produce a machine-readable list of hostnames/URL occurrences across both logs and create small Jest mock patches for the top 3 most frequent hosts.
