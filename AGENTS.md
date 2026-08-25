# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/cli** Noeta package (a signature-is-the-spec CLI framework), extracted from the noeta monorepo. Toolchain issues (the language, the `noeta` binary, `std.*`, the reflection primitives `attributes_of`/`params_of`/`invoke`) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Scope

- **Pure Noeta.** No Rust crate, no `Cargo.toml`, no `noeta-ext-abi` dependency, no cargo anywhere. `.github/workflows/toolchain-pin.yml` is shared verbatim across the para repos and rewrites Cargo pins; here it finds none and reports "nothing to rewrite" — that is correct, do not add Rust to satisfy it.
- The whole published surface is `cli.noe`. Everything under `examples/` is a separate package that consumes it by path.

## Build & test

- **The examples are the test suite.** `cli.noe` carries no `@test` block; each `examples/*/` is its own package. Run `noeta check main.noe` and `noeta test main.noe` **from inside** the example directory.
- `noeta test` never runs top-level statements, so an example's `os.exit(run())` entry line is inert under test.
- Test through `dispatch(argv)` — the synthetic-argv core — never by spawning the program. `help_text()` exists for the same reason: it is the `pub` seam for asserting on help output.
- A `@test` block is white-box over **its own** package only, so an example's tests reach para/cli's `pub` surface and nothing else — importing a private helper is E0019. Widen the `pub` surface deliberately, never just to make a test compile.
- Land new behavior with cases in the example that exercises that mode: `demo/` multi-command and nested command paths plus typed exit codes, `single/` single-command mode, `default/` a default command, `two-defaults/` the two-defaults diagnostic (a deliberately misconfigured program — every dispatch in it is expected to exit 2), `struct-args/` struct-typed parameters.
- Run `noeta fmt` before committing. The tree is canonically formatted today and CI does **not** check formatting, so drift would go unnoticed.

## Conventions

- Attribute structs must be imported **by name** in consumers (`use para.cli.{Command, Arg, run}`) — importing the module alone does not resolve `#[Command]`.
- `examples/*/noeta.lock` is **gitignored**: the examples are demos, not package roots, and their locks regenerate on every run. Do not commit them.
- Markdown is never hard-wrapped — one long line per paragraph.
- **American English** throughout, in code, comments and docs (`behavior`, not `behaviour`).
- Prefer named constants or enums over repeated magic strings.
- **Conventional commits** for every commit title — `release.yml` compiles the GitHub release notes straight from them, so a non-conventional title lands under "Other". Commit each green slice as it completes, but **never `git push` without explicit authorization**.
- Implement in full — no stubs or TODOs; new functionality lands with tests.
- Keep `README.md` current: it is the behavior reference consumers read (argument forms, coercion, exit codes), so a behavior change that skips it is half-landed. Keep this file current too.

## Releasing

A release is a bumped `[package] version` in `noeta.toml` plus a new `v*` tag — never move a published tag. `toolchain = ">=0.5"` in the same file is the consumer-visible floor: raising it is breaking for anyone on an older toolchain, so it is a deliberate decision, not a follow-along edit when the toolchain releases.

## CI

- `ci.yml` — `noeta check` + `noeta test` on every example, using the released `noeta` named by the org-level `NOETA_VERSION` Actions variable (one toolchain release moves it for every para repo at once). No Rust job.
- `release.yml` — on a `v*` tag: reuses `ci.yml` as the gate, then `noeta publish` to `registry.noeta.dev` with keyless Sigstore provenance via GitHub OIDC, and creates the GitHub release.
- `toolchain-pin.yml` — reacts to the toolchain's `release-published` dispatch (a no-op rewrite here, see Scope). `docs-backfill.yml` — manual `noeta publish --docs-only`, to regenerate docs for an already-published tag.
