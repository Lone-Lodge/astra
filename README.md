# Astra

[På svenska](README.sv.md)

A small typed scripting language for what a program decides and what it shows.
One `.astra` file is a bundle; a host embeds the language and gives the
declarations meaning. Games, tools, desktop apps: the language has no opinion
about which.

```
state sparks: nat = 0
state strike_cd: nat = 0

command strike needs strike_cd == 0:
    sparks = sparks + 1
    strike_cd = 3

every 1s if strike_cd > 0:
    strike_cd = strike_cd - 1

on sparks becomes 3:
    seen_flame = true
```

**[Syntax](docs/SYNTAX.md)** - every form that has a gate, on one page.

Nothing there reaches the machine. A rule proposes effects and the host commits
them or refuses. There is no file I/O, no sockets, no `eval`, no unbounded loop
and no escape hatch: the sandbox is the grammar, not a runtime guard. A rule that
cannot be written cannot be misused.

The whole effect vocabulary is seven cases (`orbs/astra_outcome`): set a field,
set a variable, spawn, spawn a typed kind, destroy, emit, relate. A host that
implements those seven has the entire language wired.

## The shape of a bundle

A bundle holds a **tree** or **rules**, never both. Indentation is the parent
chain in a tree, the same way it is the block structure in a rule.

```
use widgets

tokens color:
    ink: "#ece7dd"
    panel: "#26262e"

style action:
    role: button
    h: 56
    color: panel
    ink: ink

view can_strike = if strike_cd == 0 then 1 else 0

Panel name: root, anchor: fill, layout: column, gap: 14
    Text "sparks {sparks}", color: ink, size: 17
    Panel style: action, text: "STRIKE", do: strike, show: can_strike
```

Widgets are not built in. `use widgets` names a bundle that declares them as
ordinary types (`type Text x: int, text: text, ...`), so the widget vocabulary
is data a host owns, not grammar. `view NAME = expr` is a derived value
recomputed per tick, and `show:` gates a widget on one, hiding its whole
subtree with it.

`style` and `tokens` are global, so a design system is one bundle every other
bundle can name. `component NAME(params):` gives a named parameterised subtree,
and `each xs as x key x.id:` repeats one with stable per-item identity.

## Embedding

Astra is an orb. A host depends on `orbs/astra` and calls the front door in
`astra_run`:

```
compiled  = compile(source)                        # parse + check, cache per bundle
outcome   = dispatch(source, "Click", args)        # an event into the rules
tick_out  = tick(source, ctx, prev_truths)         # fire when/every on the edge
errors    = astra_check_source(source)             # the loud pass, before a reload
results   = run_tests(source)                      # the bundle's own inline tests
```

`dispatch` merges every matching handler's effects. `tick` fires a `when` on the
false-to-true edge only, so a condition that stays true does not re-fire.
`astra_check_source` is deliberately separate from evaluation: live eval is
graceful (an unknown call answers `none`, so a hot reload mid-edit never crashes
a running program), and the check pass is the loud one a tool runs first.

Atlas embeds Astra as `atlas_astra`, mapping `Value` to its own wire format.

## Build and check

Astra is written in Orion and builds with Orion's `orbit`. Clone it beside
[orion](https://github.com/Lone-Lodge/orion) in the same workspace; the
toolchain finds the standard orbs by sweeping from the workspace root.

```sh
orbit run                    # the language suite, 76 checks
cd test_parse && orbit run   # parse smoke over whole bundles
```

| gate | what it proves |
|---|---|
| `orbit run` | every declaration, statement and expression the language has |
| `test_parse` | whole bundles off disk parse, the app template's among them |

## Layout

```
orbs/astra          the facade: one dependency for a host
orbs/astra_lexer    text -> tokens
orbs/astra_parser   tokens -> AST, and every piece of sugar
orbs/astra_ast      the tree
orbs/astra_check    the loud pass: unknown call, wrong arity, duplicate name
orbs/astra_eval     walk the tree, propose effects
orbs/astra_run      the front door: compile, dispatch, tick, run tests
orbs/astra_value    the value universe (int, text, bool, ref, map, list, none)
orbs/astra_outcome  the closed effect set
orbs/astra_host     the seam a host implements
src/main.or         the suite
fixtures/           .astra programs the suite reads
```

## Status

v0.1.0. Green on Linux and Windows, and embedded in real programs.

Every word the lexer reserves has a gate or a live user. The suite covers the
declarations a host reads (`state`, `type`, `trait`, `entity`, `const`, `data`,
`source`, `animate`, `asset`, `template`, `tokens`, `style`, `view`), the rules
(`rule`, `command needs`, `on`, `when`, `every`, `law`, `patch`, `intent`,
`fact` and its family), the statements (`create`, `destroy`, `set`, `relate`,
`emit`, `effect`, `signal`, `after`, `each`/`for`, `if`) and the expressions
(`match`, `count ... where`, `all`, `exists`, `none`, ranges, text).

Known gaps:

- **`use` is a note to the host, not an import.** A bundle is a file and its
  path is its identity. `use NAME` is recorded in the parse result and the host
  decides what to load; names that cross bundles (styles, tokens, types) are
  simply global.
- **`astra_host` is a seam, not a protocol.** It carries the query shape today;
  a host still bridges by hand, the way `atlas_astra` does.
- **No generics in the value universe.** Absence is an explicit `NoneV` variant
  because Orion enums do not carry generics yet.
- **Hot reload is the host's.** The language is reload-safe (a failed compile
  leaves an empty program and says why) but does not own the reload itself.

## License

Apache 2.0. Copyright 2026 Lone Lodge. See [LICENSE](LICENSE) and
[NOTICE](NOTICE).
