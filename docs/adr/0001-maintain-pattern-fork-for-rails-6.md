# 0001. Maintain a Pattern fork of odbc_adapter for Rails 6+

- Status: Accepted
- Date: 2021-08-04 (decision taken with the `v6.0.0` release; recorded 2026-11-03)
- Deciders: Pattern engineers who merged PR #1 ("Upgrade to rails6"). Names are in the git history.

## Context

Pattern applications need to reach databases through ODBC from ActiveRecord. The README's
"Datatype References" section shows the target was Snowflake, mapped onto Rails types. The
open-source `odbc_adapter` gem (Localytics, last RubyGems release `5.0.5` on 2019-04-05) only
supported ActiveRecord 5.x. The README says the upstream repository "was not actively being
maintained" for Rails 6.

Rails 6 broke the adapter in several places:

- `Arel::Visitors::BindVisitor` no longer exists.
- `ActiveRecord::ConnectionAdapters::Column#initialize` and `sql_for_insert` changed their signatures.
- Later, Ruby 3 changed keyword-argument handling (`translate_exception`), and Rails 6.1 changed how
  type maps are built.

Pattern also needed type behaviour that upstream would not want by default. Snowflake `NUMBER` /
`VARIANT` / `OBJECT` / `ARRAY` / `GEOGRAPHY` need mappings.

Options considered:

1. **Contribute upstream and wait for a release.** Upstream was inactive, so this would block
   Pattern's Rails 6 upgrade indefinitely.
2. **Switch to a DBMS-specific adapter** (for example a native Snowflake adapter). That means a
   larger change in every consuming app. (Reconstructed option: the repository does not record
   whether this was evaluated.)
3. **Fork into `patterninc/odbc_adapter` and maintain it.** Keep the public gem name and API, fix
   Rails/Ruby compatibility, and add Pattern-specific type mappings.

## Decision

Option 3: maintain a fork at `patterninc/odbc_adapter`.

- Keep the gem name `odbc_adapter`, the `adapter: odbc` config key and the
  `ODBCAdapter.register` extension API, so consuming apps only change their Gemfile source.
- Release by git tag and GitHub Release on the fork (`v6.x.y`, matching the Rails major version
  targeted), not by publishing to RubyGems.
- Put DBMS-specific behaviour (including Snowflake type mappings) in the adapter type map and
  per-DBMS adapter classes. Put consumer-specific behaviour behind opt-in config keys (for example
  `silo_fetch_all_tables`).

## Consequences

Positive:

- Pattern apps could move to Rails 6, then 6.1 (`v6.1.0`) and Ruby 3 (`v6.0.1`), without waiting on
  upstream.
- Pattern controls fixes for its own drivers (for example `v6.0.2` and `v6.1.2`).

Negative / costs:

- Pattern owns maintenance, including future Rails upgrades. A Rails 7 attempt (PR #7) was closed
  unmerged, so the fork currently targets `activerecord >= 6.1.7` and is only verified on 6.1.
- The type map changed behaviour for every DBMS, not just Snowflake. `decimal` is aliased to
  `float`, so `DECIMAL`/`NUMERIC` values are `Float`. This is recorded as a migration note in
  `CHANGELOG.md` (6.1.0).
- Releases are not on RubyGems. Consumers must use a git source pinned to a tag, and
  `ODBCAdapter::VERSION` (still `5.0.6`) has drifted from the tags (open PR #10).
- Upstream's CI (`.travis.yml`) is defunct and was not replaced, so tests only run locally against
  a real ODBC database.
- The fork is a public repository, so internal metadata must stay out of it (SRE-3790, PR #13).
