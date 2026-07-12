# sqltap-ls

Real-time SQL traffic viewer — a [Lisette](https://lisette.run) port of
[mickamy/sql-tap](https://github.com/mickamy/sql-tap).

sqltap sits between your application and your database (PostgreSQL, MySQL, or
TiDB), captures every query off the wire protocol, and displays it in an
interactive terminal UI or a browser. Inspect queries, view transactions, run
EXPLAIN, and catch N+1 patterns and slow queries — without changing
application code.

```
┌─────────────┐      ┌───────────────────────┐      ┌───────────────────────────┐
│ Application │─────▶│  sqltap serve (proxy) │─────▶│ PostgreSQL / MySQL / TiDB │
└─────────────┘      │  captures queries     │      └───────────────────────────┘
                     │  via wire protocol    │
                     └───────────┬───────────┘
                                 │ HTTP: SSE + JSON
                     ┌───────────▼───────────┐
                     │  sqltap watch  (TUI)  │
                     │  sqltap ci     (CI)   │
                     │  Browser     (Web UI) │
                     └───────────────────────┘
```

## Usage

One binary, three subcommands:

```bash
# 1. Start the proxy daemon (PostgreSQL example: proxy :5433 -> db :5432)
DATABASE_URL="postgres://user:pass@localhost:5432/db?sslmode=disable" \
  sqltap serve --driver=postgres --listen=:5433 --upstream=localhost:5432

# 2. Point your application at :5433 instead of :5432. No code changes —
#    the proxy speaks the native wire protocol.

# 3. Watch traffic in the TUI ...
sqltap watch localhost:9091

#    ... or in the browser (add --http to serve the web UI)
sqltap serve --driver=postgres --listen=:5433 --upstream=localhost:5432 --http=:8080

# CI mode: collect until SIGINT/SIGTERM or stream end, report, exit 1 on problems
sqltap ci localhost:9091
```

Set `DATABASE_URL` (or the env var named by `--dsn-env`) to enable EXPLAIN.
Config can also come from `.sql-tap.yaml` (same schema as the Go original);
CLI flags override file values.

## Differences from the Go original

Deliberate re-designs, not accidents of porting:

- **One binary.** `sql-tapd` + `sql-tap` became `sqltap serve` / `watch` / `ci`.
- **No gRPC / protobuf.** The daemon serves one HTTP API — `GET /api/events`
  (SSE stream of JSON events) and `POST /api/explain` — which the browser UI,
  the TUI, and CI mode all share. The web assets are reused byte-for-byte from
  the original.
- **No pgproto3.** The PostgreSQL proxy relays raw bytes and parses frames
  only for capture — nothing is ever re-encoded, so auth (SCRAM etc.) needs no
  special-casing.
- **Actors instead of mutexes.** Each proxied connection is a single actor
  task owning all capture state, fed whole frames by two socket-reader tasks
  over one channel. The broker is an actor too. There are no locks in the
  program.
- **No Bubble Tea / chroma / lipgloss.** Syntax and plan highlighting are
  hand-rolled ANSI; the TUI is a select-loop over key input and event stream.
- **Sum types where Go had flag fields.** A query outcome is
  `Ok { rows_affected } | Failed { message }`; a CI problem is
  `NPlus1 { … } | Slow { avg, … }`; transaction ids are `Option<string>`.

## Development

```bash
lis check          # typecheck
lis test           # run all tests
lis test -f "ci:"  # filter by test title
lis build          # binary at target/bin/sqltap-ls
```

Module map (`src/`): `event` (domain model + JSON wire type), `proxy/postgres`
and `proxy/mysql` (wire-protocol capture), `proxy` (tx tracking), `daemon`
(enrichment pipeline: normalize, N+1, slow), `detect`, `query`
(normalize/bind), `broker` (pub/sub actor), `server` (HTTP + SSE + web UI),
`client` (SSE consumer), `ci`, `tui`, `explain`, `dsn`, `config`, `highlight`,
`clipboard`.
