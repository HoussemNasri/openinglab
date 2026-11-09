# AGENTS.md

Guidance for AI coding agents working on OpeningLab: a free, private, offline-capable chess-opening trainer. It is a **static, client-side PWA** built with HTML, CSS, and native ES modules. There is no backend, framework, bundler, chess engine, or runtime dependency. Node.js ≥ 22 is used only for validation, tests, and the production build.

Read these before non-trivial changes: [`docs/INVARIANTS.md`](docs/INVARIANTS.md) (authoritative rules), [`CONTRIBUTING.md`](CONTRIBUTING.md), [`docs/RELEASING.md`](docs/RELEASING.md), [`SECURITY.md`](SECURITY.md). If docs conflict, `docs/INVARIANTS.md` and `package.json` win. `js/openings/FORMAT.md` is partly stale (it mentions `npm test` and hand-bumping `CACHE_NAME`); use `npm run validate:curriculum` and `npm run sw:generate` instead.

## Commands

```bash
npm ci                          # install dev tooling (once)
npx playwright install chromium # browser for e2e/a11y (once)
python3 -m http.server 8080     # serve source for manual UI work (file:// is unsupported)

npm run quality                 # no browser needed: syntax, lint, version, curriculum, unit tests, static checks, sw check
npm run test:unit               # node --test tests/unit/*.test.mjs
npm run lint                    # eslint, --max-warnings=0
npm run validate:curriculum     # replays every line and checks schema/quizzes
npm run sw:generate             # regenerate committed sw.js after ANY runtime change
npm run test:e2e | test:a11y    # build + Playwright (serves dist/ on 127.0.0.1:4174)
npm run validate                # canonical full gate (quality + build + perf budget + all Playwright); same as CI
```

Iterate with `npm run quality`. Run the full `npm run validate` before declaring UI, routing, service-worker, or curriculum work done. If Chromium is unavailable, say which checks were not run. Set `PLAYWRIGHT_EXECUTABLE_PATH` to use a custom Chromium.

## Project map

| Path | Role |
|---|---|
| `index.html`, `css/styles.css`, `manifest.webmanifest` | Static shell, all styles, PWA manifest (relative paths only) |
| `js/app.js` | Hash routing, library, themes, import/export UI |
| `js/board.js` | Deterministic board model + accessible interaction |
| `js/lesson.js`, `practice.js`, `quiz.js` | Lesson tabs/playback, Guided/Recall/Sprint practice, quiz flow |
| `js/storage.js` | `ProgressStore`: progress, mastery, reviews, notes, migrations, backups |
| `js/openings/` | Curriculum (`catalog.js`, `additional.js`, `variations.js`), `schema.js` constructors, `explanations.js`, public API `index.js` |
| `js/version.js`, `js/pwa.js` | `APP_VERSION`; service-worker registration and update prompt |
| `sw.template.js` → `sw.js` | Worker source → **generated** committed worker |
| `scripts/` | Validation, build, SW generation, perf check, `serve.mjs` |
| `tests/unit/*.test.mjs`, `tests/e2e/*.spec.js` | `node:test` unit tests; Playwright flows, PWA/offline, a11y (axe), performance |
| `dist/` | Build output, git-ignored. Never edit or commit it |

## Hard constraints (partly enforced by `check:static`, all enforced in review)

- No runtime `dependencies`, third-party JS/fonts/images/CDNs, analytics, network calls for user data, or backend. All imports in shipped JS are relative.
- UI modules import curriculum **only** from `js/openings/index.js`.
- Escape all user text and curriculum text interpolated into HTML. Each UI module has its own `escapeHTML`; use it for anything going into `innerHTML`.
- Must keep working: keyboard, touch, reduced motion, light/dark theme, import/export, offline, and both root and subpath hosting.
- Stay within `performance-budget.json` (bytes, file counts, precached assets). Don't raise a budget to make a check pass; reduce the code instead, or ask.

## Data and persistence rules

- Opening IDs and variation/line IDs are permanent user-data identity: lowercase kebab-case, unique, never renamed casually. Review keys are `<opening-id>:<line-id>`.
- `ProgressStore` is the only source of truth for progress, mastery, favorites, notes, and reviews. Mastery is derived, never assigned.
- Storage key stays `openinglab-progress-v1`; schema changes bump `STORAGE_VERSION` with a migration and tests.
- Anything loaded or imported goes through normalization. Imports are transactional; invalid or newer-version data must never overwrite current progress.

## Curriculum changes

- Use the `schema.js` constructors; follow the checklist in `CONTRIBUTING.md` (one `main` line, ≥ 5 variations, ≥ 5 quiz questions, a substantive explanation per move; no placeholders like "develops").
- The validator proves moves are **legal and well-formed**, not that they are **good opening theory**. Never invent lines or explanations from memory without care, and flag new chess content for human review in your summary.
- After an intentional curriculum change: `npm run validate:curriculum` → review counts/digest → `npm run curriculum:baseline` → `npm run sw:generate`. Never hand-edit `curriculum-baseline.json`.
- Update opening/variation/question counts in `README.md` when they change.

## Generated and release-controlled files

- `sw.js` is generated (content-hashed cache name + asset list). Never hand-edit it; run `npm run sw:generate` after any change to a precached runtime file (`index.html`, `css/`, `js/`, `assets/`, `manifest.webmanifest`). The unit test `checked-in service worker exactly matches generated output` fails otherwise. Put worker logic changes in `sw.template.js`.
- Version must match in `package.json`, `js/version.js`, and a `CHANGELOG.md` heading (`check:version`). Do not bump the version or edit release notes unless asked.

## Code style

No formatter is configured; match the surrounding code. 2-space indent, double quotes, semicolons, ES modules with explicit `.js` extensions, `const`/arrow functions, and `_`-prefixed unused args (the ESLint rule requires it). Comments only where intent is non-obvious. Keep changes small and surgical; don't refactor unrelated code.

## Testing expectations

- Behavior changes need regression coverage: logic in `tests/unit/` (`node:test` + `node:assert/strict`, use `tests/helpers/memory-storage.mjs` instead of real storage); user flows in `tests/e2e/`.
- Never weaken a validator, test, lint rule, or budget to make something pass; fix the data/code, or document a deliberate invariant change in `docs/INVARIANTS.md`.
- UI changes: check desktop and phone viewports, and offline reload when the app shell, routing, or curriculum changes.

## Git and pull requests

- Don't commit, push, tag, or open PRs unless asked. Keep unrelated changes separate and leave unrelated uncommitted work in the tree untouched.
- PR bodies follow `.github/pull_request_template.md` (**What changed**, **Validation** checklist, **Data compatibility**; write "None" when there's no effect). Report the exact validation commands you ran and their results.
