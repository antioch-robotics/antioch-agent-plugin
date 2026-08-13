# CLI JSON contract

Use `--help` on the command you are automating. It is the authority for the
current option names and command-specific payload. This page defines the
shared rules that every finite JSON command follows.

## One document

One invocation writes one JSON document to stdout. A paged list uses the
common envelope:

```json
{"items": [], "next_cursor": null}
```

Pass `next_cursor` to `--cursor` to continue. A cursor already carries the
original filters, so do not repeat them. Commands that return one run or a
small result keep that result as the document instead of wrapping it again.

Progress boards, spinners, and errors use stderr. This keeps stdout safe for a
JSON parser, including `suite run --queue --json`. A follow command is the
explicit exception: `--follow --json` is an NDJSON stream with one complete
object per line. Do not parse it as one document; consume frames until the
`completed` frame.

## Values and time

Every JSON field whose name ends in `_at` is a Unix timestamp in microseconds.
Null remains null. JSON never contains a humanized clock label or duration;
the human table is the only place that uses the shared display formatter.

Repeatable filters are unions. For example, pass `--phase running --phase
queued` to keep both phases. A repeated scalar option that is not documented as
repeatable is a usage error; the CLI never keeps only the last value.

## Machine rows

`machine list --json` and `machine status --json` expose the user assignment
and runtime view: machine identity, direct URLs, GPU and placement, lifecycle
state, generation, project assignment, timestamps, stream state, and the local
`current` selection. They do not expose provider instance or boot-disk
identities, organization internals, or machine certificates.

The complete option and command list is intentionally not duplicated here.
Use `antioch <command> --help` before relying on a flag or a command-specific
result shape.
