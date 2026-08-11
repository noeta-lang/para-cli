# para/cli

A CLI framework for Noeta where **the function signature is the spec**. You annotate ordinary functions with `#[Command]`; the framework discovers them by whole-program reflection, parses `argv`, coerces each token to the parameter's declared type, dispatches the call by name, generates `--help`, and turns the command's return value into a process exit code. No parser to configure, no argument table to keep in sync with the code — the signature *is* the table.

## What it provides

One pure-Noeta module, `para.cli`:

| symbol | kind | purpose |
| --- | --- | --- |
| `Command { about = "", name = "" }` | `@attribute(Function, Method)` | marks an invokable command; `name` overrides the derived name and MAY contain spaces (a nested path) |
| `Arg { help = "", short = "", long = "", env = "" }` | `@attribute(Param)` | per-parameter help, flag spellings, and an environment-variable fallback |
| `run(): int` | fn | dispatch the real `argv` (minus argv0), return the exit code |
| `dispatch(argv: List<string>): int` | fn | the testable core: dispatch a synthetic argv |
| `help_text(): string` | fn | the top-level `--help` text exactly as `dispatch` prints it — a testable seam for asserting on help output |

Plus the framework behavior that rides on those: derived `--help` (colorized on a TTY), nested subcommands, env-var fallbacks, and `--completions <bash|zsh|fish>` shell-completion scripts.

## Installation

```sh
noeta add para/cli
```

That asks the registry for the current release and writes the caret requirement for it, so no version is pinned here to go stale. It adds:

```toml
[dependencies]
para = { version = "^X.Y", package = "para/cli" }
```

The package is keyed `para`, so its module addresses as `para.cli`. It is pure Noeta — no `[trust]` entry needed.

## Usage

Import the names you use — the attribute structs must be imported **by name** for `#[Command]` / `#[Arg]` to resolve:

```noeta
use para.cli.{Command, Arg, run}
use std.{io, os}

#[Command(about: "Add two integers")]
fn add(a: int, b: int): int {
    return a + b   // an int return becomes the exit code
}

#[Command(about: "Greet someone")]
fn greet(name: string, #[Arg(short: "l", help: "shout it")] loud: bool = false): int {
    io.outln(if loud then "HELLO ${name}" else "hello ${name}")
    return 0
}

// `noeta test` never runs top-level statements, so this line is the real entry only.
os.exit(run())
```

```
$ myprog add 2 3          # exit 5
$ myprog greet Ada -l     # HELLO Ada
$ myprog --help           # lists the commands
$ myprog greet --help     # usage for one command
```

### Env-var fallback

A parameter's value is sourced in order: an explicit `argv` token (positional/named/short), else the `#[Arg(env: "NAME")]` variable when set, else the parameter's own default (or a usage error if it is required). The fallback is shown in `--help` as `[env: NAME]`.

```noeta
#[Command(about: "Bind a server port")]
fn serve(#[Arg(env: "PORT", help: "port to bind")] port: int = 8080): int { ... }
//  serve                     # $PORT if set, else 8080
//  PORT=9000 serve           # 9000
//  serve --port 3000         # 3000 (argv outranks env)
```

> [!NOTE]
> An env value goes through the same type coercion as an argv token — a non-numeric `$PORT` on an `int` parameter is a usage error (exit 2), even though the parameter has a default. It is not silently skipped.

### Nested subcommands

`#[Command(name: "remote add")]` gives a command a multi-token path. Selection matches the **longest** command-name token-prefix of `argv`, so `remote add …` beats a bare `remote`. Top-level `--help` groups shared first-tokens hierarchically. A partial path with no matching command (e.g. a bare `config` when only `config set`/`config get` exist) is a usage error (exit 2) that lists the valid continuations.

### Struct-typed parameters

A parameter declared with a struct type is bound not as one positional but by expanding its **fields** into flags: each field becomes a `--field` long flag (a bool field is a presence-flag), and struct fields are never positional. Supplied fields are coerced to their declared field types; omitted defaulted fields are filled by the struct's own defaults — including a *middle* one, which a positional list could not express. A missing non-defaulted field is a usage error (exit 2).

