# @snrgy/dep-audit

Objective dependency footprint measurement with transparent methodology.

## Why this tool exists

Dependency analysis tools tend to produce "risk scores" and "severity ratings" that bake in editorial opinions about what constitutes a dangerous or unhealthy dependency. This tool takes a different approach: it measures raw properties of your dependency tree and reports them as numbers. You decide what the numbers mean.

Think of it like a scale: it measures and reports. Whether 400 transitive dependencies is acceptable for your project is your call.

## Status

Not yet implemented. Coming soon.

## What will be measured

- Direct and transitive dependency counts
- Median package age (days since last publish)
- License breakdown by type
- Duplicate package versions in the tree
- Packages with known CVEs (count only — no severity theater)
- Unused dependencies detected via static import analysis (probabilistic, flagged as such)

## Philosophy

- Raw metrics only — no risk scores, no health grades
- Transparent about what is excluded and why
- Statistical distributions, not just totals
- Every report includes methodology and threats to validity

## License

MIT
