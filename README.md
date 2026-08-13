# linenoise

Line editing for a terminal REPL in [sysl](https://github.com/sysl-lang/sysl) — history, cursor
movement, and the key bindings people expect from a prompt.

**Nothing has to be installed to use this.** linenoise is about 2,400 lines of C and this package
carries them: sysl compiles a library's C as part of the build, so there is no `-l` flag, no
`pkg-config`, and no build script anywhere in this repository. That is also why there is no `@link`
in the binding's header — there is no external library to name.

```
sh/sysl/linenoise/
    linenoise.sysl      the binding
    tests.sysl          9 tests, none of which needs a terminal
    c/
        c.sysl          linenoise as C declares it
        linenoise.c     vendored from antirez/linenoise
        linenoise.h
package.hocon           who this package is, and what it needs of the machine
```

**Everything that is C lives in `c/`**, module `sh.sysl.linenoise.c`. That is the two-layer shape every
binding in this organisation uses: the `c` layer has to be *faithful*, because a signature that
disagrees with the header links perfectly and corrupts the call at run time, and the layer above it has
to be *pleasant*, which is a different question that would otherwise be answered in the same breath.

**This is the smallest raw layer in the organisation, and it is empty of everything but `extern`s.**
There is no shim, because every entry point linenoise publishes takes scalars and pointers; no
`c const`, because the library has no `#define` a caller needs — the two mode switches are separate
functions rather than flags; and no `impl Drop`, because there is no handle. The only allocation
linenoise makes is the line it just read, and it is freed before the string is even inspected. So the
split here buys the one thing it always buys: ten raw C entry points that were part of this package's
published surface are behind `c.` now.

The module is **`sh.sysl.linenoise`**, and the directories are that name: a dotted module name
mirrors its path from the library root. The prefix is the reverse-DNS of `sysl.sh`, so that a package
claims a name nobody else will mint rather than the top-level word `linenoise`.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```hocon
dependencies {
  linenoise { git = "github.com/sysl-lang/linenoise", version = "0.3.0" }
}
```

The coordinate is an identity rather than a URL, so it carries no `https://`, and `version` is the
tag `v0.3.0` here. Resolution clones it, selects versions by MVS, and records what arrived in
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
| `history_save(path) -> bool` / `history_load(path) -> bool` | persist it between runs; `false` is a filesystem that would not cooperate, and a load of a file that is not there yet |
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

## Tests

```
sysl test .
```

**The history is what a suite can reach, and it is most of the surface.** Nine cases cover adding and
its duplicate rule, the maximum length and which end lines fall off, the file format as bytes, both
failure answers, a load round trip, and an entry with a newline in it.

**`read` is not covered and cannot be.** It puts the terminal into raw mode and asks its size with
`ioctl`, so a test process — whose stdin is a pipe — sends linenoise down its no-tty path, where it
reads a line with `fgets` and hands it back unedited. Testing that would test `fgets`, not the
editing. A real test needs a pseudo-terminal, which means `openpt`, `TIOCSWINSZ` and a `struct
winsize` — more C, and C in this package is C every consumer compiles.

Multi-line mode, mask mode and `clear_screen` are covered only as far as *linking*: they are `static
int`s and escape codes with no accessor between them, so the one thing observable is that the symbols
they name exist.

## Upstream

Vendored from [antirez/linenoise](https://github.com/antirez/linenoise). It is BSD-2 licensed and
copyright 2010–2023 Salvatore Sanfilippo and Pieter Noordhuis; see `LICENSE`, which carries their
notice as redistribution requires.

Vendoring rather than linking is a deliberate choice and it has a cost worth stating: **upstream
fixes do not arrive on their own.** linenoise is small, stable, and low-activity, which is what makes
that an acceptable trade here — it would not be for a library under active development.
