# linenoise

Line editing for a terminal REPL in [sysl](https://github.com/edadma/sysl) — history, cursor
movement, and the key bindings people expect from a prompt.

**Nothing has to be installed to use this.** linenoise is about 2,400 lines of C and this package
carries them: sysl compiles a library's C as part of the build, so there is no `-l` flag, no
`pkg-config`, and no build script anywhere in this repository. That is also why there is no `@link`
in the binding's header — there is no external library to name.

```
sh/sysl/linenoise/
    linenoise.sysl      the binding
    linenoise.c         vendored from antirez/linenoise
    linenoise.h
package.hocon           who this package is, and what it needs of the machine
```

The module is **`sh.sysl.linenoise`**, and the three directories are that name: a dotted module name
mirrors its path from the library root. The prefix is the reverse-DNS of `sysl.sh`, so that a package
claims a name nobody else will mint rather than the top-level word `linenoise`.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```hocon
dependencies {
  linenoise { git = "github.com/sysl-lang/linenoise", version = "0.2.0" }
}
```

The coordinate is an identity rather than a URL, so it carries no `https://`, and `version` is the
tag `v0.2.0` here. Resolution clones it, selects versions by MVS, and records what arrived in
`sysl.sum`.

Or build it into an artifact and compile against that:

```
sysl build-lib . -o /tmp/linenoise.syslib
sysl run yourprogram.sysl --lib /tmp/linenoise.syslib
```

## Example

```sysl
import sh.sysl.linenoise.*

history_set_max_len(100)
history_load(".repl_history")

var going = true

while going
    read("> ") match
        None -> going = false
        Some(line) ->
            if line != "" then
                history_add(line)

            print(f"read: ${line}")

history_save(".repl_history")
print("bye")
```

`read` answers `None` at end of input — Ctrl-D on an empty line, or a pipe reaching EOF — which is
what a REPL loop ends on. An empty line is `Some("")` and is deliberately not the same answer.

## What it binds

| sysl | does |
|---|---|
| `read(prompt) -> Option[string]` | one line, or `None` at end of input |
| `history_add(line) -> bool` | add to the history the arrow keys walk |
| `history_set_max_len(len) -> bool` | how many lines the history keeps |
| `history_save(path)` / `history_load(path)` | persist it between runs |
| `clear_screen()` | as Ctrl-L does |
| `set_multiline(on)` | wrap a long line instead of scrolling sideways |
| `set_mask_mode(on)` | hide what is typed, for a password |

**Nothing is added to the history automatically.** A REPL usually wants to skip blank lines and
repeats, and a binding that decided that for you would be one you had to work around.

### The ownership rule, which is the whole of what the binding has to get right

linenoise hands back storage it allocated with `malloc`, and the caller has to return it with
`linenoiseFree`. A sysl `string` cannot own C's heap, so `read` copies the bytes out and frees the
original in the one place that can be sure it happens — on both the success and the decode-failure
path, before the result is inspected. The `extern`s are not public precisely so that no caller
inherits a rule they would have to read C to learn.

### Not bound yet

**Completion and hints**, which are the two callback APIs — `linenoiseSetCompletionCallback` and
`linenoiseSetHintsCallback` take C function pointers, and that is a larger question than the rest of
this binding put together. Everything here is one string in and one string out.

The **non-blocking API** (`linenoiseEditStart` / `EditFeed` / `EditStop`) is also absent. It exists
for programs driving their own event loop, and it needs the `linenoiseState` struct laid out rather
than kept opaque.

## Upstream

Vendored from [antirez/linenoise](https://github.com/antirez/linenoise). It is BSD-2 licensed and
copyright 2010–2023 Salvatore Sanfilippo and Pieter Noordhuis; see `LICENSE`, which carries their
notice as redistribution requires.

Vendoring rather than linking is a deliberate choice and it has a cost worth stating: **upstream
fixes do not arrive on their own.** linenoise is small, stable, and low-activity, which is what makes
that an acceptable trade here — it would not be for a library under active development.
