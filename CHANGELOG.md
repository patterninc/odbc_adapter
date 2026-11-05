# Changelog

All notable changes to Pattern's fork of `odbc_adapter` are recorded here. Versions match the git
tags and GitHub Releases on `patterninc/odbc_adapter`. Dates are release dates.

Pattern releases are not published to RubyGems; install them from this repo by git tag. Note that
`ODBCAdapter::VERSION` (`lib/odbc_adapter/version.rb`) has read `5.0.6` on every Pattern release,
so the tag, not the constant, identifies the version.

Each breaking change has a **Migration** note.

## [Unreleased]

### Added
- `AGENTS.md`, `CHANGELOG.md`, `docs/adr/0001-maintain-pattern-fork-for-rails-6.md`,
  `.agents/pattern-agents.json` and the `.agents/skills/run-test-suite` skill. Documentation only;
  no behaviour change.

## [6.1.2] - 2024-06-27

### Added
- Opt-in `silo_fetch_all_tables` connection option (PR #9). When it is truthy in `database.yml`,
  `tables` runs `SHOW TABLES` and reads the `database_name`, `schema_name`, `name`, `kind` and
  `comment` columns. This is for DBMSs (such as Snowflake) where the ODBC catalog call fails on
  tables with long comments.

### Changed
- Without the option, `tables` reads the ODBC catalog with `fetch_all` again. The statement is now
  dropped in an `ensure` block.

**Migration:** none for existing configs. Set `silo_fetch_all_tables: true` only if your DBMS
supports `SHOW TABLES` with those column names.

## [6.1.1] - 2024-06-26

### Changed
- `tables` read the ODBC catalog with `stmt.fetch` instead of `fetch_all` (PR #8).

## [6.1.0] - 2023-03-22

Rails 6.1 upgrade (PRs #5, #6).

### Changed
- **Breaking:** requires `activerecord >= 6.1.7` and Ruby `>= 2.7`.
- **Breaking:** type map rewritten. `initialize_type_map` now calls `super` and maps native type
  names, with aliases for Snowflake types (see the README "Datatype References" table):
  `number`/`numeric` → `decimal`, then `decimal` → `float`; `float4`/`float8`/`double`/`real` → `float`;
  `variant` → `json`; `object`/`array` → `string`; `geography`/`geometry` → `char`; `bigint` → `BigInteger`.
- Removed the `BOOLEAN_TYPE`, `DATE_TYPE` and `JSON_TYPE` constants from the base and PostgreSQL
  adapters, and `VARIANT_TYPE` from the PostgreSQL adapter.
- `clear_cache!` now reloads the type map.
- Fixed booleans being converted to integers (PR #5).
- Reverted the 6.0.2 changes: `tables` uses `fetch_all` again and the PostgreSQL
  `disable_referential_integrity` works again.
- RuboCop upgraded to 1.48.1, with `rubocop-minitest` and `rubocop-rake`. `rake test` now runs
  RuboCop first.

**Migration:**
- Upgrade the app to Rails/ActiveRecord 6.1.7+ and Ruby 2.7+ before taking this version.
- `DECIMAL`/`NUMERIC`/`NUMBER` columns are now cast to Ruby `Float`, not `BigDecimal`/`Integer`.
  Check money or precision-sensitive code.
- `VARIANT` columns now come back parsed as JSON. `OBJECT`/`ARRAY` columns come back as strings.
- Code that referenced `BOOLEAN_TYPE`, `DATE_TYPE` or `JSON_TYPE` on the base or PostgreSQL
  adapter must stop doing so. `NullODBCAdapter` still defines `VARIANT_TYPE`, `DATE_TYPE` and `JSON_TYPE`.

## [6.0.2] - 2023-02-13

### Fixed
- `ArgumentError (negative string size (or size too big))` from the driver (PRs #3, #4). `tables`
  read the catalog with `stmt.fetch` instead of `fetch_all`. PostgreSQL
  `disable_referential_integrity` was turned into a no-op. Both were reverted in 6.1.0.

## [6.0.1] - 2022-12-13

### Fixed
- Ruby 3 keyword arguments: `translate_exception(exception, **message)` (PR #2).

**Migration:** subclasses that override `translate_exception` must accept keyword arguments.

## [6.0.0] - 2021-08-04

First Pattern release: Rails 6 support (PR #1). See
[ADR 0001](docs/adr/0001-maintain-pattern-fork-for-rails-6.md).

### Changed
- **Breaking:** `ODBCAdapter::Column.new` and `#new_column` no longer take `default_function` or
  `collation`. The signature is `(name, default, sql_type_metadata, null, table_name, native_type)`.
- **Breaking:** PostgreSQL `sql_for_insert` signature is now `(sql, pk, binds)`.
- Removed `Arel::Visitors::BindVisitor` (gone in Rails 6). The MySQL adapter uses
  `Arel::Visitors::MySQL` and the Null adapter uses `Arel::Visitors::PostgreSQL`. With
  `prepared_statements` on, binds are substituted into the SQL string (`bind_params`) and are no
  longer passed to the driver.
- Added `json` and date type support.
- Declared `activerecord >= 5.0.1` as a gemspec dependency; removed the `activerecord 5.0.1` pin
  from the `Gemfile`.

**Migration:** custom adapters registered with `ODBCAdapter.register` that call `new_column` or
`Column.new`, or override `sql_for_insert`, must use the new signatures.

## Earlier versions

Versions up to `5.0.5` (2019-04-05) are upstream Localytics releases, published on RubyGems. Their
history is in the git log and tags `v3.2.0` to `v5.0.5`.

[Unreleased]: https://github.com/patterninc/odbc_adapter/compare/v6.1.2...HEAD
[6.1.2]: https://github.com/patterninc/odbc_adapter/compare/v6.1.1...v6.1.2
[6.1.1]: https://github.com/patterninc/odbc_adapter/compare/v6.1.0...v6.1.1
[6.1.0]: https://github.com/patterninc/odbc_adapter/compare/v6.0.2...v6.1.0
[6.0.2]: https://github.com/patterninc/odbc_adapter/compare/v6.0.1...v6.0.2
[6.0.1]: https://github.com/patterninc/odbc_adapter/compare/v6.0.0...v6.0.1
[6.0.0]: https://github.com/patterninc/odbc_adapter/compare/v5.0.5...v6.0.0
