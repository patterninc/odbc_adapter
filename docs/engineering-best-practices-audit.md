# Engineering Best Practices Audit — odbc_adapter

| | |
|---|---|
| **Audit date** | 2026-09-08 |
| **Auditor** | Claude — gauge-repo skill |
| **Rubric version** | `item-credit-v1` — 2026-09-04 (`references/best-practices.md`) |

## Repo profile

`odbc_adapter` is a Ruby gem library — an ActiveRecord ODBC connection adapter — maintained by Pattern as a fork of `pedrocarmona/odbc_adapter` (originally Localytics) to keep it working on Rails 6+. Its deployment model is a consumed library: nothing is deployed, there is no AWS footprint, no UI, and no owned database schema (tests create throwaway tables against a caller-supplied ODBC DSN). The codebase is small (~15 Ruby files under `lib/`, 11 Minitest files under `test/`), with RuboCop tooling, a legacy Travis CI config (`.travis.yml`, Ruby 2.3.1 — defunct; no `.github/workflows/` exist), a Docker-based local test environment (`Dockerfile.dev`, `docker/`), and Backstage onboarding (`backstage.yaml`, owner `dev-pxm`). GitHub ownership is verified as `patterninc` via `gh repo view` (fork of `pedrocarmona/odbc_adapter`), so Pattern's inherited Wiz and Toolsmith controls apply. The local clone is shallow (1 squashed commit), so contributor history could not be assessed locally; the classic branch-protection API returned 403, but the org-level ruleset was verified via `gh api repos/patterninc/odbc_adapter/rulesets`. The library deployment model drives a large number of N/A verdicts (deploy, observability, service-contract, and UI items).

## Scorecard

| Metric | Value |
|--------|-------|
| **Critical gates** | **RED** |
| **Adjusted compliance** | **46.8%** |

Critical gates are RED: item 2 (AGENTS.md) and item 16 (required CI checks) are Gaps, and items 23 (unit tests) and 48 (reproducible builds) are Partial. Item 40 (scoped secrets) is a justified N/A — the repo holds no credentials and deploys nothing.

Adjusted compliance is calculated independently:

`(12 Met + 0.5 × 5 Partial) / (49 total - 18 justified N/A) = 14.5 / 31 = 46.8%`

### Status totals

| Status | Items |
|--------|------:|
| Met | 12 |
| Partial | 5 |
| Gap | 14 |
| N/A | 18 |
| **Total** | **49** |

### Per-category breakdown

| Category | Met | Partial | Gap | N/A |
|----------|----:|--------:|----:|----:|
| Documentation & Context | 1 | 0 | 6 | 2 |
| Guardrails & Enforcement | 6 | 0 | 4 | 3 |
| Testing & Feedback Loops | 2 | 2 | 3 | 6 |
| Environment & Tooling | 3 | 3 | 0 | 7 |
| Agent dispatch | 0 | 0 | 1 | 0 |
| **Total** | **12** | **5** | **14** | **18** |

## Documentation & Context

| # | Practice | Status | Evidence | Recommendation / rationale |
|---|----------|--------|----------|----------------------------|
| 1 | Skills / reusable prompt workflows | **Gap** | No `.claude/skills/`, `.claude/commands/`, or equivalent | Add a small skill/command for the recurring task of running the Dockerized DSN test matrix. |
| 2 | AGENTS.md | **Gap** | No `AGENTS.md` or `CLAUDE.md` | Add `AGENTS.md`: how to run `rake` (RuboCop runs as a test prerequisite), the DSN/`CONN_STR` test requirement, the Docker test flow, and fork constraints (keep divergence from upstream minimal). |
| 3 | Architecture decision records | **Gap** | No `docs/adr/`; key decisions (Rails 6 fork rationale, Snowflake type-mapping approach) live only in `README.md` prose | Capture the fork rationale and the datatype-mapping decision as short ADRs in `docs/adr/`. |
| 4 | Runbooks | **Gap** | No release/versioning docs; `README.md` covers testing only | Document the gem release process (version bump in `lib/odbc_adapter/version.rb`, tag, publish/consume path) in `docs/`. |
| 5 | API contract docs (OpenAPI / protobuf) | **Not applicable** | — | Library with no wire API; its contract is ActiveRecord's adapter interface, exercised by the integration suite. |
| 6 | README with setup & run instructions | **Met** | `README.md`: install, `database.yml` usage, test instructions, Docker workflow | — |
| 7 | Changelog with migration notes | **Gap** | No `CHANGELOG.md`; consumers cannot see what changed vs upstream or between versions | Add `CHANGELOG.md` starting from the Rails 6 fork; note upgrade steps per version. |
| 8 | On-call playbooks | **Not applicable** | — | Library gem; no production service or on-call surface — incidents occur in consuming applications. |
| 9 | CODEOWNERS | **Gap** | No `CODEOWNERS`; `backstage.yaml` names owner `dev-pxm` but reviews are not auto-assigned | Add `CODEOWNERS` mapping `*` to the dev-pxm team so ruleset-required reviews route automatically. |

