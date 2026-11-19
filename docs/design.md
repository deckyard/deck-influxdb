<!-- SPDX-License-Identifier: LGPL-3.0-only -->
<!-- SPDX-FileCopyrightText: Copyright (C) 2026 mplx <mplx@jennifer-lang.dev> -->

# Design

## Scope

The deck writes measurements to InfluxDB and reads them back: line-protocol
points built with `point` and the field and tag steps, written with `write`;
InfluxQL queries with `query`, parsed into `Series`; Flux queries with
`queryFlux`, returned as annotated CSV. It covers InfluxDB 1.x and 2.x, and
3.x as far as 3.x speaks those APIs.

It is deliberately not an administration client:

- **No management APIs.** Buckets, organizations, tokens, retention
  policies, tasks and dashboards are configured once, usually by hand or by
  infrastructure tooling; a program that pushes metrics does not need them.
  An InfluxQL statement that the 1.x `/query` endpoint takes (`CREATE
  DATABASE`, ...) still works through `query`.
- **No Flux result decoder.** Flux answers in annotated CSV, which the
  bundled `csv` module parses; a bespoke decoder would duplicate it.
- **No batching, buffering or retry queue.** `write` sends the points it is
  given in one request. When to flush and what to do with a failed batch is
  the caller's policy.
- **No 3.x SQL or Arrow Flight.** 3.x takes `client2` writes and InfluxQL
  through a 1.x client, which covers the push-metrics use.

## Decisions

- **A deck, not a bundled module.** A client for one vendor's database
  belongs in the deck ecosystem. The deck is one of the bundled modules that
  `jennifer` moves out (M30.1 in its milestones, formerly M28.32) and keeps
  the module's surface, so migrating changes only the import.
- **One `Client`, a `Version` discriminator.** `client` / `clientWith` make a
  1.x client, `client2` a 2.x one; `write` dispatches on `version`. The line
  protocol is the same for every generation, so the `Point` builders and
  `line` are shared. Unused fields of the other generation stay `""`.
- **Points carry rendered fragments.** Tags and fields are stored as escaped
  `key=value` strings, so one point mixes float, integer (`i` suffix), string
  and boolean fields, which a homogeneous map could not hold. Every builder
  step returns a new `Point`.
- **Strict line protocol.** A point that is not valid line protocol throws
  instead of being sent: no fields, an empty measurement or one starting with
  `#` (a comment line the server would drop), an empty tag or field key, or a
  newline anywhere (the point separator, with no escape, so it would corrupt
  the whole batch). A tag with an empty value is left out, as the official
  clients do.
- **Escaping that cannot be undone.** Commas, spaces and equals signs are
  backslash-escaped where the protocol needs it. A backslash is doubled only
  where it would pair with what follows (a special character, another
  backslash, the end of the component), so a value ending in `\` cannot
  escape the separator, and a lone backslash is kept, which 1.x stores
  literally where 2.x would unescape `\\`.
- **Nanosecond precision only.** Writes send `precision=ns`; `at` takes
  nanoseconds and `atTime` a `time.Time`.
- **Query cells are strings.** An InfluxQL result is a `Result` of `Series`
  whose rows are `list of list of string`: `time` as its RFC 3339 text,
  numbers in their shortest form, booleans as `"true"` / `"false"`, `null` as
  `""`. A typed cell union would cost every caller a match; a caller converts
  the columns it knows are numeric.
- **The statement goes in the body.** `query` POSTs `q=...` form-encoded
  rather than putting it in the URL, so statements (a `CREATE USER ... WITH
  PASSWORD` included) stay out of proxy and access logs and URL length
  limits.
- **Same result shape as the Prometheus deck.** The parsed `Series` matches
  the retrieval half of `@jennifer/prometheus`, so code reading either looks
  alike.

## Errors

`InfluxdbError` (kind `"influxdb"`) is the one struct for every error of
that kind: `status` is the HTTP status of the server's answer (0 when there
was none) and `code` a 2.x error's `code` (`unauthorized`, `not found`,
`invalid`, ...; `""` when the answer had none). It is thrown for:

- a non-2xx answer, with the server's message (1.x `error`, 2.x `message`)
  or `request failed with HTTP <status>` when the body has neither;
- an error reported inside a `200` InfluxQL result (top level or per
  statement, `status` 0);
- a `200` query answer that is not JSON;
- the point checks above, and a transport failure that is none of the
  generic kinds (malformed HTTP framing, a bad URL).

The generic kinds pass through unchanged, as the bundled module, which never
wrapped transport failures, does: a socket failure (refused, reset, failed
lookup, TLS) is `http`'s `net.NetError` (kind `"net"`, with `op`, `reason`,
`sent` and `replied`), a reached cap or a query reply past `json`'s decoding
caps is `"limit"`, a deadline `"timeout"`, an answer cut short `"closed"`,
a cancelled task `"cancelled"`. On interpreters without transport error
kinds (M28.65) a socket failure arrives as kind `"runtime"` and is wrapped as
an `InfluxdbError`.

## Security and limits

- **Credentials ride in headers.** 1.x Basic auth and the 2.x
  `Authorization: Token` header never appear in the URL.
- **Redaction.** Every `InfluxdbError` message passes through `redact`: the
  whole `Authorization` value first, then the bare token or password when it
  is at least 6 characters (shorter ones would match unrelated text). A
  generic-kind error is passed on only when its message holds no credential;
  otherwise it is wrapped and redacted.
- **Server text is cleaned.** Text from the server goes into a message with
  control characters (except tab) replaced by spaces, so it cannot forge log
  lines or terminal output, and cut at 512 characters.
- **Plain HTTP is allowed.** InfluxDB's default endpoint is
  `http://localhost:8086` and many installations run without TLS on a local
  network, so the deck does not refuse `http://`; credentials sent to a
  remote host over it travel in the clear. TLS client certificates are not
  wired; use a reverse proxy for them.
- Response size and deadlines are `http`'s defaults; JSON decoding is under
  `json`'s nesting and node caps.

## Testing and validation

`src/influxdb_test.j` covers the line protocol directly (escaping of every
component, backslashes, mixed field types, timestamps, every refused point),
the request builders (endpoint, query parameters, auth headers, form body),
query parsing (series, tags with `/` and `~` in their keys, top-level and
per-statement errors), error extraction, cleaning and redaction, and the
`InfluxdbError` fields. Writes, InfluxQL and Flux queries run against an
`httpd` server on a loopback port that checks the method, path, parameters,
auth and body as InfluxDB would. A refused connection is shown to arrive as
a `net.NetError` with no credential in its message, and an over-deep query
reply as kind `"limit"`.

The module's 2.x and 3.x work (M23.7) was validated against an independent
reference implementation when it was added. The tests never reach a real InfluxDB; `examples/write.j` writes and queries
against a server named on its command line (default
`http://localhost:8086`) and reports the error when none is running, so CI
runs it offline.

## Known limitations and open questions

- A result the server cut at its row limit (`"partial": true`) comes back as
  far as it goes, without a flag; page with `LIMIT` / `OFFSET`.
- Only nanosecond precision is written.
- No gzip request bodies, which large batches would benefit from.

## Dependencies on the interpreter

- The floor is `>=0.26.0`, the release that deprecates the bundled module
  (M30.1).
- Thrown error structs are classified by kind in `testing.assertThrows` and
  accepted by lint's L104 (M28.51 / M28.50); a decoding cap is kind
  `"limit"` (M28.68); a socket failure is a `net.NetError` (M28.65).
- `http` needs the `net` capability, so writing and querying need the
  default `jennifer` binary; building and rendering points also work on
  `jennifer-tiny`.
