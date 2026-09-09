# store-postgres

A **PostgreSQL adaptor** for [`store`](https://github.com/broodlang/store), the
Brood storage library. Contains a native Postgres wire driver — SCRAM auth, the
v3 simple + extended query protocols, and type codecs — plus a supervised
connection pool, all pure Brood over `tcp`'s binary mode (no libpq). The
`pg` module implements `store`'s adaptor contract over that driver: `store` is
adaptor-agnostic, and this package makes it speak to Postgres.

## Usage

Depend on both `store` and `store-postgres`, then point `store`'s repo at the
`pg` adaptor with a connection pool:

```brood
(:alias store)
(:alias pg)

(let (pool (pg/start-pool {:host "localhost" :port 5432
                           :database "myapp" :user "postgres" :password "…"
                           :size 10}))
  (store/query (store/repo pg pool) "SELECT * FROM users WHERE id = $1" [42]))
```

See `store`'s README for the query/insert/migration API; this package only
supplies the driver and pool underneath it.

## How the driver behaves

**Statements are prepared once per connection.** The first time a connection sees a
piece of SQL it Parses it as a named statement and remembers its result columns;
later executions of the same text bind that name, which sends three protocol
messages instead of five and reads four frames instead of six — and lets Postgres
skip its own parse and plan. Two things follow:

- **DDL is safe.** A cached plan invalidated by DDL, or a statement discarded by a
  rolled-back transaction, comes back as a SQLSTATE the driver recognises; it
  forgets the connection's statements and re-Parses once. Callers never see it.
- **Build SQL with parameters, not interpolation.** The cache is keyed by statement
  text and capped at 64 per connection. Text assembled per call — `(str "… WHERE id
  = " id)` — is a new statement every time, which past the cap simply stops being
  cached, but is worth avoiding for the injection reasons anyway. `$1` placeholders
  are bound server-side.

**The pool hands out the most recently used connection**, not the least. A run of
queries stays on one connection rather than round-robining onto cold ones. Idle
members are not stranded: every connection runs its own keepalive.

## Publishing

Releases go to [hive](https://github.com/broodlang/hive), the Brood package
registry at <https://brood.fly.dev>.

**One-time setup** — register and mint an API token:

1. Create an account at <https://brood.fly.dev/register>.
2. Mint an API token on your <https://brood.fly.dev/settings> page (it's shown
   once), then expose it to `nest`:

   ```bash
   export HIVE_TOKEN=<your token>
   # or, persistently, add to ~/.config/brood/config.blsp:  :registry-token "<your token>"
   ```

**Each release:**

1. Bump `:version` in `project.blsp` — releases are **immutable**, so a version
   can never be republished.
2. Confirm the tests pass:

   ```bash
   nest test
   ```

3. Publish:

   ```bash
   nest publish
   ```

`nest publish` builds a source tarball (excluding `_deps/`, `tests/`, `.git/`,
and the lock file), records its sha256, and POSTs it to the registry. Only
`:version` (registry) dependencies are recorded — store-postgres depends on
[`store`](https://github.com/broodlang/store), which is itself published to the
registry, so a published store-postgres resolves cleanly. Docs build
automatically and appear at `https://brood.fly.dev/packages/store-postgres`.

## License

AGPL-3.0-only. See [LICENSE](LICENSE).
