# Contributing

## Build and prove it

Astra is written in Orion. Install the
[orion](https://github.com/Lone-Lodge/orion) toolchain; astra needs nothing
else.

```sh
orbit run                    # the language suite, 76 checks
cd test_parse && orbit run   # whole bundles off disk parse
python tools/guide.py --check  # the page still matches docs/SYNTAX.md
```

## The shape of a change

- **A new form needs a gate.** Every word the lexer reserves has one. Three
  words were once reserved and never parsed; they only took names away from
  whoever writes a bundle, and they are gone.
- **Document it in `docs/SYNTAX.md`.** The page is generated from it, so the
  reference and the site cannot drift. Run `python tools/guide.py` after.
- **The simplest thing that works**, and a comment only where the code cannot
  speak for itself.
- **One subject line, lower case, saying what is now true.**

## Where things live

```
orbs/astra_lexer     text -> tokens
orbs/astra_parser    tokens -> AST, and every piece of sugar
orbs/astra_check     the loud pass
orbs/astra_eval      walk the tree, propose effects
orbs/astra_run       the front door
src/main.or          the suite
fixtures/            .astra programs the suite reads
```
