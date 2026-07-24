# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/cli** Noeta package (a signature-is-the-spec CLI framework, pure Noeta), extracted from the noeta monorepo. Toolchain issues (the language, the `noeta` binary, `std.*`, the reflection primitives) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Repo layout

- `noeta.toml` — the package manifest (`name = "para/cli"`). No `native` key: this package is pure Noeta.
- `cli.noe` — the whole surface: the `Command`/`Arg` attribute structs and the `run`/`dispatch` engine over the three reflection primitives (`attributes_of`, `params_of`, `invoke`).
- `examples/*/` — each a standalone package depending on this repo via `para = { path = "../.." }`, with its own committed `noeta.lock`. Each carries an `@test` block — `@test` only works in the entry/example program, so the examples ARE the test suite.
- `.github/workflows/` — CI (`ci.yml`) and the tag-triggered registry publish (`release.yml`).

## Build & test

Pure Noeta — no cargo anywhere in this repo.

- `noeta check <file>.noe` / `noeta test <file>.noe` in each `examples/*` directory is the test suite. `noeta test` never runs top-level statements, so the `os.exit(run())` entry line is inert under test.
- Test against `dispatch(argv)` (the synthetic-argv core), never by spawning the program.

## Conventions

- Attribute structs must be imported **by name** in consumers (`use para.cli.{Command, Arg, run}`) — importing the module alone does not resolve `#[Command]`.
- `noeta.lock` files under `examples/` **are committed** — leave resolved locks in place.
- Markdown never hard-wraps lines; American English throughout.
- Conventional commits. Never move a published `v*` tag — a release is a new tag.

## CI

`ci.yml` checks and tests every example with a pinned released `noeta`; `release.yml` publishes the tag to the hosted registry (`noeta publish`, keyless Sigstore provenance via GitHub OIDC). Both go green only once the toolchain repo is published under github.com/noeta-lang/noeta.
