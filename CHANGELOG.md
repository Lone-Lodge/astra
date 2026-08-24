# Changelog

Notable changes to Astra. Format follows [Keep a Changelog](https://keepachangelog.com/),
versions follow [Semantic Versioning](https://semver.org/).

## [0.1.0] - 2026-08-24

First public release. The language is not new - it has been embedded in real
programs for months - but this is the first version anyone else can read.

### Added
- **A gate for every reserved word.** The suite went from 51 checks to 76,
  covering the declarations a host reads (`const`, `data`, `law`, `trait`,
  `source`, `animate`, `asset`, `patch`), the statements (`set`, `destroy`,
  `relate`, `signal`, `effect`, `after`), the expressions (`match`, `exists`,
  `none`) and the `fact` family. Nothing in the lexer is unproven now.
- **CI.** `green.yml` bootstraps Orion from its committed seed and runs both
  suites on Linux and Windows, from a checkout with no binary on disk. Manual
  only: a push does not need a runner to repeat what `orbit run` already said.
- **The docs a reader needs**: a README in English and Swedish, and
  [docs/SYNTAX.md](docs/SYNTAX.md), which lists every form the language has.

### Fixed
- **A rule body now ends at `patch` and `law`.** Both were missing from the
  rule-starter set, so a `patch command` after any rule body was parsed as a
  statement and failed. The feature could not be used in the shape it was
  written for.
- **`trait X f: t` is the real spelling.** The comments promised
  `trait X { f: t }`, but traits share the type parser, which rejects braces.

### Removed
- **The path dependency into the orion checkout.** Astra used to reach the `log`
  orb through `path:../orion/orbs/log`, which meant a clone only built as a
  sibling of orion. The toolchain now puts its own orbs on the near side
  (orion `5a0bf1f`), so astra depends on nothing but the toolchain running it.
- **`spawn`, `record` and `derive` are no longer reserved.** All three were in
  the lexer and none was ever parsed, so they only took three words away from
  whoever writes a bundle. `create` is the spelling for a spawn.

### Changed
- **`test_parse` exits with the number of bundles that failed to parse**, so it
  can gate CI instead of only printing for a human to read.
- The `fact` probe folded into the main suite. One suite, one exit code.

[0.1.0]: https://github.com/Lone-Lodge/astra/releases/tag/v0.1.0
