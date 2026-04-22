# NPM Dependency Upgrade Coordinator

Tactfully upgrade npm dependencies by **package-family groupings**, proceeding automatically from lowest risk to highest. Upgrade one grouping at a time, fully resolve all issues (build, types, tests, lint) before moving to the next. Never blindly `npm update`.

**Invoke with:** `/quality-routines:npm-deps-upgrade` + optional argument:
- **Single module:** `/quality-routines:npm-deps-upgrade npm-dependency/mtauth-ui`
- **All modules:** `/quality-routines:npm-deps-upgrade all`
- **No argument:** defaults to `all` — upgrades every npm module in the project.

---

## Principles

1. **Group by family.** Upgrade related packages together (`@heroui/*`, `@storybook/*`, `eslint-*`).
2. **Low-to-high risk, always.** Process groupings in strict risk order — never jump ahead.
3. **Resolve before advancing.** Every grouping must build, pass types, and pass tests before touching the next grouping.
4. **One commit per grouping.** Clean git history for easy bisect.
5. **Respect peer constraints.** If a grouping introduces peer conflicts, resolve them within the same grouping — don't defer.
6. **Preserve version conventions.** If the project uses exact versions (no `^`/`~`), maintain that style.

---

## Phase 0: Reconnaissance

### 0.1 Detect Package Manager

```bash
cd <module-path>
ls package-lock.json pnpm-lock.yaml yarn.lock 2>/dev/null
```

Set `$PKG` to `npm`, `pnpm`, or `yarn` based on which lockfile exists. Prefer `pnpm` > `npm` > `yarn` if multiple.

### 0.2 Validate Baseline

Before any upgrades, confirm the project is currently healthy. **Do not upgrade on top of a broken project.**

#### Clean Install & Build

Use the module's Makefile targets (every module in this repo has a Makefile):

```bash
make -C <module-path> clean
make -C <module-path> install
make -C <module-path> build
```

#### Type Check

```bash
cd <module-path> && npx tsc --noEmit 2>&1
```

Record the baseline error count (some projects carry known type errors). Upgrades must not increase this count.

#### Tests

```bash
make -C <module-path> test
```

#### E2E Tests (if available)

Check the module's Makefile for e2e targets:

```bash
grep -E '^e2e' <module-path>/Makefile
```

If present, run them in mock/standalone mode:

```bash
make -C <module-path> e2e
```

Record which tests pass — this is your regression baseline.

#### Workspace Consumers (monorepo only)

For modules consumed by other workspace packages or apps, rebuild the full chain:

```bash
make -C <upstream-dependency> build
make -C <downstream-consumer> build
```

If a consumer has e2e tests, run those too:

```bash
make -C <consumer-app> e2e
```

#### Baseline Snapshot

Record all results before proceeding:

```
Baseline:
- make clean + install + build: PASS/FAIL
- tsc --noEmit: PASS (or N known errors)
- make test: X passed, Y failed
- make e2e: X passed, Y failed (or N/A)
- Consumers: <list> all build
```

**If baseline is broken:** Stop immediately. Report the failure to the user. Do not upgrade on top of a broken project.

### 0.3 Collect Outdated Packages

```bash
$PKG outdated 2>&1
```

Capture the full outdated table: package name, current version, wanted version, latest version.

### 0.4 Check Security Vulnerabilities

```bash
$PKG audit 2>&1
```

Note critical/high severity issues — these get flagged in the plan but do NOT jump the queue.

### 0.5 Build the Grouping Plan

Analyze package names from the outdated list and organize into **family groupings** by scope or prefix:

**Auto-detect groupings by these rules (in priority order):**

1. **Scoped packages** — group by npm scope: `@heroui/*`, `@storybook/*`, `@types/*`, `@fortawesome/*`, `@testing-library/*`, `@tanstack/*`
2. **Plugin families** — group by shared prefix: `eslint-*`, `postcss-*`, `babel-*`, `rollup-plugin-*`
3. **Framework ecosystems** — group tightly coupled packages: `react` + `react-dom` + `@types/react`, `next` + `eslint-config-next`
4. **Standalone packages** — everything else, grouped into a final "misc" grouping

