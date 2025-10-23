# Failing tests (results from `npm test` run)

This file lists failing test suites and tests observed when running `npm test` on branch `fix/testing_failures`.

## Summary

- Test suites: 24 failed, 23 passed, 47 total
- Tests: 84 failed, 2 skipped, 7 todo, 138 passed, 231 total
- Snapshots: 2 failed, 11 passed, 13 total

---

### Full list of failing suites / failing tests (excerpts)

The following test files had failures (errors or assertion failures). For each file I list the failing tests or a representative failure message.

- `test/scripts/sync/sync-org-projects.test.ts`
  - multiple tests throwing: { error: { message: "Error calling Snyk api" }, statusCode: 500 }
  - examples: `skips target & projects error if getting default branch fails`, `successfully updated several targets (dryRun mode)`

- `test/scripts/sync/clone-and-analyze.spec.ts`
  - assertion mismatches (unexpected files in import lists)
  - network/git clone errors for localhost fixtures (fatal: unable to access 'https://localhost/...': Couldn't connect to server)
  - DNS or network errors (getaddrinfo ENOTFOUND gitlab.com)

- `test/system/orgs:data/errors.test.ts`
  - CLI output mismatch: expected "ERROR: Bad credentials" but received other SSL/local issuer errors

- `test/lib/git-clone.spec.ts`
  - expectations about successful git clones fail; git output shows "fatal: unable to access 'https://localhost/...': Couldn't connect to server"

- `test/system/sync.test.ts`
  - TypeError: Cannot read properties of undefined (reading 'on') when running CLI child process assertions

- `test/system/import.test.ts`
  - Import failures, e.g. "Expected a 201 response, instead received: {\"data\":\"\"}" and test assertions expecting empty stderr fail

- `test/scripts/polling.test.ts`
  - ENOENT: missing failed-polls log file (tests expect files under `test/scripts/` to exist)

- `test/scripts/generate-imported-targets-from-snyk.test.ts`
  - Snyk API 500 errors and unexpected target lists returned

- `test/scripts/generate-targets-data.test.ts`
  - "No targets could be generated. Check the error output & try again." thrown from script code

- `test/scripts/create-orgs.test.ts`
  - Failure creating SNYK_LOG_PATH directories (errors: "Failed to auto create the path ... provided in the SNYK_LOG_PATH")

- `test/lib/org.test.ts` and `test/lib/orgs.test.ts`
  - API calls returning 500 errors and tests timing out

- `test/lib/index.test.ts`
  - pollImportUrl errors: "Could not poll Url" and undefined return values for projects

- `test/scripts/sync/import-target.spec.ts`
  - expected projects imported `1` but received `0`

- `test/system/projects.test.ts`
  - updateProject errors: Missing required parameters (orgId/projectId)

- other failing suites: various system/integration tests that rely on external services or fixture files

---

## Categorized remediation suggestions

Below I group the failures into three actionable categories and describe practical fixes you can apply locally (mocks, env vars, test fixtures).

### 1) Local git server / clone failures (localhost fixtures)

Problem symptoms:
- Errors like: `fatal: unable to access 'https://localhost/...': Failed to connect to localhost port 443` or `getaddrinfo ENOTFOUND gitlab.com`.
- Tests that expect successful cloning or diff results from repos (e.g. `clone-and-analyze.spec.ts`, `git-clone.spec.ts`) fail with network errors.

What can be done (mocking / local server):
- Provide a local git HTTP server hosting test fixtures on `https://localhost` (self-signed certs may be needed). The test suite expects to clone `https://localhost/snyk-fixtures/...` so ensure those fixture repos exist and are reachable.
- Alternatively, modify the tests or test setup to mock git operations:
  - Replace `git` invocation in `src/lib/git-clone.ts` with a mock that returns a synthetic successful `repoPath` and `gitResponse` when tests run.
  - Use dependency injection in tests (or jest mocks) to stub out `child_process.exec` or `spawn` calls used for `git clone`.
  - Add a dedicated mock helper that creates temporary directories and populates them with the minimal fixture files expected by the tests (so clone simulation can succeed without network).
- For quick local runs, enable connecting to local fixtures by running a simple static Git HTTP server (e.g., `git daemon`, `git-http-backend`, or a small Express server that serves bare repositories) and ensure `https://localhost` points to it. Note: you'll likely need to disable strict SSL verification for git or provide a test CA.

Files to check/adjust:
- `src/lib/git-clone.ts` — central git clone logic
- `test/scripts/sync/clone-and-analyze.spec.ts` and `test/lib/git-clone.spec.ts` — tests expecting clone behavior

Quick local approach (no code edits):
1. Run a local git server serving fixtures under `/snyk-fixtures/` on https://localhost (requires certificates). Or run plain http and update tests to use `http://localhost` if acceptable.
2. Ensure DNS/resolution for `localhost` is working (it normally is) and that port 443 isn't blocked.

---

### 2) SNYK_LOG_PATH / missing directories (file-system errors)

Problem symptoms:
- Tests failing with messages like: `Failed to auto create the path /fixtures/create-orgs/1-org/ provided in the SNYK_LOG_PATH` or ENOENT when reading expected log files in `test/scripts`.
- Tests expecting certain log files to be created or read from.

What can be done locally:
- Ensure `SNYK_LOG_PATH` is set in the test environment and points to a writable directory. Many tests create or read files using paths derived from `SNYK_LOG_PATH`.
  - Example: export `SNYK_LOG_PATH=$(pwd)/test/tmp-logs` before running tests.
- Pre-create the directory structure expected by tests in `test/fixtures` or in a temporary writable folder and point `SNYK_LOG_PATH` there.
- Update test setup to create / clean expected log directories before running each test (jest setup or `beforeAll` hooks). Many tests already try to create paths but fail due to permissions or relative path differences; ensuring `SNYK_LOG_PATH` is absolute and writable resolves this.

Files to check/adjust:
- `src/lib/get-logging-path.ts` — logic that auto-creates the logging directory
- Tests under `test/scripts/create-orgs.test.ts`, `test/scripts/polling.test.ts`, and system tests referencing log files

Quick commands to try locally:
```bash
mkdir -p test/tmp-logs
export SNYK_LOG_PATH=$(pwd)/test/tmp-logs
npm test -- --testPathPattern=test/scripts
```

Or add `SNYK_LOG_PATH` export to your CI/test runner script.

---

### 3) Snyk API errors / mocked API responses

Problem symptoms:
- Tests failing with Snyk API errors: `{ error: { message: "Error calling Snyk api" }, statusCode: 500 }`.
- Tests expecting specific Snyk responses, target lists, or HTTP status codes (201 on import, 401 unauthorized, etc.).

What can be done using mocks:
- The repo contains `test/mocks/snyk-request-manager.js` (and other mocks). Update the mock to ensure it returns the expected status codes and bodies for the tests that assert on them.
  - For imports, ensure the mock returns `{ statusCode: 201, data: ... }` when the import kick-off endpoint is called.
  - For listing targets, ensure the mock returns the expected arrays of targets for the different integration types.
- Make the mock more configurable: allow per-test overrides (e.g., reading a JSON fixture in `test/fixtures` that instructs the mock what to return for given endpoints). This avoids hardcoding behavior and makes tests deterministic.
- For tests that expect 401 (Unauthorized) or other errors, ensure the mock returns those codes when `Authorization` or token values are set to specific values in the test.

Files to check/adjust:

## Failing tests (results from `npm test` run)

This file lists failing test suites observed when running `npm test` on branch `fix/testing_failures`.

Summary

- Test suites: 24 failed, 23 passed, 47 total
- Tests: 84 failed, 2 skipped, 7 todo, 138 passed, 231 total
- Snapshots: 2 failed, 11 passed, 13 total

Full list of notable failing suites (excerpts)

- `test/scripts/sync/sync-org-projects.test.ts`
  - multiple tests throwing: { error: { message: "Error calling Snyk api" }, statusCode: 500 }

- `test/scripts/sync/clone-and-analyze.spec.ts`
  - assertion mismatches (unexpected files in import lists)
  - network/git clone errors for localhost fixtures (fatal: unable to access 'https://localhost/...': Couldn't connect to server)

- `test/lib/git-clone.spec.ts`
  - expectations about successful git clones fail; git output shows "fatal: unable to access 'https://localhost/...': Couldn't connect to server"

- `test/system/import.test.ts`
  - Import failures, e.g. "Expected a 201 response, instead received: {\"data\":\"\"}" and test assertions expecting empty stderr fail

- `test/scripts/polling.test.ts`
  - ENOENT: missing failed-polls log file (tests expect files under `test/scripts/` to exist)

- `test/scripts/generate-imported-targets-from-snyk.test.ts`
  - Snyk API 500 errors and unexpected target lists returned

- `test/scripts/create-orgs.test.ts`
  - Failure creating SNYK_LOG_PATH directories (errors: "Failed to auto create the path ... provided in the SNYK_LOG_PATH")

- `test/lib/org.test.ts` and `test/lib/orgs.test.ts`
  - API calls returning 500 errors and tests timing out

- `test/lib/index.test.ts`
  - pollImportUrl errors: "Could not poll Url" and undefined return values for projects

Categorized remediation suggestions

1) Local git server / clone failures (localhost fixtures)

Symptoms

- `fatal: unable to access 'https://localhost/...': Failed to connect` or DNS errors like `getaddrinfo ENOTFOUND`.

Remediation

- Mock `git clone` in tests by stubbing `src/lib/git-clone.ts` or the child_process it uses. Return prepared fixture directories for tests.
- Alternatively run a local git HTTP server to serve fixtures for quicker local iteration.

Files to check

- `src/lib/git-clone.ts`
- `test/scripts/sync/clone-and-analyze.spec.ts`

2) SNYK_LOG_PATH / missing directories (filesystem errors)

