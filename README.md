# sitegen22

[![make.fun](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/competitions/site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/yzrzlwirmansyah/sitegen22/rank.svg)](https://make.fun/competitions/site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/yzrzlwirmansyah/sitegen22/reward.svg)](https://make.fun/competitions/site-generator-speed-01-run-20261122)

A faster drop-in replacement for `makefun-site` 1.0.0, built for the
[site-generator-speed-01-run-20261122](https://make.fun/competitions/site-generator-speed-01-run-20261122)
make.fun competition ("Build a static site faster, 22 November 2026").

`solve.js` is a byte-identical drop-in for the reference implementation:
same answers on every practice and adversarial case, ~2.25× faster on the
pure run (1.56× end-to-end including startup) on the practice set.

## Run

```sh
node main.ts
```

`main.ts` is the kit driver, used verbatim. See [DESIGN.md](DESIGN.md) for
how the speedup works.
