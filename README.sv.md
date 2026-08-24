# Astra

[In English](README.md)

Ett litet typat skriptspråk för vad ett program bestämmer och vad det visar.
En `.astra`-fil är en bunt; en värd bäddar in språket och ger deklarationerna
sin mening. Spel, verktyg, skrivbordsappar: språket har ingen åsikt om vilket.

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

**[Syntaxen](docs/SYNTAX.md)** - varje form som har en grind, på en sida.

Ingenting där når maskinen. En regel föreslår effekter och värden verkställer
dem eller vägrar. Det finns ingen fil-IO, inga sockets, ingen `eval`, ingen
obegränsad loop och ingen nödutgång: sandlådan är grammatiken, inte en vakt vid
körning. En regel som inte går att skriva går inte att missbruka.

Hela effektordförrådet är sju fall (`orbs/astra_outcome`): sätt ett fält, sätt
en variabel, skapa, skapa av en typ, förstör, sänd, relatera. En värd som
hanterar de sju har hela språket inkopplat.

## Hur en bunt ser ut

En bunt håller ett **träd** eller **regler**, aldrig båda. Indenteringen är
föräldrakedjan i ett träd, precis som den är blockstrukturen i en regel.

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

Widgetarna är inte inbyggda. `use widgets` namnger en bunt som deklarerar dem
som vanliga typer (`type Text x: int, text: text, ...`), så widgetordförrådet är
data en värd äger, inte grammatik. `view NAMN = uttryck` är ett härlett värde som
räknas om varje tick, och `show:` grindar en widget på ett sådant och gömmer hela
dess underträd med den.

`style` och `tokens` är globala, så ett designsystem är en bunt som varje annan
bunt kan namnge. `component NAMN(params):` ger ett namngivet parametriserat
underträd, och `each xs as x key x.id:` upprepar ett med stabil identitet per
post.

## Att bädda in

Astra är en orb. En värd beror på `orbs/astra` och anropar ytterdörren i
`astra_run`:

```
compiled  = compile(source)                        # parse + kontroll, cacha per bunt
outcome   = dispatch(source, "Click", args)        # en händelse in i reglerna
tick_out  = tick(source, ctx, prev_truths)         # tänder when/every på flanken
errors    = astra_check_source(source)             # den högljudda passen, före omladdning
results   = run_tests(source)                      # buntens egna inbyggda tester
```

`dispatch` slår ihop effekterna från varje hanterare som matchar. `tick` tänder
ett `when` bara på flanken falskt till sant, så ett villkor som förblir sant
tänder inte om. `astra_check_source` är medvetet skild från evalueringen: levande
eval är förlåtande (ett okänt anrop svarar `none`, så en omladdning mitt i en
redigering aldrig kraschar ett program som kör), och kontrollpassen är den
högljudda som ett verktyg kör först.

Atlas bäddar in Astra som `atlas_astra` och mappar `Value` till sitt eget
trådformat.

## Bygg och kontrollera

Astra är skriven i Orion och byggs med Orions `orbit`. Klona den bredvid
[orion](https://github.com/Lone-Lodge/orion) i samma arbetsyta; verktygskedjan
hittar standardorbarna genom att svepa från arbetsytans rot.

```sh
orbit run                    # språksviten, 51 kontroller
cd test_parse && orbit run   # parse-rök över hela buntar
```

| grind | vad den bevisar |
|---|---|
| `orbit run` | dispatch, reaktiva flanker, frågor, text, inbyggda tester, kontrollpassen |
| `test_parse` | en riktig bunt med många deklarationer parsar, och mallen tur och retur håller |

## Karta

```
orbs/astra          fasaden: ett beroende för en värd
orbs/astra_lexer    text -> tokens
orbs/astra_parser   tokens -> AST, och allt socker
orbs/astra_ast      trädet
orbs/astra_check    den högljudda passen: okänt anrop, fel antal argument, dubbelt namn
orbs/astra_eval     gå trädet, föreslå effekter
orbs/astra_run      ytterdörren: compile, dispatch, tick, kör tester
orbs/astra_value    värdeuniversumet (int, text, bool, ref, map, list, none)
orbs/astra_outcome  den slutna effektmängden
orbs/astra_host     sömmen en värd fyller i
src/main.or         sviten
fixtures/           .astra-program som sviten läser
```

## Läge

v0.0.1. Grön, och inbäddad i riktiga program, men ung.

Vad som bevisas av en grind eller ett levande program: `state`, `type`, `rule`,
`command needs`, `on Event`, `on X becomes Y`, `when`, `every N s`, `fn`,
`view`, `component`, `each ... key`, `style`, `tokens`, `template`, `test`,
`intent`, `fact` med familj (`goal`, `constraint`, `belief`, `conf`), `emit`,
`create`, `entity`, `all`, `count ... where`.

Kända luckor:

- **Deklarerade men ogrindade.** `law`, `source`, `animate`, `asset`, `patch`,
  `trait`, `record`, `relate`, `exists`, `derive`, `signal`, `const`, `match`,
  `effect`, `spawn` och `destroy` finns i lexern och parsern men har varken test
  eller användare. Betrakta dem som ofärdiga tills var och en har en grind.
- **`use` är en notis till värden, inte en import.** En bunt är en fil och dess
  sökväg är dess identitet. `use NAMN` skrivs ner i parse-resultatet och värden
  avgör vad som ska laddas; namn som korsar buntar (styles, tokens, typer) är
  helt enkelt globala.
- **`astra_host` är en söm, inte ett protokoll.** Den bär frågeformen idag; en
  värd kopplar fortfarande för hand, så som `atlas_astra` gör.
- **Inga generiska typer i värdeuniversumet.** Frånvaro är en uttrycklig
  `NoneV`-variant eftersom Orions enum:ar inte bär generics än.
- **Omladdningen är värdens.** Språket är omladdningssäkert (en misslyckad
  kompilering lämnar ett tomt program och säger varför) men äger inte
  omladdningen självt.

## Licens

Apache 2.0. Copyright 2026 Lone Lodge. Se [LICENSE](LICENSE) och
[NOTICE](NOTICE).