## Guardrails & Enforcement

| # | Practice | Status | Evidence | Recommendation / rationale |
|---|----------|--------|----------|----------------------------|
| 10 | Linters | **Met** | RuboCop with `rubocop-rake`/`rubocop-minitest` (`.rubocop.yml`); `Rakefile` makes `rubocop` a prerequisite of `test` | — |
| 11 | Formatters | **Met** | RuboCop layout/style cops serve as the canonical Ruby formatter (`.rubocop.yml`, `NewCops: enable`) | — |
| 12 | Type checking | **Gap** | No RBS/Sorbet signatures | Low priority: add RBS signatures for the public `ODBCAdapter.register` surface if the gem's API grows. |
| 13 | Pre-commit hooks | **Gap** | No `.pre-commit-config.yaml`, `.githooks/`, or overcommit config | Add a pre-commit hook running `rubocop` (e.g. overcommit or pre-commit framework). |
| 14 | Commit message conventions | **Gap** | No commitlint/convention config; no documented format | Adopt Conventional Commits and document it in the contributing section (enables changelog automation for item 7). |
| 15 | Branch protection rules | **Met** | Org ruleset `require-pr-review` (active, `gh api repos/patterninc/odbc_adapter/rulesets/3174764`): blocks deletion and force-push on the default branch, requires 1 approving review with stale-review dismissal | — |
| 16 | Required CI checks before merge | **Gap** | No `.github/workflows/`; `.travis.yml` is defunct (Travis, Ruby 2.3.1); ruleset has no required status checks | Add a GitHub Actions workflow running RuboCop plus the test matrix (MySQL/PostgreSQL via ODBC, mirroring `docker/test.sh`) and make it a required check; delete `.travis.yml`. |
| 17 | Dependency allow-lists / deny-lists | **Not applicable** | — | Two runtime dependencies (`activerecord`, `ruby-odbc`) are fixed by the gem's purpose; a package policy list adds no value at this scale. |
| 18 | License compliance scanning | **Not applicable** | — | MIT gem with two stable runtime deps; license compliance for full dependency trees is scanned in consuming applications. |
| 19 | Secret scanning | **Met** | Inherited Pattern Wiz policy (owner verified as `patterninc`) | — |
| 20 | SAST / static analysis gates | **Met** | Inherited Pattern Wiz policy (owner verified as `patterninc`) | — |
| 21 | Max complexity limits | **Met** | `Metrics/CyclomaticComplexity` (max 9), `Metrics/AbcSize` (max 32), `Metrics/PerceivedComplexity` enforced via `.rubocop_todo.yml` burndown; note `Metrics/MethodLength`/`ClassLength` disabled in `.rubocop.yml` | — |
| 22 | Import boundary enforcement | **Not applicable** | — | Single flat gem (~15 files, one namespace); no architectural layers to protect. |

## Testing & Feedback Loops

| # | Practice | Status | Evidence | Recommendation / rationale |
|---|----------|--------|----------|----------------------------|
| 23 | Unit tests | **Partial** | 11 Minitest files in `test/`, but `test/test_helper.rb` unconditionally establishes a live ODBC connection — no isolated unit layer; even `registry_test.rb`/`version_test.rb` require a DSN | Split DB-independent tests (registry, version, column metadata) so they run without a live DSN; keep DB-backed tests separate. |
| 24 | Integration tests | **Met** | CRUD, metadata, migrations, and connection-management tests run against real MySQL and PostgreSQL over ODBC (`test/`, `docker/test.sh` three-DSN matrix) | — |
| 25 | Snapshot / golden-file tests | **Not applicable** | — | Library asserts adapter behavior directly; no generated artifacts suited to golden files. |
| 26 | Contract tests (Pact) | **Not applicable** | — | No service API; the contract is ActiveRecord's adapter interface, covered by the integration suite. |
| 27 | End-to-end tests (Playwright) | **Not applicable** | — | No UI of any kind; headless library. |
| 28 | Visual regression tests | **Not applicable** | — | No visual surface. |
| 29 | Test coverage thresholds | **Partial** | SimpleCov enabled in `test/test_helper.rb` but no `minimum_coverage` and no CI enforcement | Set a SimpleCov `minimum_coverage` floor and enforce it once CI (item 16) exists. |
| 30 | Mutation testing | **Gap** | No mutant/mutation config | Low priority: consider `mutant` on the type-mapping and quoting logic once CI is in place. |
| 31 | Load / performance benchmarks | **Gap** | No benchmark files | Low priority: a micro-benchmark of adapter overhead vs raw `ruby-odbc` would catch regressions in the hot query path, but throughput is dominated by driver/DB. |
| 32 | Flaky test quarantine | **Not applicable** | — | 11-file suite with no CI parallelism; a quarantine mechanism adds no value at this scale — failures are investigated directly. |
| 33 | Structured CI output | **Gap** | No CI exists (see item 16) | When adding CI, emit JUnit XML via `minitest-reporters` so failures are machine-readable. |
| 34 | Deterministic test fixtures | **Met** | `test/test_helper.rb` defines a fixed schema (`force: true`) and fixed seed rows recreated on every run | — |
| 35 | Smoke tests for deploys | **Not applicable** | — | Nothing is deployed; the gem is consumed by other applications. |

