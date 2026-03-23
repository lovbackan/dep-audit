# dep-audit — Task Map

Status markers: `[ ]` todo · `[~]` in progress · `[x]` done

---

## Phase 0 — Spec & design (complete before writing code)

The README has a high-level outline, but several measurement decisions need to be locked in before implementing. Document decisions in README before moving to Phase 1.

- [ ] **Package manager scope:** npm only for v1 (parse `package-lock.json`). Yarn and pnpm deferred — note this explicitly in README limitations.
- [ ] **Transitive tree source:** use `package-lock.json` v2/v3 `packages` field rather than walking `node_modules` — more reliable, doesn't require install to have run. Fallback: `node_modules` walk if no lock file.
- [ ] **Package age source:** npm registry API (`registry.npmjs.org/<name>`) — latest publish date. Decide: do we call the registry at runtime, or require offline mode? Proposal: call registry by default, `--offline` flag to skip.
- [ ] **CVE source:** run `npm audit --json` as subprocess and parse output. Do NOT implement our own CVE database. Note limitation: npm audit requires network access and a lock file.
- [ ] **Unused dependency detection:** static analysis — scan `src/` for `import`/`require` statements and match against `dependencies` in `package.json`. Flag as probabilistic (dynamic imports, build-time plugins, CLI binaries may not appear as imports).
- [ ] **Finalize full metric list and write detailed README** (same structure as `code-size` README)

---

## Phase 1 — Project setup

- [ ] Install dependencies: `commander`, `tsx`, `typescript`
- [ ] `tsconfig.json`
- [ ] `src/index.ts` — CLI entry point
- [ ] Verify `--help` works

---

## Phase 2 — Dependency tree parsing

Goal: from `package.json` + `package-lock.json`, extract the full dependency tree.

- [ ] Load `package.json` — extract `dependencies`, `devDependencies`, `peerDependencies`
- [ ] Classify each as `direct-prod`, `direct-dev`, `direct-peer`
- [ ] Parse `package-lock.json` v2/v3 `packages` field for full transitive tree
- [ ] Fallback: if no lock file, walk `node_modules` with warning
- [ ] Count: direct prod, direct dev, total transitive
- [ ] Detect duplicate versions: same package name with multiple semver versions in the tree
  - Report: package name, versions present, count of each
- [ ] Unit tests: correctly counts tree from fixture lock files

---

## Phase 3 — Package metadata (requires network)

Goal: for each **direct** dependency, fetch publish date from npm registry.

- [ ] Fetch `https://registry.npmjs.org/<name>` for each direct dep
- [ ] Extract `time.<version>` for the installed version → compute age in days
- [ ] Batch requests with concurrency limit (e.g. 10 at a time) to avoid hammering registry
- [ ] Graceful failure: if registry is unreachable, report "age unavailable" per package — don't crash
- [ ] `--offline` flag: skip all registry calls, report age as unavailable
- [ ] Report: distribution of package ages (P10/P50/P90 in days), oldest packages (top 10), packages not updated in >2 years (raw count — no "outdated" label, just age)

**Note:** We report age as days since last publish, not a "freshness score". What age is acceptable depends on the package ecosystem and stability expectations — that's your call.

---

## Phase 4 — License analysis

Goal: report the license breakdown of all dependencies (direct + transitive).

- [ ] Read `license` field from each package's `node_modules/<name>/package.json`
- [ ] Group by SPDX license identifier
- [ ] Report: count per license type, sorted by count descending
- [ ] Flag: packages with no `license` field (count + list)
- [ ] Flag: packages with non-standard license strings (can't parse as SPDX)
- [ ] Do NOT classify licenses as "permissive" / "copyleft" / "risky" — report the raw identifier. License compliance depends on your use case.

---

## Phase 5 — CVE / vulnerability data

Goal: surface known vulnerabilities as raw counts. No severity theater.

- [ ] Run `npm audit --json` as a child process
- [ ] Parse output: extract vulnerability count per `severity` level (`info`, `low`, `moderate`, `high`, `critical`)
- [ ] Report raw counts per level — do NOT compute a composite "risk score"
- [ ] If `npm audit` fails (no lock file, network error): report "vulnerability data unavailable" with reason
- [ ] `--no-audit` flag: skip this step entirely

**Philosophy note in report:** "Vulnerability counts are from npm audit at the time of this run. Severity labels are npm's own classification. Whether a vulnerability is exploitable in your specific usage requires manual review."

---

## Phase 6 — Unused dependency detection (static analysis)

Goal: find `dependencies` entries that appear to have no import in source files.

- [ ] Scan all `.ts`, `.tsx`, `.js`, `.jsx`, `.mjs` files under configurable `--src` path (default: `src/`, `lib/`, `app/`)
- [ ] Extract all `import` and `require()` specifiers
- [ ] Match against package names in `dependencies` (not `devDependencies` — those aren't expected to be imported directly in all cases)
- [ ] Unused candidates = dependencies with zero matching imports found
- [ ] Mark entire section as **PROBABILISTIC** — always print: "These packages had no detected imports in source files. This does not mean they are unused: dynamic imports, build plugins, CLI binaries, and indirect usage may not appear as imports."
- [ ] Do NOT label packages as "unused" — label as "no imports detected"

---

## Phase 7 — Output

### Markdown report structure

```
# dep-audit report — <project name>
<timestamp> · <direct prod deps> direct · <total> transitive

## Overview
<3-line TL;DR>

## Dependency counts
...

## Duplicate versions
...

## Package age distribution
...

## License breakdown
...

## Vulnerability summary (npm audit)
...

## No imports detected (probabilistic)
...

## Methodology
<auto-generated from config>

## Threats to validity
```

- [ ] Implement markdown report generator
- [ ] Auto-save to `reports/<project-name>/<timestamp>.md` + `.json`
- [ ] `--json` stdout mode

### Auto-generated methodology section

- [ ] Lock file version used (or node_modules fallback)
- [ ] Registry calls made (or --offline)
- [ ] Source paths scanned for import detection
- [ ] npm audit version (from `npm audit --version`)
- [ ] Threats to validity (static analysis limitations, registry data freshness, single-point-in-time snapshot)

---

## Phase 8 — Performance budgets (`--budget`)

- [ ] Available budget keys:
  - `directProdDeps` — direct production dependency count
  - `totalDeps` — total transitive dependency count
  - `duplicateVersions` — count of packages with multiple versions
  - `medianAgeDays` — median package age in days
  - `highSeverityVulnerabilities` — count of high + critical CVEs
- [ ] Exit code 1 on budget exceeded
- [ ] Budget results in report (PASS / FAIL per key)

---

## Phase 9 — End-to-end validation

- [ ] Run against `code-size` repo
- [ ] Run against `web-speed-test` repo
- [ ] Verify package counts match `package-lock.json` manually
- [ ] Verify license output matches spot-checked packages
- [ ] Review README against actual implementation

---

## Open questions

- [ ] Should devDependencies be included in age/license analysis? Proposal: yes, but reported separately.
- [ ] Monorepo support (workspaces)? Defer to v2.
- [ ] Should we cache registry responses locally to avoid re-fetching? Proposal: yes, TTL 24h, stored in `.dep-audit-cache/`. Implement in v1 to avoid hammering registry in repeated runs.
