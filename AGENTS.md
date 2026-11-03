# AGENTS.md

Guidance for AI agents (and humans) working in `patterninc/odbc_adapter`.

## Purpose

`odbc_adapter` is a Ruby gem: an ActiveRecord connection adapter that talks to any DBMS through ODBC
(via the `ruby-odbc` gem). Pattern forked it (upstream: `localytics/odbc_adapter`, GitHub parent
`pedrocarmona/odbc_adapter`) to keep it working on Rails 6 / 6.1, and later added Snowflake data type
mappings (see the README "Datatype References" section and [ADR 0001](docs/adr/0001-maintain-pattern-fork-for-rails-6.md)).

- It is a **library**. It is not deployed anywhere, has no AWS footprint, and owns no database schema.
- Full support is built in for MySQL and PostgreSQL. Any other DBMS falls back to the `Null` adapter,
  and consumers can register their own with `ODBCAdapter.register`.
- The repository is **public**. Do not commit internal hostnames, credentials, customer data or
  internal ownership metadata (see PR #13, SRE-3790).

## Directory layout

```
lib/
  odbc_adapter.rb                              # entry point; requires the AR adapter below
  active_record/connection_adapters/
    odbc_adapter.rb                            # ActiveRecord::Base.odbc_connection + ODBCAdapter < AbstractAdapter, type map
  odbc_adapter/
    adapters/{mysql,postgresql,null}_odbc_adapter.rb  # per-DBMS overrides
    registry.rb                                # maps the DBMS name reported by the driver to an adapter class
    database_metadata.rb, database_limits.rb   # driver/DBMS metadata fetched at connect time
    database_statements.rb, schema_statements.rb, quoting.rb
    column.rb, column_metadata.rb, error.rb
    version.rb                                 # ODBCAdapter::VERSION
test/                                          # Minitest suite (*_test.rb) + test_helper.rb
bin/setup, bin/console, bin/ci-setup           # bundle install; IRB with a connection; Travis DB setup
docker/, Dockerfile.dev                        # local MySQL + PostgreSQL test container
.rubocop.yml, .rubocop_todo.yml                # lint config (TargetRubyVersion 2.7)
.travis.yml                                    # legacy upstream CI (see "CI" below)
odbc_adapter.gemspec, Gemfile, Rakefile
docs/adr/                                      # architecture decision records
.agents/                                       # agent dispatch manifest and repo-local skills
CHANGELOG.md
```

## Requirements

From `odbc_adapter.gemspec`:

- Ruby `>= 2.7`
- `activerecord >= 6.1.7`, `ruby-odbc ~> 0.9`
- Dev: `minitest ~> 5.10`, `rake ~> 13.0`, `rubocop 1.48.1`, `rubocop-minitest 0.29.0`,
  `rubocop-rake 0.6.0`, `simplecov ~> 0.14`, `pry ~> 0.11.1`
- System: unixODBC (headers needed to build `ruby-odbc`) plus the ODBC driver for each DBMS you test against.

`Gemfile.lock` is git-ignored (normal for a gem).

## Commands

| Task | Command |
|---|---|
| Install dependencies | `bin/setup` (runs `bundle install`) |
| Lint only | `bundle exec rake rubocop` |
| Lint + tests (default task) | `CONN_STR='<odbc connection string>' bundle exec rake` |
| Tests via a DSN | `DSN=MyDatabaseDSN bundle exec rake test` |
| Interactive console | `DSN=MyDatabaseDSN bin/console` (or `CONN_STR=...`) |
| Build the gem locally | `bundle exec rake build` (from `bundler/gem_tasks`; writes `pkg/`) |

Notes:

- `rake test` has `rubocop` as a prerequisite (Rakefile), so a lint failure stops the test run.
- Tests need a **live database** reached through ODBC. `test/test_helper.rb` uses `CONN_STR`, else
  `DSN`, else the DSN `ODBCAdapterPostgreSQLTest`. There is no offline/mocked mode.
- Example connection strings used by `docker/test.sh` and `.travis.yml`:
  - MySQL: `DRIVER=MySQL;SERVER=localhost;DATABASE=odbc_test;USER=root;PASSWORD=;`
  - PostgreSQL: `DRIVER={PostgreSQL ANSI};SERVER=localhost;PORT=5432;DATABASE=odbc_test;UID=postgres;`
  - PostgreSQL UTF-8: `DRIVER={PostgreSQL UNICODE};SERVER=localhost;PORT=5432;DATABASE=odbc_test;UID=postgres;ENCODING=utf8`
- Docker path (README): `bundle package`, `docker build -f Dockerfile.dev -t odbc-dev .`, run the
  container, then `docker/test.sh` inside it. `Dockerfile.dev` is based on `ruby:2.4.0` (Debian
  jessie), which is older than the gemspec's `required_ruby_version >= 2.7`; expect to update the
  base image before this path works.
- The step-by-step test workflow is in [.agents/skills/run-test-suite/SKILL.md](.agents/skills/run-test-suite/SKILL.md).

## CI

There is no working CI on the default branch. `.travis.yml` is the upstream Travis config
(Ruby 2.3.1, MySQL 5.5 / PostgreSQL 9.1). It predates the Ruby 2.7 floor, and there is no
`.github/workflows/` directory. Run lint and tests locally and say in the PR what you ran.

## Release flow

Releases are git tags plus GitHub Releases on this repo (`v6.0.0` through `v6.1.2`), usually on the
merge commit of the PR being released. Release notes are the auto-generated "What's Changed" list.
Pattern versions are **not** published to RubyGems. rubygems.org has only the upstream Localytics
releases (latest `5.0.5`), so a consumer that wants a Pattern release has to point Bundler at this
repo as a git source pinned to a tag.

To release:

1. Merge the PR to `master`.
2. Add an entry to `CHANGELOG.md` (with migration notes for anything breaking).
3. Tag the merge commit `vX.Y.Z` and create a GitHub Release from the tag.

Known drift: `ODBCAdapter::VERSION` in `lib/odbc_adapter/version.rb` is still `'5.0.6'` on every
Pattern tag, so the gem version Bundler reports does not match the release tag. Open PR #10
proposes bumping it. Changing it is a code change, so do it in its own PR.

Do **not** run `bundle exec rake release`. It would push the gem to rubygems.org under the upstream
gem name.

## Conventions

- Every Ruby file starts with `# frozen_string_literal: true`.
- RuboCop with `NewCops: enable`, max line length 170. `.rubocop_todo.yml` holds accepted
  offences; shrink it, don't grow it.
- Tests are Minitest classes in `test/*_test.rb` that require `test_helper`.
- DBMS-specific behaviour goes in `lib/odbc_adapter/adapters/<dbms>_odbc_adapter.rb` (a subclass of
  `ActiveRecord::ConnectionAdapters::ODBCAdapter`) and is wired up through `Registry`. Don't add
  `if dbms == ...` branches to the base adapter.
- Behaviour that only some consumers want goes behind a `database.yml` config key that defaults to
  off. For example, `silo_fetch_all_tables` in `SchemaStatements#tables` (v6.1.2).
- Branches and PRs go against `master`; release notes come from PR titles, so make the titles
  descriptive.

## Guardrails

Do:

- Keep the gem working for the Rails floor in the gemspec (`activerecord >= 6.1.7`) and Ruby `>= 2.7`.
- Run `bundle exec rake rubocop`, and the test suite against at least one real database (PostgreSQL
  and/or MySQL) when you change `lib/`. If you can't, say "not run: <why>" in the PR.
- Add a `CHANGELOG.md` entry for user-visible changes, with migration notes for breaking ones.
- Record significant design decisions as a new ADR in `docs/adr/`.

Don't:

- Don't point the tests at a shared, staging or production database. `test_helper.rb` runs
  `create_table ... force: true` for `users` and `todos`, which drops existing tables with those names.
- Don't commit credentials, DSNs or connection strings for real environments. The repo is public.
- Don't publish to RubyGems (`rake release`) or move or rewrite existing release tags.
- Don't widen the `activerecord` or `ruby-odbc` constraints without running the suite on the new versions.
- Don't add a CLAUDE.md. This file is the agent guide.