### 0.6 Classify Each Grouping by Risk Tier

Within each family grouping, classify the semver bump type:

- **Patch** (`1.2.3` → `1.2.9`) — bug fixes only
- **Minor** (`1.2.3` → `1.5.0`) — new features, possible subtle changes
- **Major** (`1.2.3` → `2.0.0`) — breaking changes expected

If a family grouping contains a mix of bump types (e.g., some packages are patch, one is major), classify the **entire grouping** by its highest bump type. This ensures the grouping lands in the correct risk tier.

### 0.7 Sort Groupings into Execution Order

**This is mandatory. Always execute in this exact order, top to bottom.** Do not rearrange, do not skip ahead. If a grouping is empty (no outdated packages match), skip it silently and proceed to the next.

#### Tier 1 — Patch Only (Lowest Risk)

| Order | Category | What matches | Why low risk |
|-------|----------|--------------|--------------|
| 1.1 | `@types/*` (patch) | Scoped type packages with patch bumps | Type-only, zero runtime effect |
| 1.2 | Patch-only families | Any family where ALL packages are patch bumps | Bug fixes only, no API changes |
| 1.3 | Standalone patches | Individual packages with patch bumps that don't belong to a family | Isolated, minimal blast radius |

#### Tier 2 — Minor Bumps (Low-Medium Risk)

| Order | Category | What matches | Why medium risk |
|-------|----------|--------------|-----------------|
| 2.1 | `@types/*` (minor/major) | Type packages with non-patch bumps | Type-only but may require code changes |
| 2.2 | Dev tool families (minor) | `eslint-*`, `prettier`, `postcss-*`, `@biomejs/*` | Dev-only, no runtime effect |
| 2.3 | Test library families (minor) | `@testing-library/*`, `vitest`/`@vitest/*`, `jest-*`/`@jest/*`, `ts-jest` | Test-only, no prod effect |
| 2.4 | Build tool families (minor) | `tsup`, `rollup`, `esbuild`, `@swc/*`, `terser` | Build-only |
| 2.5 | UI component families (minor) | `@heroui/*`, `@radix-ui/*`, `@fortawesome/*` | Runtime but self-contained |
| 2.6 | SDK families (minor) | `@aws-sdk/*`, `@stripe/*`, `openai` | Runtime, well-versioned |
| 2.7 | Utility library families (minor) | `zod`, `axios`, `lodash`, `date-fns`, `commander`, `framer-motion` | Runtime, widely used |
| 2.8 | CSS/styling families (minor) | `tailwindcss`, `@tailwindcss/*`, `postcss` | Affects visuals but not logic |
| 2.9 | Storybook family (minor) | `storybook`, `@storybook/*`, `@chromatic-com/*` | Dev tool, isolated |
| 2.10 | Remaining minor families | Any family not matched above with minor bumps | Catch-all |

#### Tier 3 — Major Bumps (High Risk)