Symptoms

- Tests fail with `Failed to auto create the path ... provided in the SNYK_LOG_PATH` or ENOENT when reading expected log files.

Remediation

- Set `SNYK_LOG_PATH` to a writable folder before running tests and/or add a `test/jest.setup.js` that creates the path in the test filesystem (real or memfs) prior to importing modules.

Quick local commands

```bash
mkdir -p test/tmp-logs
export SNYK_LOG_PATH=$(pwd)/test/tmp-logs
npm test -- --testPathPattern=test/scripts
```

3) Snyk API errors / mocked API responses

Symptoms

- Tests receive 500 or unexpected responses from the Snyk request mock and fail assertions that expect 200/201.

Remediation

- Make `test/mocks/snyk-request-manager.js` fixtures-driven and per-test configurable. Tests can write or reference fixtures that map endpoints to responses.

Suggested next steps (pick one to start)

- Create a small test setup (jest `setupFiles`) that ensures `SNYK_LOG_PATH` exists and is writable. This is the fastest win.
- Implement a `git-clone` test mock to remove network/DNS flakiness from clone-heavy suites.
- Make the Snyk request-manager mock configurable by test fixtures.

I can implement the first item (test setup for `SNYK_LOG_PATH`) now and re-run a subset of the tests. Which should I do next?
