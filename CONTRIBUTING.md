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

## Sending a change

Two branches, and that is all: **`dev`** is where work lands, **`main`** is what
has been released. Open your pull request against `dev` - it is the default
branch, so a fork targets it without you having to think about it.

You cannot push here, and that is the point: fork, branch off `dev`, and open a
pull request. CI runs the gates on every one, on Linux and Windows, and `dev`
will not take a merge until they are green. Nothing lands unread and unproven.

`main` only moves for a release, and only by the maintainer.

What gets merged: one thing at a time, small enough to read in a sitting, with
the gate that proves it. What does not: a rewrite nobody asked for, a change
with no way to tell whether it works, or a diff that mixes a fix with a
reformat. If you are unsure whether something is wanted, open an issue first
and ask - that costs you nothing and saves you an afternoon.

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