**Major bumps require extra reconnaissance before installation.** See [Major Bump Pre-Flight](#major-bump-pre-flight) below.

**Major bumps are always executed one family at a time, in this order:**

| Order | Category | What matches | Why high risk |
|-------|----------|--------------|---------------|
| 3.1 | Dev tool majors | `eslint` major, `prettier` major, `@biomejs/*` major | Dev-only but config may break |
| 3.2 | Test framework majors | `jest` 29→30, `vitest` major, `@testing-library/*` major | Test rewrites possible |
| 3.3 | Build tool majors | `vite` major, `@vitejs/*` major, `tsup` major, `esbuild` major | Config/plugin API changes |
| 3.4 | Type system majors | `typescript` major (e.g., 5→6) | Affects ALL code, cascading errors |
| 3.5 | UI framework majors | `@heroui/react` major, `@radix-ui/*` major | Component API rewrites |
| 3.6 | SDK majors | `stripe` major, `@stripe/*` major, `@aws-sdk/*` major | API contract changes |
| 3.7 | i18n majors | `i18next`, `react-i18next`, `next-i18next` major | Config + API changes |
| 3.8 | Framework majors | `react`/`react-dom` major, `next` major | Highest blast radius |
| 3.9 | Remaining majors | Anything else with a major bump | Case-by-case |

#### Tier 4 — Security Remediation (After All Tiers)

Re-audit after all upgrades. Fix remaining vulnerabilities using the same grouping + gate cycle.

### 0.8 Present the Grouping Plan

Before making ANY changes, present the sorted plan as a numbered checklist showing tier, grouping name, package count, and bump types. Mark any security-flagged packages with a warning icon.

**Do NOT wait for confirmation on the ordering.** The ordering is fixed by the risk tiers above. Only pause for confirmation if:
- A major bump grouping has known breaking changes worth flagging
- The user explicitly asked to review the plan first

Otherwise, proceed directly into execution starting at Tier 1.1.

---

## Phase 1: Execute Groupings (Repeat for Each Grouping, In Order)

**Process every grouping in the exact order determined by Phase 0.7.** Do not skip a grouping unless it is empty. Do not reorder.

### Step 1 — Announce the Grouping

Before installing, briefly state which grouping is being processed:

```
## Grouping 2.5: @heroui/* (12 packages, patch/minor)
```

### Step 2 — Install the Grouping

Upgrade all packages in the current grouping to their target versions:

```bash
$PKG install <pkg1>@<target> <pkg2>@<target> ...
```

If peer dependency conflicts appear:
- **Read the conflict message carefully.** Identify which package needs what peer version.
- **Resolve within this grouping.** Pull in the required peer version or adjust the target version down.
- **Never use `--force` or `--legacy-peer-deps`** to bypass — that hides problems.

### Step 3 — Type Check

```bash
npx tsc --noEmit 2>&1
```

If type errors appear:
- Fix type incompatibilities introduced by the upgrade (renamed types, changed generics, removed exports).
- If a fix requires code changes, make them now — they belong to this grouping's commit.

### Step 4 — Build

```bash
$PKG run build 2>&1
```

If build fails:
- Read the error. Common causes: removed/renamed exports, changed API signatures, new required config.
- Fix within this grouping. All fixes are part of the same commit.

### Step 5 — Tests

```bash
$PKG test 2>&1
```

If tests fail:
- Distinguish between **test code needing update** (changed API) vs **actual regressions** (broken behavior).
- Update test code to match new APIs.
- If a test reveals a genuine regression, **roll back this grouping** and pin the offending package at its current version. Note it in the skip list.

### Step 6 — Lint (if available)

```bash
$PKG run lint 2>/dev/null
```

Fix any new lint errors introduced by code changes in steps 3-5.

### Step 7 — Gate Check

All must pass before proceeding:
- [ ] `tsc --noEmit` — zero errors (or project has no tsconfig)
- [ ] `$PKG run build` — exit 0
- [ ] `$PKG test` — all pass (or no test script)
- [ ] `$PKG run lint` — no new errors (or no lint script)

**If ANY check fails and cannot be resolved within 3 fix attempts:** Roll back the entire grouping.

```bash
git checkout -- package.json package-lock.json pnpm-lock.yaml src/
$PKG install
```

Log the grouping as **SKIPPED** with the failure reason, and **automatically advance** to the next grouping. Do not stop to ask.

### Step 8 — Commit the Grouping

```bash
git add package.json package-lock.json pnpm-lock.yaml 2>/dev/null
git add -u  # catch any other modified files from fixes
```

Commit message format:

```
chore(<module>): upgrade <grouping-name> packages

<pkg1>: <old> → <new>
<pkg2>: <old> → <new>

Code changes:
- <brief description of any fixes applied>
```

### Step 9 — Advance to Next Grouping

The committed state from this grouping is now the new baseline. **Automatically proceed** to the next grouping in the ordered list. No pause, no confirmation needed.

**Exception:** Before entering Tier 3 (major bumps), pause once and inform the user:

```
Tiers 1-2 complete. Entering Tier 3 (major version upgrades).
These may require migration effort. Proceeding with: <next grouping name>
```

Then continue automatically unless the user intervenes.

---

## Major Bump Pre-Flight

Before installing any Tier 3 (major) grouping, run these extra steps:

### Fetch Migration Guide

Use context7 or the library's official docs to check for a migration guide:

```
mcp__context7__resolve-library-id → mcp__context7__query-docs
Query: "migration guide v{old} to v{new} breaking changes"
```

**Key things to extract:**
- Package consolidation (e.g., 20 individual packages → 1 umbrella package)
- Removed/renamed exports and their replacements
- API pattern changes (e.g., single component → compound component pattern)
- CSS/styling integration changes (e.g., JS plugin → CSS import)
- Provider/wrapper removals

### Check for Component Pattern Changes

Major UI library upgrades often change the **component composition pattern**, not just props. Look for:

- **Flat → Compound:** `<Modal><ModalHeader>` → `<Modal><Modal.Header>`
- **Props → Children:** `<Input label="x" errorMessage="y">` → `<TextField><Label>x</Label><FieldError>y</FieldError></TextField>`
- **Single → Slots:** `<Button startContent={icon}>` → `<Button><Icon/>text</Button>`

These require JSX restructuring across every usage — not just find-and-replace.

### Check Tailwind/CSS Integration Changes

UI libraries sometimes change how they integrate with Tailwind between majors:

| Pattern | Example |
|---------|---------|
| JS plugin → CSS import | `heroui()` in tailwind.config.js → `@import "@heroui/styles"` in CSS |
| Content paths change | `node_modules/@heroui/theme/dist/**` → no longer needed |
| Theme config moves | JS theme object → `@theme { --color-primary: #xxx; }` in CSS |
| Provider removed | `<HeroUIProvider>` → nothing (React Aria handles it) |

### Map Workspace Dependency Chain

For monorepo upgrades, identify the rebuild order:

```
shared-lib (jefeui) → dependent-lib (mtauth-ui) → consuming-app (mtauth-demo)
```

After upgrading the shared lib:
1. Build the shared lib first
2. Check for type/API errors in dependent libs
3. Rebuild dependent libs
4. Verify consuming apps start and render

### Verify Build Output Compatibility

After upgrading a library that gets bundled (via Vite/tsup/rollup):
- Check the ESM output for CJS interop shims (`__commonJSMin`, `__require`)
- If present, the bundle includes CJS-only dependencies that may break in Turbopack/ESM-strict environments
- **Fix:** externalize the CJS dependency in the build config so the consuming app resolves it at runtime

---

## Phase 1.5: Consumer App Verification (Workspace Packages Only)

After a grouping upgrades a **workspace package** (anything under `npm-dependency/`), run consumer verification before committing:

### Rebuild Downstream Dependencies

```bash
# Rebuild the upgraded package
make -C npm-dependency/<package> build

# Rebuild any workspace packages that depend on it
make -C npm-dependency/<downstream-package> build
```

### Verify Consumer App Starts

For each app that imports the upgraded package:

```bash
# Start the dev server and check it renders
cd app-demos/<app>
rm -rf .next  # clear cached builds
$PKG run dev &
sleep 15
curl -sk https://localhost:3000 | head -20  # verify HTML returned
kill %1
```

### Run E2E Tests If Available

```bash
# Check for playwright/cypress
ls playwright.config.* cypress.config.* 2>/dev/null

# Run in mock mode first (no backend dependency)
MOCK=1 $PKG exec playwright test
```

E2E tests catch component rendering issues that type checks miss (e.g., compound component children not rendering, form inputs not triggering onChange).

---

## Phase 2: Security Remediation

After all family groupings are done, re-run the audit:

```bash
$PKG audit 2>&1
```

If vulnerabilities remain that weren't resolved by the grouping upgrades:
- If the fix is a patch/minor within an already-upgraded package: install it directly.
- If the fix requires a major bump: treat it as a single-package grouping and run the full Step 1-8 cycle.
- **Never run `$PKG audit fix --force`.**

---

## Phase 3: Final Verification

Run the full gate one last time against the final state:

```bash
$PKG install        # clean install from lockfile
$PKG run build
$PKG test
npx tsc --noEmit
```

Compare before/after:

```bash
$PKG outdated 2>&1  # should show fewer (ideally zero) outdated
$PKG audit 2>&1     # should show fewer (ideally zero) vulnerabilities
```

---

## Phase 4: Report

Present the final summary:

```
## Upgrade Report for <module-name>

### Completed Groupings (in execution order)
| # | Tier | Grouping | Packages | Status |
|---|------|----------|----------|--------|
| 1 | 1.1 | @types/* (patch) | 3 | DONE |
| 2 | 1.2 | ts-jest (patch) | 1 | DONE |
| 3 | 2.5 | @heroui/* (minor) | 12 | DONE |
| 4 | 3.4 | typescript (major) | 1 | SKIPPED — cascading type errors |
...

### Skipped / Pinned
| Grouping | Reason |
|----------|--------|
| typescript 5→6 | 47 type errors across 12 files, needs dedicated migration |

### Security
- Vulnerabilities before: X
- Vulnerabilities after: Y
- Remaining: <details>

### Final Status
- Build: PASS / FAIL
- Tests: PASS / FAIL
- Types: PASS / FAIL
- Lint: PASS / FAIL
```

---

## Multi-Module Mode (`all`)

When the user specifies `all` instead of a single module path:

1. **Detect all npm modules** — scan for directories under `npm-dependency/` with a `package.json` and a lockfile. **Also include the root `package.json`** — it contains shared devDependencies (e.g., `@biomejs/biome`, `typescript`, `husky`, `commitlint`) that must be upgraded alongside module-level packages.
2. **Run Phase 0 for every module (including root)** — collect outdated lists across all modules. For the root package, run `pnpm outdated` from the workspace root.
3. **Build a cross-module grouping plan** — families that appear in multiple modules (e.g., `@aws-sdk/*` in 5 modules) become a single grouping executed module-by-module. Root-level packages form their own groupings following the same risk tiers.
4. **Execute in risk order across all modules** — for each grouping in the sorted list, apply it to every affected module, gating each module independently. Root-level groupings follow the same tier order.
5. **If one module fails a grouping, skip it for that module only** — other modules in the same grouping continue.
6. **Commit per module per grouping** — `chore(aws-eventbridge-sdk): upgrade @aws-sdk/* packages` as a separate commit from `chore(contact-us-services): upgrade @aws-sdk/* packages`. Root-level upgrades use `chore(root): upgrade <grouping-name> packages`.

---

## Guard Rails

- **Never run `npm audit fix --force`** — it ignores semver and breaks things silently.
- **Never `--force` or `--legacy-peer-deps`** — resolve peer conflicts properly.
- **Never upgrade `react`, `next`, or `auth.js` in a mixed grouping** — these are their own grouping, always.
- **Check `.nvmrc` or `engines`** — don't upgrade packages beyond the project's Node.js version.
- **Respect `overrides` / `resolutions`** — understand why they exist before touching them.
- **Watch bundle size** — for UI libraries, compare build output sizes. Flag >10% increases.
- **One grouping, one commit** — never combine multiple groupings in a single commit.
- **Rollback is normal** — if a grouping can't be cleanly resolved, skip it. Report it. Don't force it.
- **Never reorder the risk tiers** — the low-to-high order is the entire point of this skill.
- **Skip alpha/beta/rc versions** — unless the project already uses prereleases for that package, never upgrade to a prerelease version (e.g., `1.1.0-alpha.2`). Pin at the latest stable version instead.

---

## Rollback

If a grouping goes sideways mid-resolution:

```bash
# Discard all uncommitted changes
git checkout -- .
$PKG install
```

If the entire upgrade session needs to be undone:

```bash
# Find the pre-upgrade commit
git log --oneline -20
# Ask user before resetting
```

Never force-reset without user confirmation.