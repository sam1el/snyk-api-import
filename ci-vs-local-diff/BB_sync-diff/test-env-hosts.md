# Test environment hosts summary

Top hosts found across npm and CI logs:

- jestjs.io: 56 occurrences
- api.snyk.io: 32 occurrences
- snyk.docs.apiary.io: 16 occurrences
- gitlab.example: 16 occurrences
- api.: 12 occurrences
- app.dev.snyk.io: 12 occurrences
- snyk.io: 12 occurrences
- ghe.example: 4 occurrences
- ghe.dev.snyk.io: 4 occurrences
- github.com: 2 occurrences

Files created:

- doc/hosts-npm.txt
- doc/hosts-ci.txt
- doc/hosts-combined.txt

Recommendation:

- Add network mocks for the top hosts to speed up and isolate system tests.
- If hosts look like example domains (gitlab.example, ghe.example) check test fixtures and environment variables.