## Environment & Tooling

| # | Practice | Status | Evidence | Recommendation / rationale |
|---|----------|--------|----------|----------------------------|
| 36 | Devcontainer config | **Partial** | `Dockerfile.dev` + `docker/docker-entrypoint.sh` give a containerized dev/test environment (documented in `README.md`), but it is `ruby:2.4.0` on Debian jessie — below the gemspec's `required_ruby_version >= 2.7` | Rebase `Dockerfile.dev` on a supported Ruby image matching the gemspec floor; optionally wrap it as `.devcontainer/devcontainer.json`. |
| 37 | One-command setup (make dev) | **Partial** | `bin/setup` runs `bundle install`, but ODBC system drivers and DSN config remain manual (`README.md`, `bin/ci-setup`) | Extend `bin/setup` (or a `make dev` target) to detect/install unixodbc drivers or delegate to the Docker flow. |
| 38 | Seed scripts for local databases | **Met** | `bin/ci-setup` creates the test databases; `test/test_helper.rb` seeds deterministic starter data | — |
| 39 | MCP servers for external tools | **Met** | Toolsmith-managed MCP access (inherited; owner verified as `patterninc`) | — |
| 40 | Scoped secrets per environment | **Not applicable** | — | No credentials in the repo and nothing deployed; tests take caller-supplied `DSN`/`CONN_STR` env vars. |
| 41 | Preview environments per PR | **Not applicable** | — | Library gem; nothing to deploy per PR. |
| 42 | Hot-reload / watch mode | **Not applicable** | — | Library developed via `bin/console` REPL; a file-watch test loop adds little when the suite requires a live DSN. |
| 43 | Structured logging (JSON) | **Not applicable** | — | Library logs through the host application's ActiveRecord logger; log format is the consumer's decision. |
| 44 | Observable traces and metrics | **Not applicable** | — | Not a deployed service; instrumentation flows through ActiveRecord notifications in consuming apps. |
| 45 | Feature flags with local overrides | **Not applicable** | — | No runtime service; behavior is configured per-connection via `database.yml`. |
| 46 | Database migration tooling | **Not applicable** | — | The repo owns no database schema; the test schema is recreated from scratch on every run. |
| 47 | Dependency update automation | **Met** | Org-wide Wiz (owner verified as `patterninc`) | — |
| 48 | Reproducible builds (lockfiles) | **Partial** | `Gemfile.lock` is gitignored (`.gitignore`) per gem convention; dev tooling is exact-pinned in `odbc_adapter.gemspec` (e.g. `rubocop 1.48.1`), but runtime deps are floor-pinned and no Ruby version file or lockfile pins CI/test environments | Commit `Gemfile.lock` (acceptable for an app-consumed internal gem) or add a `.ruby-version` plus lockfile caching in CI so test runs are reproducible. |

## Agent dispatch

| # | Practice | Status | Evidence | Recommendation / rationale |
|---|----------|--------|----------|----------------------------|
| 49 | Agent-dispatch manifest | **Gap** | No `.agents/pattern-agents.json`; repo is Pattern-owned and Backstage-onboarded (`backstage.yaml`), so the dispatch model applies; no AWS footprint, so no `aws[]` is required | Add `.agents/pattern-agents.json` with `schema_version`, `github.repo`, `clickup_list_id`, `slack_channel`, and `skills.plugins` (omit `aws[]`). |

## Prioritized recommendations

