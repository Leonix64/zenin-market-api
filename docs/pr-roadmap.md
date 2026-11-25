# Zenin Market — PR Roadmap

Companion to the [Project Plan](project-plan.md). Last updated Nov 25, 2026.

The plan says **what** we build. This file says **in what order** and **in which PR**.

## 1. Where we are today

**zenin-market-api** — folder skeleton only. No `package.json`, no TypeScript, no code.
It has the README, LICENSE, `.editorconfig`, `.gitignore` and empty module folders.

**zenin-market-web** — clean Angular 22.3 scaffold on SCSS. `npm ci`, `ng build` and `ng test` pass.
WEB-01 is done (PRs #3 and #4). Leftovers, all handled in later PRs:
- One unused `signal` import in `app.ts` (fixed in WEB-02).
- No ESLint, no lint or format scripts, no CI (WEB-02).
- No `src/styles/` tokens and no folder structure (WEB-03).

Both repos: `develop` is the default branch, `main` is behind. Everything goes through pull requests.

## 2. Gaps found in the plan, and how we close them

1. **The price check digit (C) is not pinned down.** The four test vectors are consistent with
   more than 100 different weight and formula combinations (found by brute force). The UPC check
   digit K matches on all four, but C needs the real GS1 General Specifications §7.9.3 algorithm.
   Get the official text (gs1.org, free PDF) and add more real labels as vectors before trusting
   it. Closed in API-05.
2. **luxon is listed in Phase 2 but Phase 1 needs it.** Sell-by is "days from today" in store
   time. Install it in API-08.
3. **The Phase 0 seed includes the demo station**, but users and argon2 belong to Phase 2.
   Split it: catalog seed in Phase 1, users seed in Phase 2.
4. **"The demo seed ships with a linked demo station" cannot work as written.** The link lives in
   a browser cookie, and a database seed cannot set a cookie in a visitor's browser. Decision needed
   before API-10: a `DEMO_MODE=true` flag that lets `/` request a station session automatically
   (never enabled outside the demo deploy).
5. **Dev auth bypass trap.** Declare routes with `authenticate` and `requireRole(...)` from day one
   in Phase 1. In dev, `authenticate` injects a fake clerk when `DEV_AUTH_BYPASS=true`. The app
   refuses to boot if that flag is on with `NODE_ENV=production`. Phase 2 swaps the implementation
   with no route rewiring.
6. **`POST /labels` must be idempotent on `clientId`** (unique index; a duplicate returns the
   existing label). Phase 4 offline sync depends on it, and it is cheap to build now.
7. **Money logic lives in two repos.** The API recalculates every total (source of truth, never
   trust the client total). The web needs the same formulas for the live total while weighing and
   for offline mode. Copy the same test vectors into both repos.
8. **Seed data must use invented brands.** Real scale categories (real brands) cannot go in a
   public portfolio repo.
9. **Open items are closed in the PR that needs them:** label size in mm (WEB-10), PLU and Search
   jump behavior (WEB-08), recent-labels count and per-station scope (API-08), session-expired
   state (WEB-11).

## 3. PR workflow (every PR, both repos)

```
git switch develop; git pull
git switch -c <type>/<short-name>
# code + tests
npm run typecheck; npm run lint; npm test          # API
npm run lint; npm run build; npm test              # Web
git add -A; git commit -m "<type>(<scope>): <summary>"
git push -u origin HEAD
```

The `lint` and `typecheck` scripts exist after API-02 and WEB-02. Until then, run build and test only.

- Open the PR against `develop`. Squash and merge. PR title = Conventional Commit title. Delete the branch.
- One concern per PR, roughly under 400 changed lines (generated code excluded).
  `develop` always builds, lints and passes tests.
- Every PR description: what changed, how to test it, which plan sections it covers.
- `develop` → `main` only at the end of a block, with a tag (`v0.1.0` after Block 1, and so on).
- Install packages with `npm install <pkg>` inside the PR that first needs them.
  Never copy versions from another repo.
- Checked on Nov 24: `typescript-eslint` accepts TS `<6.1`; `zod-to-openapi` wants `zod ^4`;
  `@angular/cdk` 22.3 and `transloco` support Angular 22; `angular-eslint` 22.x is out.
  No version conflicts expected.

## 4. Block 0 — Setup (Phase 0)

Goal: both repos build, lint, test and run an empty app on CI. No business logic.

Order: API-01 → WEB-01 ✅ → API-02 → WEB-02 → API-03 → API-04 → WEB-03.

**API-01 · `docs/project-plan`**
- Add the Plan and this roadmap to `docs/`. Replace the README.
- Fix the README (dead plan link, roadmap checkboxes, honest "Getting started").
- Done when: the README links resolve on GitHub.

**WEB-01 · `chore/frontend-cleanup`** ✅ (PRs #3 and #4)
- Project renamed to `zenin-market-web`, placeholder removed, title "Zenin Market",
  `typescript@~6.0.3`, SCSS everywhere, README fixed, Prettier clean.

**API-02 · `chore/backend-tooling`**
- `npm init`; Node engine 24; `.node-version`.
- Install dev tooling: typescript (~6.0.3), tsx, `@types/node@24`, eslint + typescript-eslint,
  prettier, vitest.
- `tsconfig` strict; `.prettierrc` (4 spaces, double quotes); `.prettierignore` (`package.json`,
  `package-lock.json`, `dist`); eslint config.
- Scripts: `dev` (`tsx watch --env-file=.env`), `build`, `start`, `typecheck`, `lint`, `format`, `test`.
- One smoke test.
- GitHub Actions: typecheck + lint + test on PRs to `develop`.
- Done when: CI is green on the PR itself.

**WEB-02 · `chore/frontend-tooling`**
- `ng add angular-eslint`; prettier scripts (`format`, `format:check`); `lint` script.
- Remove the unused `signal` import in `app.ts`.
- GitHub Actions: lint + build + test on PRs to `develop`.
- Done when: CI is green.

**API-03 · `feat/backend-skeleton`**
- Install core deps: express, mongoose, zod, helmet, cors, pino, pino-http, pino-pretty (+ `@types`).
- `src/shared/config` (zod-validated env, loaded with `--env-file`), a clean `.env.example`, logger.
- `app.ts` (`trust proxy` 1, helmet, cors, json, pino-http) and `server.ts` (Mongo connect,
  graceful shutdown).
- Error classes and error middleware (one JSON error shape). `GET /health`.
- Done when: a `supertest` test covers `/health` and the error shape, and the server boots against Atlas.

**API-04 · `feat/backend-openapi`**
- Install zod-to-openapi and swagger-ui-express. Registry, `GET /openapi.json` and `/docs`.
- Register `/health` as the first documented route.
- Validation middleware (body, params, query → parsed data in `res.locals`, because `req.query`
  is read-only in Express 5).
- Done when: Swagger UI loads and a bad body returns the standard error shape.

**WEB-03 · `feat/frontend-foundation`**
- `src/styles/`: `_tokens.scss` (dark blue categories, lighter blue products, red selected,
  spacing, radii), `_mixins.scss`, global reset. Barlow Condensed and Inter from `@fontsource`
  (local, no Google Fonts).
- Folders `core / shared / features`, created only when a file lives in them.
- `app-config` constants: store name, tagline, address, website, `STORE_TIMEZONE`, currency.
- Transloco with `en.json`. Dev proxy `/api` → API.
- Routes skeleton: `/` (scale, lazy), `/setup`, `/office` (lazy, placeholders).
- Done when: `ng serve` shows a placeholder per route, and lint, build and test are green.

## 5. Block 1 — Scale core (Phase 1)

Goal: the full clerk flow works end to end with the dev auth bypass:
category → product → weigh → tare → sell-by → PRINT → label on the simulated printer. Tag `v0.1.0`.

**API-05 · `feat/money-weight-barcode`** (first real code: pure functions, TDD, no database)
- `money`: `calcWeightTotal(milliLb, priceCents)` with `Math.round`, `calcPieceTotal`,
  `MAX_LABEL_TOTAL_CENTS = 9999`.
- `weight`: `netMilliLb(gross, tare)`.
- `barcode`: `buildBarcode(plu, totalCents)`, `parseBarcode`, `upcCheckDigit`, `priceCheckDigit`.
- Tests: the four vectors, 580×999 → 579, 575×549 → 316, zero weight, total over 9999 throws,
  PLU left-pad, non-numeric PLU rejected, parse round trip.
- Resolve gap #1 (official C algorithm) here and cite the spec section in a comment.
- Done when: all vectors pass and the C formula is traceable to the spec.

**API-06 · `feat/catalog-models-seed`**
- Schemas: Department, Category, Product, ContainerPreset. Indexes: unique `(departmentId, plu)`,
  unique `(departmentId, slot)` on categories, unique `(categoryId, slot)` on products.
- zod schemas registered in OpenAPI. `npm run seed`: Deli, fictional categories (9) and about
  40 products (mix of weight and unit, one category with 16+ items to test paging), presets 8/16/32 oz.
- Done when: the seed is re-runnable (wipes and recreates) and model tests pass with mongodb-memory-server.

**API-07 · `feat/catalog-endpoints`**
- `authenticate` + `requireRole` (dev bypass per gap #5).
- `GET /categories`, `GET /categories/:id/products?page=` (page = ceil(slot / 15)),
  `GET /products/by-plu/:plu`, `GET /products?search=&mode=` (escape regex),
  `GET /container-presets`.
- Module layout: routes, controller, service, repository, schema, validation, tests.
- Done when: integration tests cover paging, not found (404 shape), search modes and regex-injection input.

**API-08 · `feat/labels`**
- Label model (`clientId` unique). `POST /labels`: validate, load product, resolve price
  (`PriceResolver` returns base for now), recompute totals on the server, reject a total over
  9999 (422), build the barcode, store the snapshot. Idempotent on `clientId`.
- `GET /labels?limit=` (decide: how many, per station or global).
- Install luxon; sell-by from days or exact date in `America/Chicago`.
- Done when: tests cover rounding, limit, idempotency, snapshot immutability and sell-by across DST.

**WEB-04 · `feat/scale-shell`**
- Letterbox scaler (design 800×480, scale to fit), header with live store-time clock, scale route layout.
- Shared pipes: money, weight (lb), scale date `MM.DD.YY`, office date. Unit tests for the pipes.
- Done when: resizing the window keeps the 800×480 ratio.

**WEB-05 · `chore/openapi-client`** (after API-07)
- Install ng-openapi-gen; `npm run api:gen` reads the API's `/openapi.json`.
  Decide whether to commit the generated code (recommended: commit, with a header comment).
- Done when: one generated service call works against the local API.

**WEB-06 · `feat/scale-catalog`**
- Category grid (5×3), product grid (15 per page, ▲ / ▼ PAGE only when there is more than one
  page), product selected → detail panel with name, PLU, price per lb.
- Data goes through a `CatalogRepository` interface (HTTP implementation now; Dexie replaces it
  in Phase 4). HOME resets state.
- Done when: component tests cover the paging buttons and slot placement.

**WEB-07 · `feat/scale-weighing`**
- Weight simulator service and panel. Tare presets 8/16/32 oz (weight items only);
  quantity − / + (piece items).
- Local total using the same formulas and the same four vectors as the API (gap #7).
- PRINT and SELL DATE disabled rules from Plan §4; red banner over $99.99.
- Done when: unit tests cover every disabled-state rule.

**WEB-08 · `feat/scale-plu-search`**
- PLU keypad, "PLU not found" red card, text search with contains / startsWith and an empty state.
- Close the open item: jump to the category page, or only load the detail panel.

**WEB-09 · `feat/scale-sell-by`**
- CDK dialog: days or exact date, print date (default yes), print time (default no).
  Pure date helper tested across store-time boundaries.

**WEB-10 · `feat/scale-label-print`**
- `clientId = crypto.randomUUID()` per label. PRINT → `POST /labels` → show the result.
- Label templates (by weight, by piece) with the `@page` size as one CSS variable
  (close the mm open item with a real label). jsbarcode draws only the API's barcode digits.
- Simulated printer panel (demo only) and recent labels (read-only).
- Done when: an end-to-end manual run works and the printed barcode is read correctly by a real
  decoder (a phone scanner app is enough). Tag `v0.1.0`.

## 6. Block 2 — Office and auth (Phase 2)

Goal: the dev bypass is gone. Station session on the scale, office login, full catalog and
specials management. Tag `v0.2.0`.

- **API-09 · `feat/auth-core`** — User model (role, displayName), argon2id, jose, refresh rotation
  with reuse detection, httpOnly cookies, rate limit. `POST /auth/login|refresh|logout`.
  Port from the reference auth repo with the nine changes listed in the plan. Tests for rotation,
  reuse detection and expiry.
- **API-10 · `feat/rbac-stations`** — real `authenticate`, `requireRole` on every route,
  `POST /auth/link-station` (office only), long-lived station refresh, users seed (one office
  user, one demo station), `DEMO_MODE` decision (gap #4).
- **API-11 · `feat/specials`** — Special CRUD, overlap check (409), start 00:00 and end 23:59:59
  store time stored in UTC, `PriceResolver` uses specials, labels store `priceSource` and
  `specialId`. DST-boundary tests.
- **API-12 · `feat/office-catalog-crud`** — products and categories CRUD, layout endpoints
  (move or swap slots, slot conflicts), duplicate PLU → 409, deactivate instead of delete.
- **WEB-11 · `feat/auth-core`** — token signal, session restore on app start (`/auth/refresh`,
  `withCredentials`), interceptor with one in-flight refresh, guards, office login,
  session-expired state.
- **WEB-12 · `feat/station-setup`** — `/setup` flow (office signs in, picks a station);
  the scale boots straight to Categories.
- **WEB-13 · `feat/office-products`** — office layout (sidebar with "Open scale"), products list and form.
- **WEB-14 · `feat/office-grid-editors`** — categories list, 5×3 grid editor and per-category
  product grid with pages (CDK drag-drop).
- **WEB-15 · `feat/office-specials`** — list and form, show the 409 overlap error clearly.
- **WEB-16 · `feat/scale-special-banner`** — SPECIAL banner with end date and base price.

## 7. Block 3 — Queue (Phase 3) · tag `v0.3.0`

- **API-13 · `feat/tickets`** — Ticket model, atomic counter (`findOneAndUpdate` + `$inc`),
  daily reset in store time, next / repeat / cancel / status endpoints.
- **API-14 · `feat/queue-realtime`** — socket.io, rooms, events on every ticket change,
  auth on the socket handshake.
- **WEB-17 · `feat/queue-flag-clerk`** — `queueEnabled` flag, QUEUE buttons, clerk controls.
- **WEB-18 · `feat/queue-ticket-display`** — customer ticket and live display with socket.io-client.

## 8. Block 4 — Offline (Phase 4) · tag `v0.4.0`

- **API-15 · `feat/sync-endpoints`** — catalog snapshot with `updatedAt` / ETag, label batch
  upload (idempotent on `clientId`).
- **WEB-19 · `feat/offline-catalog`** — dexie, `CatalogRepository` swapped to local-first.
- **WEB-20 · `feat/offline-label-outbox`** — save the label locally first, sync when online,
  retries, sync indicator in the header slot.
- **WEB-21 · `feat/pwa`** — `@angular/service-worker`, app shell cache.

## 9. Block 5 — Polish (Phase 5) · tag `v1.0.0`

- Tests: API coverage for money, barcode and specials; web component tests; optional Playwright happy path.
- Deploy: API on Render or Railway, web on Vercel or Netlify with a `/api/*` rewrite to the API
  (cookie `sameSite: "lax"`), Atlas env vars, demo mode.
- README (EN) with screenshots, architecture diagram, navigation diagram and API docs link;
  short demo video.
- Close every item in Plan §15.

> Blocks 2 to 5 are an outline. Each block is expanded into full PR specs when we get there,
> using what we learned in the previous one.
