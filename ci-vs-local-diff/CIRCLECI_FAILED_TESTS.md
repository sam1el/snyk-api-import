
# CircleCI test run - failing tests summary

## Quick summary

- Many test suites failed in the CircleCI excerpt (representative list below). The failures span unit, integration and system tests.
- Representative failing suites (not exhaustive):


  This document summarizes the high-level failing suites seen in the CircleCI `ci.log` and suggests focused remediation actions.

  Quick summary

  The failing suites include a mix of unit, integration and system tests. Many failures are caused by external dependencies (git hosts, Snyk API) or brittle test mocking.

  Representative failing suites (non-exhaustive)

  - `test/scripts/sync/sync-org-projects.test.ts`
  - `test/lib/org.test.ts`
  - `test/scripts/sync/clone-and-analyze.spec.ts`
  - `test/scripts/import-projects.test.ts`
  - `test/scripts/generate-imported-targets-from-snyk.test.ts`
  - `test/scripts/generate-targets-data.test.ts`
  - `test/system/list:imported.test.ts`
  - `test/scripts/create-orgs.test.ts`
  - `test/lib/git-clone.spec.ts`
  - `test/lib/find-files.test.ts`
  - `test/scripts/sync/import-target.spec.ts`
  - `test/scripts/polling.test.ts`
  - `test/lib/orgs.test.ts`

  Main failure themes

  - Mock configuration and spy issues: tests attempt to spy on non-configurable exports or assume mutable module state.
  - Network/git clone failures: CI cannot reach internal fixtures or GHE hosts (`ghe.dev.snyk.io`).
  - Request manager / API mocking: tests sometimes hit the real Snyk API or receive maintenance/4xx/5xx responses from mocks.
  - Filesystem / environment variables: missing `SNYK_LOG_PATH` directories cause ENOENT or mkdir failures in tests.

  Actionable remediation (prioritized)

  1) Stabilize local file/log paths

  - Ensure CI and local test runs set `SNYK_LOG_PATH` to a writable temp directory. Add a `test/jest.setup.js` (or use `setupFiles`) that creates the path before modules import.

  2) Make the Snyk request manager mock deterministic

  - Convert `test/mocks/snyk-request-manager.js` into a fixtures-driven dispatcher (endpoint -> response). Tests can provide per-case fixtures for predictable outcomes.

  3) Stub or mock git clone in tests

  - Add a test-time mock for `src/lib/git-clone.ts` (or the underlying child_process call) to return prepared fixture directories. This eliminates DNS/network dependence for clone-heavy suites.

  4) Fix fragile spies

  - Use `jest.resetModules()` / `jest.isolateModules()` and set up mocks before requiring the module under test, or refactor exports to be spy-friendly (export objects or factories).

  Files to inspect while triaging

  - `src/lib/git-clone.ts` — central cloning logic
  - `test/mocks/snyk-request-manager.js` — primary Snyk API mock
  - `src/lib/find-files.ts` — file discovery and matching (recent changes may affect tests)

  If you'd like, I can implement #1 and #3 (create test setup that ensures `SNYK_LOG_PATH` exists and add a simple `git-clone` test mock) in a small PR. Which should I do first?
  - Tests are observing 401/403 or maintenance responses instead of expected mocked responses. Ensure `test/mocks/snyk-request-manager.js` is used in CI runs and that environment variables for API host/credentials are set to test values. Where external calls are needed, use HTTP mocks (nock) or updated request-manager mocks.
