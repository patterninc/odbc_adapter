---
name: run-test-suite
description: Lint and run the odbc_adapter Minitest suite against a real MySQL or PostgreSQL database over ODBC. Use when verifying a change under lib/ or test/ before opening a PR, or when asked to run the tests or RuboCop for this gem. Do not use against a shared, staging or production database, because the suite drops and recreates the users and todos tables.
---

# Run the odbc_adapter test suite

The tests need a live database reached through ODBC. There is no mocked mode, and there is no
working CI (`.travis.yml` is defunct), so this run is the only verification a change gets.

## Inputs

- `CONN_STR` or `DSN`: how to reach a **throwaway** database named `odbc_test`. Set only one;
  if both are set, `DSN` wins (`odbc_connection` checks `:dsn` first) and `CONN_STR` is ignored.
  If neither is set, `test/test_helper.rb` uses the DSN `ODBCAdapterPostgreSQLTest`.
- Which DBMS to cover: PostgreSQL, MySQL, or both. Cover both when you touch shared code
  (`lib/active_record/connection_adapters/odbc_adapter.rb` or `lib/odbc_adapter/*.rb`). Covering
  the one you touched is enough for `lib/odbc_adapter/adapters/<dbms>_odbc_adapter.rb`.

## Steps

1. **Install gems.** Run `bin/setup` (that is `bundle install`). This needs Ruby >= 2.7 and the
   unixODBC development headers, because `ruby-odbc` builds a native extension.
2. **Lint.** Run `bundle exec rake rubocop`. Fix the offences; don't add new entries to
   `.rubocop_todo.yml` to silence them.
3. **Check the ODBC driver is registered.** `odbcinst -q -d` must list the driver named in
   `CONN_STR` (`MySQL`, `PostgreSQL ANSI` or `PostgreSQL UNICODE`). `bin/ci-setup` shows how to
   register them (`odbcinst -i -d -f ...`) and create `odbc_test`.
4. **Run the suite** once per database. `rake` runs RuboCop and then the tests:
   - PostgreSQL: `CONN_STR='DRIVER={PostgreSQL ANSI};SERVER=localhost;PORT=5432;DATABASE=odbc_test;UID=postgres;' bundle exec rake`
   - PostgreSQL UTF-8: `CONN_STR='DRIVER={PostgreSQL UNICODE};SERVER=localhost;PORT=5432;DATABASE=odbc_test;UID=postgres;ENCODING=utf8' bundle exec rake`
   - MySQL: `CONN_STR='DRIVER=MySQL;SERVER=localhost;DATABASE=odbc_test;USER=root;PASSWORD=;' bundle exec rake`
   - Or with a DSN from `~/.odbc.ini`: `DSN=MyDatabaseDSN bundle exec rake test`
   - `docker/test.sh` runs all three in sequence inside the `Dockerfile.dev` container. That image
     is `ruby:2.4.0`, below the gem's Ruby 2.7 floor, so it needs a newer base image first.
5. **Run one file** while iterating: `bundle exec ruby -Itest -Ilib test/<name>_test.rb` (with
   `CONN_STR` set).
6. **Report.** In the PR, list each command you ran and the database/driver it ran against,
   with the pass/fail counts. If you could not run something (no Ruby, no driver, no database),
   write `not run: <reason>`. Never write "tests pass" for a run that didn't happen.

## Guardrails

- Only point `CONN_STR`/`DSN` at a local or disposable database. `test_helper.rb` runs
  `create_table :users, force: true` and `create_table :todos, force: true` and inserts fixture rows.
- Don't commit connection strings with real hosts or passwords. The repository is public.
- Don't change `required_ruby_version` or the `activerecord` constraint just to get the suite to run.