1. **[S] Gap — AGENTS.md (item 2, critical gate):** Add `AGENTS.md` covering `rake` (lint + test), the `DSN`/`CONN_STR` requirement, the Docker test matrix, and fork constraints.
2. **[M] Gap — required CI checks (item 16, critical gate):** Add a GitHub Actions workflow (RuboCop + MySQL/PostgreSQL ODBC test matrix mirroring `docker/test.sh`), mark it required in the branch ruleset, and delete the defunct `.travis.yml`.
3. **[S] Partial — reproducible builds (item 48, critical gate):** Commit `Gemfile.lock` or add `.ruby-version` plus lockfile-pinned CI so builds and test runs are reproducible.
4. **[M] Partial — unit tests (item 23, critical gate):** Split DB-independent tests (registry, version, column metadata) into a layer that runs without a live ODBC DSN.
5. **[S] Gap — changelog (item 7):** Add `CHANGELOG.md` with migration notes, starting from the Rails 6 fork divergence.
6. **[S] Gap — agent-dispatch manifest (item 49):** Add `.agents/pattern-agents.json` with core fields; no `aws[]` needed.
7. **[S] Gap — CODEOWNERS (item 9):** Map `*` to the dev-pxm team so required reviews auto-assign.
8. **[S] Gap — pre-commit hooks (item 13):** Wire `rubocop` into a pre-commit hook (overcommit or pre-commit framework).
9. **[S] Gap — commit conventions (item 14):** Adopt Conventional Commits; feeds changelog automation.
10. **[S] Gap — release runbook (item 4):** Document the version-bump/tag/publish flow in `docs/`.
11. **[S] Gap — ADRs (item 3):** Record the fork rationale and Snowflake type-mapping decision in `docs/adr/`.
12. **[S] Gap — skills (item 1):** Add a reusable command/skill for the Dockerized DSN test-matrix workflow.
13. **[S] Gap — structured CI output (item 33):** Emit JUnit XML via `minitest-reporters` once CI exists.
14. **[L] Gap — type checking (item 12):** Consider RBS signatures for the public adapter/registry API.
15. **[L] Gap — mutation testing (item 30):** Consider `mutant` on quoting/type-mapping logic after CI lands.
16. **[L] Gap — performance benchmarks (item 31):** Optional micro-benchmark of adapter overhead vs raw `ruby-odbc`.
17. **[S] Partial — coverage threshold (item 29):** Set SimpleCov `minimum_coverage` and enforce in CI.
18. **[S] Partial — one-command setup (item 37):** Extend `bin/setup` to handle ODBC driver install or delegate to Docker.
19. **[M] Partial — devcontainer (item 36):** Rebase `Dockerfile.dev` from `ruby:2.4.0`/jessie to a supported image matching `required_ruby_version >= 2.7`.

## Declined practices

| # | Practice | Rationale |
|---|----------|-----------|
| 5 | API contract docs | No wire API; the contract is ActiveRecord's adapter interface, exercised by the integration suite. |
| 8 | On-call playbooks | Library gem; no production service or on-call surface. |
| 17 | Dependency allow/deny lists | Two purpose-fixed runtime dependencies; a policy list adds no value at this scale. |
| 18 | License compliance scanning | MIT gem with two stable deps; full-tree license scanning happens in consuming applications. |
| 22 | Import boundary enforcement | Single flat gem, one namespace; no layers to protect. |
| 25 | Snapshot / golden-file tests | Behavior asserted directly; no generated artifacts to snapshot. |
| 26 | Contract tests | No service API; adapter-interface contract covered by integration tests. |
| 27 | End-to-end tests | No UI; headless library. |
| 28 | Visual regression tests | No visual surface. |
| 32 | Flaky test quarantine | Tiny single-job suite; quarantine machinery adds no value. |
| 35 | Smoke tests for deploys | Nothing is deployed. |
| 40 | Scoped secrets per environment | No credentials in the repo and no deployment; tests use caller-supplied env vars. |
| 41 | Preview environments per PR | Nothing to deploy per PR. |
| 42 | Hot-reload / watch mode | REPL-driven library development; watch loop adds little with a live-DSN test suite. |
| 43 | Structured logging | Logging format is the consuming application's decision via ActiveRecord's logger. |
| 44 | Observable traces and metrics | Not deployed; instrumentation belongs to consuming apps. |
| 45 | Feature flags | No runtime service; behavior configured per connection. |
| 46 | Database migration tooling | Repo owns no schema; test schema recreated per run. |

## Beyond the checklist

- `Rakefile` wires RuboCop as a prerequisite of `rake test`, so lint runs on every local test invocation even without CI.
- `.rubocop_todo.yml` uses the auto-gen burndown pattern with dated regeneration metadata — lint debt is explicit and shrinkable, not hidden.
- `odbc_adapter.gemspec` sets `rubygems_mfa_required = 'true'`, hardening the gem-publish path.
- The org ruleset includes `require_extra_approval_for_unattributed_changes` and a Copilot code-review rule — AI-authored changes get extra review scrutiny by policy.
- Backstage onboarding (`backstage.yaml`) declares ownership (`dev-pxm`) and cost-center metadata, keeping the fork discoverable in Pattern's catalog.
- `docker/test.sh` runs the suite against three DSN configurations (MySQL, PostgreSQL ANSI, PostgreSQL UNICODE/UTF-8), catching encoding-specific regressions.
