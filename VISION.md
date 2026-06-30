# Astra — next-gen DSL for authored game logic

> One language for game rules. AI-native, time-travel-aware, sandbox-by-grammar.
> Hot-reloadable. Replay-correct. Designed for 2026, not 1995.

## What Astra is

A **small typed host-agnostic language** for the logic of a game. A rule is a
pure function from inputs to **proposed Effects** — the host (Atlas) decides
whether to commit them.

Astra writes the **decisions**. Atlas runs the **engine**. Orion writes the
**systems** underneath both. Each layer owns exactly what it's best at:

```
┌──────────────────────────────────────────────────────────┐
│  Astra  — rules / intents / behaviors  (designers, AI)   │
│           write-side: emits Effects                       │
├──────────────────────────────────────────────────────────┤
│  Veil   — views / UI / render trees     (designers, AI)  │
│           read-side: declares Views                       │
├──────────────────────────────────────────────────────────┤
│  Atlas  — world / ECS / render / input  (engine devs)    │
│           tick-log, effects, channels                    │
├──────────────────────────────────────────────────────────┤
│  Orion  — systems language               (lib authors)   │
│           native-compiled, full FFI                      │
└──────────────────────────────────────────────────────────┘
```

**Astra and Veil are siblings.** Astra writes; Veil shows.
- Astra rule: `on_click(x, y) → emit Place { cell }`
- Veil view: `Hud → stack [ score, health, minimap ]`
Both compile to pure declarations. Both are hot-reloadable. Both are
LLM-friendly. Neither touches state directly.

## What Astra is NOT

- **Not a general-purpose language.** No `printf`, no sockets, no file I/O.
  Those are Effects the host commits.
- **Not Lua/Rhai with extra syntax.** Those embed *general* scripting;
  Astra is a *vertical* DSL for game logic specifically.
- **Not Verse/Unreal Blueprints.** Astra is text-first, AI-friendly,
  text-diffable. No graph-of-nodes editor required.
- **Not data-only (JSON/YAML).** Astra has computation — rules, conditions,
  bindings — without giving up sandboxing.

## The seven pillars

What makes Astra **distinct** from every other embedded scripting language.
Each pillar is **unique**, not "better lua".

### 1. AI-native primitives

`intent`, `belief`, `goal`, `prompt`. AI authoring is a first-class concern
of the grammar, not a library bolted on. An LLM looking at an Astra file
sees declarative goals, not control flow it has to reverse-engineer.

```astra
intent FeedSelf for Worker:
    require self.hunger > 50
    plan: [find_food, walk_to(food), eat(food)]
    utility: self.hunger / 100
```

### 2. Behavior trees / state machines / GOAP as grammar

Standard game AI primitives are **syntax**, not patterns. No more
"which BT library do we use" — `sequence:`, `selector:`, `parallel:`,
`state X:`, `plan X:` are part of the language.

```astra
behavior Guard:
    selector:
        sequence: [enemy_visible?, chase]
        sequence: [at_post?, idle]
        return_to_post
```

### 3. Time-travel as a language feature

Rule lives on a tick-log. Astra exposes it: `at tick N`, `replay`,
`rewind to event`, `branch from`, time-windowed aggregates. Debugging
becomes *inquiry from inside the language*.

```astra
rule alert_grief:
    let recent_kills = count(emit Kill from last 30 seconds where by == self)
    require recent_kills > 5
    emit FlagForReview { player: self }
```

### 4. Reactive (when/every/until)

State-change triggers are syntax. No `on_health_change` callback to register.
The runtime auto-fires rules when their `require`s become true.

```astra
when player.hp < 10:
    emit LowHealthWarning { player }

every 5 seconds:
    emit RegenerateMana { amount: 1 }
```

### 5. Tests + contracts first-class

`test`, `property`, `golden`, `bench` are statements. A rule's
`require`/`expect` clauses double as test specifications — the test
runner derives counterexamples automatically.

```astra
rule place_block(cell):
    require cell.empty
    emit Place { cell }
    expect cell.occupied

test cannot_place_twice:
    apply place_block(c1); tick 1
    apply place_block(c1)
    expect effects == 0   # second call refused
```

### 6. Live coding + hot reload as semantics

A `live rule` rebinds on file save. The running game picks up changes
within one tick. No restart, no "edit-and-continue" magic — it's just
how the language works.

```astra
live rule difficulty_curve:
    let n = world.tick / 600    # increase every 10 seconds
    set spawn_rate = base_rate + n * 0.5
```

### 7. Safety by construction

The sandbox is the **grammar**. There is no `dangerous_function`. There is
no need to whitelist or audit calls. A rule that **can't be written** can't
be misused. Every IO is an Effect the host commits.

- No `while true` (loops are bounded by `count` or query)
- No raw memory or pointers
- No mutation outside effects
- No nondeterminism (`random`, `now()` are effects — host supplies seed)
- `pure rule X:` checker-enforces no host calls

## Effect universe — closed set

Every write the language can propose. Nothing more is needed:

| Effect | Means |
|---|---|
| `Set entity.field = value` | Field write |
| `Spawn Kind { ... }` | New entity |
| `Destroy entity` | Retire entity |
| `Emit Event { ... }` | Signal (host may dispatch to handlers) |
| `Relate a kind b` | Bidirectional relation |
| `Unrelate a kind b` | Remove relation |

Host (Atlas) folds each into the tick-log. Replay reproduces every game.