```noeta
struct ServerOpts {
    port: int
    host: string = "localhost"
    verbose: bool = false
}

#[Command(about: "Start the server")]
fn serve(opts: ServerOpts): int { ... }
//  serve --port 8080                                # host/verbose fall to the struct's defaults
//  serve --port=9090 --host example.com --verbose
//  serve --host example.com                         # error: missing required argument '--port'
```

### Colored help

When stdout is a terminal (`io.is_tty()`), `--help`/usage output is colorized — bold section headers, dimmed types and annotations. Piped output is plain, so captured help carries no escape codes.

### Shell completions

`--completions <bash|zsh|fish>` prints a sourceable completion script to stdout (exit 0) that completes the program's command-name tokens (nested paths included) and every command's flag names (`--long`/`-short`). It is a framework flag, not a `completions` subcommand, so it never collides with a user command name. A missing or unknown shell is a usage error (exit 2). The script is pure string generation from the command model — no user code is invoked.

```
$ myprog --completions bash > /etc/bash_completion.d/myprog
$ source <(myprog --completions zsh)
$ myprog --completions fish > ~/.config/fish/completions/myprog.fish
```

### Testing commands

`run()` is `dispatch(args.all().slice(1, …))`. Test against `dispatch` directly — because a command's `int` return is its exit code, an arithmetic command is self-verifying with no output to capture:

```noeta
@test {
    fn adds(): void {
        assert(dispatch(["add", "2", "3"]) == 5)
    }
    fn usage_error(): void {
        assert(dispatch(["add", "2"]) == 2)   // missing required
    }
}
```

## Behavior

- **Command selection.** With several commands, the first non-flag token picks the subcommand; with exactly one command, there is no subcommand token — every argv token is that command's argument.
- **Argument forms.** `--long value`, `--long=value`, `-s value`, `-s=value`; a `bool` parameter is a flag (`--flag` presence, `--flag=false`, or `--no-flag`); `--` ends option parsing; remaining bare tokens fill the non-bool parameters in declaration order. A parameter's long spelling defaults to its name (`#[Arg(long: ...)]` overrides it); it has no short unless `#[Arg(short: ...)]` declares one. Named arguments may come in any order.
- **Coercion.** `int` via `to_int`, `float` via `to_float`, `bool` via `true`/`1`/`false`/`0`; any other type receives the raw string.
- **Optional parameters.** A parameter with a default may be omitted; the argument list is left short and the callee fills its own default. Defaults are trailing-only, so once one optional is absent the rest are too.
- **Help.** `--help`/`-h` as the first token (or an empty argv, in multi-command mode) prints the top-level help; after a command name — anywhere before a `--` — it prints that command's usage. Top-level help matches how the parser dispatches: multi-command mode lists the commands, while single-command mode prints the lone command's own usage under the program's name — no `Commands:` section, because there is no subcommand token to type. Per-command help lists each parameter with its type and its `(-s)` / `[optional]` / `[env: NAME]` annotations; a struct-typed parameter is shown as its fields' `--flag`s, exactly as the parser accepts them.
- **Return values.** An `int` return becomes the exit code; any other return value is success (`0`); an `Err` returned by the command is a runtime failure (`1`).
- **Error output.** Every failure prints `error: <message>` to stderr. A usage error follows it with the command's usage line; a command-selection error follows it with the top-level help.
- **Exit codes.** `0` success (or `--help`), `1` the command returned an `Err`, `2` a usage error (unknown command/option, missing required argument, bad coercion, too many positionals).

## Examples

- [`examples/demo/`](examples/demo) — multi-command mode, nested subcommands.
- [`examples/single/`](examples/single) — single-command mode (no subcommand token).
- [`examples/struct-args/`](examples/struct-args) — struct-typed arguments.

## Requirements

None beyond the `noeta` toolchain — this package is pure Noeta.

## Development

Each directory under `examples/` is its own small package depending on this repo by path; run `noeta check` / `noeta test` there. See [AGENTS.md](AGENTS.md) for the repo layout and environment details.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
