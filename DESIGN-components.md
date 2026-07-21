# Astra components — design (settled), implementation pending

Status: **design locked, not yet built.** This is the blueprint; the code is
a focused next step. Grounded in the real AST (`astra_ast/lib.or`):
a widget is `SpawnEntity(Text, [FieldExpr])`, `each` is
`ForEach(Text, Expr, Expr, [Stmt])`.

## The problem

The palette row today is a flat mega-property Panel — no structure, not reusable:

```
each commands as cmd:
    Panel style: cmd_row, image: cmd.id, text: cmd.label, hint: cmd.hint, do: cmd.id, search: palette_query
```

## The feature: `component NAME(params):` + a widget subtree

```
component CommandRow(cmd, query):
    Panel style: cmd_row, do: cmd.id, search: query
        Panel image: cmd.id, size: 20
        Text cmd.label, color: ink, size: 15
        Spacer
        Text cmd.hint, color: ghost, size: 13

each commands as cmd:
    CommandRow(cmd, palette_query)
```

A component is a **named, parameterized widget subtree**. An instantiation
`CommandRow(cmd, palette_query)` expands the subtree with params substituted,
re-parented under the enclosing container.

## Desugar — pure parser sugar, NO runtime/eval change, NO enum churn

The instantiation is expanded **at parse time**, so it never becomes a
persistent `Stmt` variant (avoids the "new enum variant breaks every
exhaustive match" tax — see orion-lang-gotchas).

1. **AST**: add `pub data ComponentTemplate: name: Text, params: [Param], body: [Stmt]`
   to `astra_ast` (a `data` struct, not an enum variant → no match churn).
2. **ParseResult**: add field `component_templates: [ComponentTemplate]`
   (update the 2 `ParseResult{…}` construction sites in `astra_parser`;
   ParseResult is field-accessed, not exhaustively matched, so low ripple).
3. **Parse the decl** `component NAME(params):` — reuse the param loop from
   `parse_template_decl` (LParen … Comma … RParen) + parse the indented body
   the same way `each` parses its body block (a `[Stmt]` of `SpawnEntity`/
   nested `ForEach`).
4. **Expansion post-pass** over every tree body: when a statement is
   `Ident(args)` where `Ident` ∈ component names, splice in the component's
   body with `subst(param → arg_expr)` applied, then run the existing
   `infer_foreach_body` / parent-inference so the spliced widgets re-parent
   + get `order` under the call site.
5. **`subst_expr` / `subst_stmt` / `subst_field`** mirror `ast_copy_expr` /
   `ast_copy_stmt` but replace `VarRef(param)` with its arg expression.
   ~30 lines; the only genuinely new code.

**Disambiguation**: a capitalized `Ident(` at tree position is a component
call iff the name is a declared component; `Panel`/`Text` stay widget spawns.
(The `component` keyword is free — it was cubsy-era and removed; only `type`
declares ECS components now.)

## Reserved but NOT built yet (designed so they slot in without churn)

- **`local NAME: type = v`** inside a component body → per-instance state.
  Deferred: the palette row is pure (params → subtree). When the first
  stateful component appears (e.g. a collapsible tree node with `local
  expanded: bool`), `local` gets a per-instance resource key namespaced by
  the instance's entity id.
- **`bind:` scope** becomes lexical once `local` exists: a `local NAME` in
  the component wins over a shared `state NAME` (same rule as var scoping).
  Until `local` exists, `bind:` targets shared state as today.
- **Slots** (`<slot>` / children passed to a component) → for wrappers like
  `Card` that enclose arbitrary content. Deferred; the palette row needs none.
- **`effect NAME(args)`** vs `emit`: `emit X` = an internal fact rules can
  observe via `on X`; `effect X` = an external op leaving the app (window/
  file/net), NOT observable — it feeds the provability moat ("what external
  effects does this command trigger"). Window ops move `emit WindowMin` →
  `effect WindowMinimize` when built. Deferred (separate mini-project).

## Collection-identity (`each X as x key x.id`) — the prerequisite

Discovered while grounding components in the real inference code:
`infer_foreach_body` **flattens** loop bodies (every widget re-parents to the
container; nested children in a loop need explicit `parent:`), and entity ids
**batch per type**, so a static `parent: "row"` for looped spawns binds every
child to the LAST row. → a *structured* row (icon | label | spacer | hint as
nested widgets) inside `each` needs **per-instance identity**. That's this
feature, and it also gives stable identity across re-renders (animation /
focus / diff — the moat).

Design: `key EXPR` stamps `__key` onto the loop body's root spawns (same
mechanism as `__eachcol`, **no ForEach enum change**). The runtime derives a
per-instance name from `__key` and resolves nested `parent:` within the same
keyed instance.

Increments (each build+verify with `orbit shot`):
1. **✅ DONE — parser** `key EXPR` clause + `stamp_body_key` → `__key` on root
   spawns. Verified: existing keyless `each` unchanged (check green, render
   identical). No behaviour yet — scaffolding for the runtime.
2+3. **✅ DONE — `infer_keyed_body`** (astra_parser): a keyed loop body infers
   nesting by column (like infer_one_stmt) and names every widget
   `"__k{pos}_" + keyexpr` (root + children); children's `parent:` → the keyed
   root name. **Verified end-to-end** via `orbit shot`: the palette renders 4
   structured rows (icon child + label child, per-instance parented). Three
   bugs found + fixed along the way (each caught by the headless shot + a
   temporary entity dump — name/parent/box/order):
   - **root order** — a root in a keyed body must sort at the loop's slot
     (`order_val`), not a local `0`, else it collides with earlier siblings.
   - **`collect_flex_children` used `==`** (POINTER eq) on parent names —
     a keyed child's runtime-built parent string never matched the container's
     name object. Fixed to `text_bytes_eq` (content). This is why `check`
     passed (panel_named_exists already used text_bytes_eq) but layout didn't.
     General correctness fix, not keyed-only.
   - **child global order** — `order` is a GLOBAL paint index; a child at
     local `i` (1,2) painted FIRST, then the card painted over it. Fixed to
     `order_val + i` so the whole subtree paints at the row's z-slot.
   Note: a structured (container) row needs an explicit height — a nested
   auto-size container doesn't sum its children yet (general layout gap).
4. **✅ DONE — component sugar** (astra `30ae52f`): `component NAME(params):`
   + a widget subtree; `NAME(args)` at tree position parses to a SpawnEntity
   with __argN fields; expand_components (post-pass, before hoisting) splices
   the body in with params substituted (ct_subst_*) and columns/__eachcol/__key
   rebased to the call site. The palette migrated to `CommandRow(cmd,
   palette_query)` — renders identically (dump: byte-identical entities). No
   new Stmt variant, no runtime change.

ALL FOUR INCREMENTS DONE. `component` + `each … key` are the shipped
next-gen surface.

Polish landed after:
- **✅ search sees child text** (atlas `955d028`): filter_text_of gathers a
  row's own text + its direct children's text, so type-to-filter works on
  structured/component rows. Verified: "wind" → only the 3 "window" rows.

Polish landed after (cont.):
- **✅ content-fit sizing** (atlas `68b6b37`): a container child with no
  explicit w/h derives its size from its children, so a component row no
  longer needs `h:`. Explicit-sized widgets unchanged (base editor verified
  byte-identical).

Known limits (not blocking):
- **Component params don't substitute into SYMBOLIC-FIELD bareword values.**
  `search: query` parses `query` as a bareword atom (LitText), not VarRef, so
  ct_subst (which replaces VarRef) leaves it literal. Workaround: reference the
  global directly in the body (`search: palette_query`), or pass the param via
  a non-symbolic field. A real fix would substitute a bareword whose name
  matches a param — deferred.

## Build order (components, after collection-identity lands)

1. `ComponentTemplate` in astra_ast + `component_templates` in ParseResult.
2. `parse_component_template` (decl + subtree body).
3. `subst_{expr,stmt,field}` in astra_ast (mirror `ast_copy_*`).
4. Expansion post-pass in `parse()` (after decls collected, before
   parent-inference finalizes) — splice + subst + re-infer parent/order.
5. Migrate `atlas-editor/features/shell/shell.astra` palette to
   `component CommandRow`.
6. Verify: `orbit shot` — the palette must render pixel-identical to today
   (the migration is a refactor, output unchanged), then `orbit dev` +
   click a row to confirm `do:`/`search:` still route.
