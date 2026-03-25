# Verus PBaaS Documentation

Comprehensive documentation for the Verus PBaaS system, built using the
[Diátaxis framework](https://diataxis.fr/) (Tutorials, How-to Guides, Concepts, Reference).

## Status

**Early draft.** Documentation is being built bottom-up from live RPC testing
on vrsctest. Pages are verified against daemon behavior, not copied from
existing sources.

## Structure

```
docs/
  reference/        — RPC command reference (parameters, return values, examples)
  concepts/         — Explanations of how Verus systems work
  how-to/           — Task-oriented guides for specific goals
  tutorials/        — Learning-oriented walkthroughs (coming soon)
```

## Current coverage

- **Identity** — 11 RPC reference pages, 4 concept pages, 3 how-to guides
- **Marketplace** — 5 RPC reference pages, 1 concept page, 1 how-to guide
- **Multichain** — 2 RPC reference pages, 3 concept pages, 1 how-to guide
- **Data** — 6 RPC reference pages, 1 concept page, 3 how-to guides

## Methodology

All documentation is built bottom-up:

1. Test each RPC command live on vrsctest
2. Document parameters, behavior, edge cases with real output
3. Extract into presentable pages separated by Diátaxis quadrant
4. Claims are backed by verified daemon behavior
