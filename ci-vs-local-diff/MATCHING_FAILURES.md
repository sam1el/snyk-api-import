
```markdown

# Matching failing tests (local run vs CircleCI)

This file lists test suites that failed in both the saved CircleCI log (`docs/ci.log`) and the most recent local run (`test-run.log`). Use this as a prioritized triage checklist — items here are high-impact because they fail consistently across environments.

Overlapping failing suites (CI ∩ local run)

- test/scripts/sync/sync-org-projects.test.ts
- test/lib/org.test.ts
- test/scripts/sync/clone-and-analyze.spec.ts
- test/scripts/import-projects.test.ts
- test/scripts/generate-imported-targets-from-snyk.test.ts
- test/scripts/generate-targets-data.test.ts
- test/system/list:imported.test.ts
- test/scripts/generate-org-data.test.ts
- test/system/sync.test.ts
- test/system/orgs:create.test.ts
- test/system/import:data.test.ts
- test/scripts/create-orgs.test.ts
- test/lib/git-clone.spec.ts
- test/lib/find-files.test.ts
- test/system/import.test.ts
- test/lib/index.test.ts
- test/scripts/sync/import-target.spec.ts
- test/scripts/polling.test.ts
- test/lib/orgs.test.ts
- test/lib/source-handlers/bitbucket-cloud/bitbucket-cloud.test.ts

Common root causes observed

- SNYK_LOG_PATH / filesystem ordering: `get-logging-path` performs `fs.mkdirSync` at import time. When tests mount memfs or expect the path to be created later, import-time side-effects cause ENOENT/failed-mkdir errors (see `create-orgs` and many script tests).
- Network / git clone dependencies: many tests attempt `git clone` from fixtures that reference `localhost` or internal GHE hosts; absent a local git server or appropriate mock these fail with DNS/connection errors (see `clone-and-analyze`, `git-clone.spec.ts`).
- Uncontrolled Snyk request mock responses: `test/mocks/snyk-request-manager.js` returns 500/ERROR by default in some flows causing upstream suites to fail unpredictably. Tests need per-test configurable responses.
- find-files / glob behavior diffs: the hardened glob matching changed which files are returned in some fixtures (python/mvn fixtures), causing expectations to differ.
- Test isolation / module re-use: several tests spy/mock module exports without isolating module imports which leads to `TypeError: Cannot redefine property` or `mockReset` errors — `jest.resetModules()` / `jest.isolateModules()` required.

Quick, high-impact fixes (priority order)

1. Ensure `SNYK_LOG_PATH` exists before modules import
  - Implement a Jest setup file (e.g. `test/jest.setup.js`) run via jest `setupFiles` that creates the directories used in tests and sets `process.env.SNYK_LOG_PATH` to a writable temp path. This will quickly remove many ENOENT/mkdir failures.

2. Mock `git clone` in tests
  - Add a deterministic `git clone` test mock (or a tiny local git HTTP server serving fixture repos) so tests relying on `git clone` do not attempt network access. This fixes GHE/localhost clone failures.

3. Make the Snyk request manager test-configurable
  - Convert `test/mocks/snyk-request-manager.js` into a per-test-configurable stub that can return specific codes and payloads, or provide a small helper test fixture that wrappers the mock to return the desired sequence of responses.

4. Use module isolation in tests that rewire spies
  - Update failing tests to use `jest.resetModules()` and require modules after mocks are installed, or wrap tests in `jest.isolateModules()` so module-level mocks don't leak and `defineProperty` conflicts go away.

5. Reconcile `find-files` expected outputs
  - Review `test/lib/find-files.test.ts` expectations and update either the tests or the new matching behavior if the intended semantics changed. Prefer to keep the secure, non-RegExp matcher and update tests to the new, documented behavior.

Medium-term items

- Consider making `get-logging-path` side-effect free at import time (return a getter function instead of calling `fs.mkdirSync` during import), or ensure the function is tolerant when memfs is used and directories are created later.
- Harden all tests that depend on external services (GHE, git, Snyk API) to use local fixtures or per-test mocks.

If you'd like, I can implement the Jest setup file and a small `git clone` mock next — the jest setup is the fastest win and will reduce many ENOENT/mkdir failures immediately. After that I can make the Snyk mock configurable.

```
