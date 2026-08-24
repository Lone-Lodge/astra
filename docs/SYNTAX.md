# Astra syntax

Every form here is exercised by a gate (`orbit run`, `test_parse`) or by a
running program. Forms that exist in the parser but have neither are listed as
gaps in the [README](../README.md) and are deliberately absent from this page.

A bundle is one `.astra` file. Indentation is structure: the block of a rule,
the parent chain of a tree. `#` starts a comment. A bundle holds a tree or
rules, never both.

## Facts

```
state sparks: nat = 0
state door: enum(Shut, Open, Locked) = Shut
```

A `state` is what the host stores and what a rule may change. An `enum` state
is checked for exhaustiveness: if some `when` branches switch on it by `==`
and a variant has no branch, the check pass says so. Any other reference to
the state (`!=`, a call, a field) suppresses that check.

```
type Text x: int, text: text, color: text, size: int
entity Block { col: int, row: int, color: Text }
```

`type` declares a record the host registers; a widget vocabulary is just a
bundle of these. `entity Kind { ... }` is the authoring form the host uses to
build one from `create`.

## Rules

```
rule scale(x): require x > 0 let d = x * 2 emit Scaled()

command strike needs strike_cd == 0:
    sparks = sparks + 1

on Click(x, y): require x > 0 emit Confirmed()
on sparks becomes 3: seen_flame = true

when score >= 100: emit Milestone()
every 3: emit Pulse()
every 1s if strike_cd > 0: strike_cd = strike_cd - 1
```

`rule` is called by name. `command` is a user action: `needs` is its
precondition and a widget dispatches it with `do: NAME`. `on Event(args)` is a
handler; every handler that matches an event fires, and their effects merge.
`on X becomes Y` fires on the transition, once.

`when` fires on the false-to-true edge only, so a condition that stays true
does not fire again. `every N` counts ticks, `every N s` counts wall-clock
seconds, and `if` adds a guard.

A failed `require` proposes zero effects. An illegal move is a silent no-op,
never an error.

## Inside a rule

```
let d = x * 2          # explicit binding
doubled = x * 2        # the same thing, implicitly

if x > 5: emit Big()   # single-branch guard

each cell in cells:
    create Block col: col + cell.dc, row: row - cell.dr

for r in 0 to 3:       # `for` is a spelling of `each`
    when r == 2: emit Hit row: r
```

Ranges: `0 to 3`, `from 1 to 5`, `1 through 4`, `from 2 through 6`. `to` is
half-open, `through` includes the last value.

Expressions carry the usual precedence, comparison and `and` / `or` / `not`.
`if COND then A else B` is an expression. `+` concatenates when either side is
text, and `"n={n}"` interpolates.

Queries over a list:

```
count(b in items where b.color == "red")
all x in [2, 4, 6] where x > 0
```

## Effects

```
emit Place col: 3, row: 5, color: "red"
create Block col: 1, row: 2, color: "red"
sparks = sparks + 1
```

`emit X` is an internal fact other rules can observe with `on X`. `create Kind`
builds an entity from an `entity` declaration. A bare assignment writes a
`state`.

## Helpers

```
fn double(n: int) -> int: n * 2

double(x):                 # types are optional
    x * 2
```

A helper is pure and callable from any rule expression. A call to a name that
is neither a builtin nor a declared helper is what the check pass catches.

## Trees

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

Panel name: root, layout: column, gap: 14
    Text "sparks {sparks}", color: ink, size: 17
    Panel style: action, text: "STRIKE", do: strike, show: can_strike
```

A widget line is `Type field: value, ...`, and indentation parents it. `style`
is a reusable bundle of fields, `tokens NS:` a namespace of named values; both
are global across bundles. `view NAME = expr` is a derived value recomputed
each tick, and `show:` gates a widget on one, hiding its whole subtree.

```
component CommandRow(cmd, query):
    Panel style: row, do: cmd.id
        Text cmd.label, size: 15

each commands as cmd key cmd.id:
    CommandRow(cmd, palette_query)
```

A `component` is a named parameterised subtree, expanded where it is called.
`key EXPR` gives each pass of a loop its own identity, so nested children
parent to their own row rather than to the last one.

```
template greeting(name) = "hello, " + name
```

A `template` is a parameterised expression fragment, invoked like a helper.

## Tests

```
test a:
    apply scale(5)
    expect effects == 1

test c:
    tick 3
    expect tick == 3
```

`apply` runs a rule, `tick N` advances the clock, and `expect` reads `effects`
or `tick`. The host runs them with `run_tests(source)`.

## The declarative side

```
intent FeedSelf for Worker:
    require hunger > 50
    utility hunger

fact alive = health > 0
fact reach = score > 100 in goal
fact enemy_at = 3 in belief:Guard conf 73
```

An `intent` is scored each tick into `intent:NAME:applicable` and
`intent:NAME:score`, which a host reads to pick an action. `fact NAME = EXPR`
is a derived value under a namespace: bare it is a `view`, `in goal` or
`in belief:WHO` place it elsewhere, and `conf N` records a confidence.