## How it compares

| Need | Lua | Rhai | Wren | Verse | **Astra** |
|---|---|---|---|---|---|
| Embeddable | ✅ | ✅ | ✅ | ❌ (Unreal-only) | ✅ |
| Typed | ❌ | partial | ❌ | ✅ | ✅ |
| Sandbox by grammar | ❌ | ❌ | ❌ | ✅ | ✅ |
| Hot reload | manual | manual | ❌ | ✅ | **built-in** |
| Behavior trees | library | library | library | library | **built-in** |
| Replay-correct | ❌ | ❌ | ❌ | ❌ | **built-in** |
| Time-travel queries | ❌ | ❌ | ❌ | ❌ | **built-in** |
| AI-native primitives | ❌ | ❌ | ❌ | ❌ | **built-in** |
| Tests as syntax | ❌ | ❌ | ❌ | partial | ✅ |
| LLM-friendly | poor | OK | OK | OK | **designed for it** |

## Status today (Dec 2026)

Shipped in pure Orion (`astra/orbs/`):
- ✅ `astra_lexer` — full token vocab
- ✅ `astra_parser` MVP — `rule name(params): emit Event(...)`
- ✅ `astra_eval` MVP — walk AST, bind params, accumulate effects
- ✅ `astra_run` — lex/parse/eval pipeline
- ✅ `atlas_astra` — bridge: Astra Effects → atlas world

Cubsy v1.0 dispatches every click through a `.astra` rule. The wiring
proves the chain end-to-end. **The grammar today is 5% of the vision.**

## Roadmap

### M1 — Make rules useful (1-2 sessions)

The grammar piece needed before any *real* game logic can move to Astra.

- [ ] **Expressions** — binops with precedence climbing (`a + b * c`)
- [ ] **`require`** — guard statement (silent no-op when false)
- [ ] **`let`** — local binding
- [ ] **`if cond then a else b`** — branching expressions
- [ ] **Field access** — `cell.col`, `player.hp`
- [ ] **Function calls** — `pixel_to_col(x)`
- [ ] **Comparison + boolean** — `==`, `<`, `and`, `or`, `not`

Outcome: cubsy's `handle_tray_click` validation can move to `.astra`.

### M2 — Reactive + Behavior (1-2 weeks)

The pillars that make Astra **distinct** from Rhai.

- [ ] **`when cond: ...`** — auto-fired when state matches
- [ ] **`every N: ...`** — periodic rules
- [ ] **`behavior X: sequence: ...`** — BT primitives
- [ ] **`state X: on enter ...`** — FSM as syntax
- [ ] **`on Event(args): ...`** — handler dispatch

Outcome: cubsy AI opponent in `.astra` (auto-place piece every N ticks).

### M3 — AI-native + Time-travel (2-4 weeks)

The features that make Astra **next-gen**.

- [ ] **`intent X for Y: ...`** — declarative goal blocks
- [ ] **`goal X utility: ...`** — utility AI
- [ ] **`plan X: ...`** — GOAP/STRIPS
- [ ] **`at tick N: ...`** — historical queries
- [ ] **`replay file.tlog: ...`** — re-run sessions as syntax
- [ ] **`live rule X: ...`** — hot-reload as semantics

Outcome: Astra is now a **scripting language no other engine has**.

### M4 — Tests + Authoring (1-2 weeks)

The features that make Astra **comfortable**.

- [ ] **`test X { apply Y; tick N; expect Z }`** — inline tests
- [ ] **`entity X { ... }`** — authored entities
- [ ] **`record X { field: type }`** — schemas
- [ ] **`view X: ...`** — read-side render trees
- [ ] **`property X for_all: ...`** — property-based tests
- [ ] **`tweak X: 0..10 default 3`** — designer sliders

### M5 — MMO + Determinism (longer-term)

- [ ] `synced rule X:` — deterministic across peers
- [ ] `replicate field:` — auto-network sync
- [ ] `predict / reconcile` — netcode primitives

## Anti-goals

- **No general-purpose escape hatch.** No `eval`, no `dlopen`, no `exec`.
  If a rule needs a capability, the host adds an Effect for it.
- **No control flow outside rules.** No top-level `while true`. Top-level
  is declarations only.
- **No silent imperative mode.** Every state change goes through an Effect.
- **No "but Lua does it"** as a reason to add a feature. Astra is opinionated.

## Open questions

1. **Effects with multi-field payloads** — `astra_value` currently has
   `IntV/TextV/BoolV/RefV/NoneV`. Add a `MapV([(Text, Value)])` to support
   `emit Spawn { hp: 100, name: "Goblin" }`?
2. **Module system** — `import "ai/behaviors.astra"` syntax + how it
   composes when multiple files define rules?
3. **Reactive trigger storage** — `when X: ...` needs the runtime to know
   what state-change to watch. Tied into atlas_ecs's reactive flags
   (added/changed/removed)?
4. **AI-prompt-as-comment grammar** — `# prompt: spawn enemies` — should
   this be standardized so editors can offer LLM-fill?
5. **Hot-reload determinism** — when a rule swaps mid-session, are
   prior emits still valid? Need rebinding semantics.

## Naming

The language is **Astra**. The file extension is `.astra`. The grammar
prefers lowercase keywords (`rule`, `let`, `require`, `emit`).
`AstraEffect` is the runtime tag used in host bindings (so callers
know it's distinct from atlas's own `Effect`).
