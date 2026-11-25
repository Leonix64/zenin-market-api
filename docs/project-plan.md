# Zenin Market — Project Plan v2
 
Final plan after the mockup review · Nov 25, 2026 · MVP scope: Deli department only
 
## 1. Overview
 
Zenin Market is a portfolio project that models a supermarket deli counter: a touch-screen scale where the clerk picks a product, weighs it and prints a label, plus a back office where products, categories, prices and specials are managed.
 
It is not a cash-register POS. The scale prints a label with a barcode that already contains the price, and a cashier scans that label later, so the register does not need the weight.
 
All names, brands, prices, logos and addresses are fictional. The project and the fictional store share one name: Zenin Market — Fine Foods & Provisions.
 
Everything is in English: UI, code, comments, database fields, API, commits and README. The UI is i18n-ready (translation files, English only for now) so Spanish can be added later without rework.
 
The scale screens copy the layout of a real DIGI deli scale. Reference screen: 800×480.
 
## 2. Scope
 
### MVP — Scale (clerk, no login screen)
 
- Category grid (5×3 slots) and product grid with pages (15 slots per page, ▲ PAGE / ▼ PAGE buttons).
- PLU keypad and text search (contains / starts with).
- Tare presets (8oz, 16oz, 32oz) for weight items; quantity − / + for piece items.
- Sell-by dialog: number of days or exact date, print date yes/no, print time yes/no.
- Active price shown automatically, with a SPECIAL banner when a special applies.
- PRINT prints directly, with no confirmation step. A total over $99.99 blocks printing.
- Recent labels: read-only list.
- Demo helpers: weight simulator and a simulated printer panel that shows the printed label.
- One-time station setup.
### MVP — Office
 
- Login (email + password).
- Products: list and form (PLU, name, category, sale type, base price, shelf life, ingredients, grid slot, active).
- Categories: list and 5×3 grid layout.
- Product grid per category, with pages.
- Specials with dates and overlap check.
- 'Open scale' link in the sidebar.
### Phase 3 — Queue (behind the queueEnabled flag)
 
Clerk controls (next, repeat, cancel number), customer ticket, live display with WebSockets. The QUEUE buttons are hidden on the scale while the flag is off.
 
### Post-MVP ideas
 
- Reprint of a recent label (if added: reprint the same snapshot and barcode, do not create a new Label).
- Label log in office, read-only, with filters by department, date and user. It must never show totals or be presented as sales data.
- Other departments (butcher, bakery, guacamole and salsas, salads, produce, tortilleria), added as data.
- Spanish (UI translations and an optional Spanish product name, searchable).
- Cashier module that scans the label barcode.
- Ionic/Capacitor wrapper.
### Non-goals
 
- Batches, expiration tracking, waste reports. A printed label does not mean something was sold (labels also identify new stock).
- Checkout and payments.
- Real scale or printer hardware. Weight is simulated and labels are printed as PDF or print-friendly HTML.
- Unit price change button, HOT ITEMS button, container preset screen (presets are seed data), allergen fields, per-category colors.
## 3. Screens (23 mockups)
 
**Scale:** 1 Categories · 2 Products + weighing (paged) · 3 Sell-by dialog · 4 PLU keypad · 5 Search · 6 Recent labels · 16 By piece (quantity) · 17 Weight simulator · 18 Over $99.99 · 19 Special active · 20 PLU not found · 21 Printed label (demo only) · 23 One-time station setup
 
**Labels:** 7 By weight · 8 By piece
 
**Office:** 14 Login · 11 Products · 15 Categories + grid · 22 Product grid with pages · 12 Specials
 
**Queue (Phase 3):** 9 Customer ticket · 10 Display · 13 Scale queue controls
 
## 4. Navigation and button states
 
Main flow: Categories → Products (page) → weigh / tare / quantity → optional Sell-by → PRINT.
 
- PLU keypad and Search load the product directly. HOME returns to Categories and clears product, weight and tare.
- SELL DATE is disabled until a product is selected.
- PRINT is disabled when there is no product, net weight is 0, quantity is 0, or the total is over $99.99 (red banner: split into two labels).
- ▲ / ▼ PAGE and 'PAGE x/y' appear only when the category has more than one page.
- Tare buttons only apply to weight items. Piece items show − QTY / + QTY instead.
- SPECIAL banner shows when the active price comes from a special (end date and base price).
- PLU not found: stay on the keypad with a red card. Search with no results shows an empty-state message.
- The header shows a live clock in store time.
- QUEUE buttons only render when queueEnabled is on.
## 5. Authentication and roles
 
Roles: clerk and office. A clerk is a scale station (for example Deli Scale 1), not a person.
 
- `/` Scale: no login screen, ever. The session comes from the station's long-lived refresh cookie, so after a restart the scale opens on Categories.
- `/setup`: one-time station link. An office user signs in on the device and picks the station. Clerks never see it.
- `/office`: login with email and password, office role only.
- The office sidebar has 'Open scale' (new tab). The scale has no link to the office.
- Every Label stores createdBy (the station or office user display name).
- Clerks cannot change prices.
- The public demo seed ships with a linked demo station, so visitors land on Categories.
Reference repo for auth: Login24.22.0 (Express 5, argon2id, jose, refresh token rotation with reuse detection, httpOnly cookie). Reuse structure and logic, not versions. Changes for Zenin:
 
1. Add role (clerk | office) to User and to the access token, plus a requireRole() middleware.
2. Remove /auth/register. Users come from the seed script.
3. Change ISSUER to zenin-market-api.
4. Validate body, params and query. In Express 5, req.query is read-only: store parsed data in res.locals.
5. app.set('trust proxy', 1) for Render/Railway.
6. sameSite 'lax' cookies break cross-site in production: proxy /api/\* from Vercel/Netlify to the API.
7. Fix .env.example (JWT\_ACCESS\_SECRET holds a pasted command) and the README env names.
8. @types/node must match Node 24. Translate Spanish comments to English.
9. Angular: access token in memory (signal), session restored with /auth/refresh (withCredentials) on app start, and the interceptor serializes refresh (one in-flight request).
## 6. Business rules
 
### PLU codes
 
Stored as a string, 1 to 5 digits, never as a number. Deli uses 4 digits. The barcode has room for 5, so it is left-padded with zeros (8137 becomes 08137). PLU is unique inside a department.
 
### Sale types
 
Each product is sold by weight (price per pound) or by unit (price per piece). 'Prepared in store' is not a sale type, just a product with optional ingredients and a shelf life.
 
### Money and weight
 
- Money is stored as integer cents (549 = $5.49). Weight is stored as integer thousandths of a pound (575 = 0.575 lb).
- Weight total: totalCents = Math.round(weightMilliLb \* priceCents / 1000). Example: 580 × 999 / 1000 = 579.42, rounds to 579, so $5.79.
- Piece total: quantity \* priceCents.
- Net weight = gross weight − tare. Each container preset stores its own tareMilliLb.
- Limit: the barcode has 4 price digits, so a label total cannot exceed $99.99 (9999 cents). Over the limit, PRINT is blocked.
### Prices and specials
 
- Office sets the base price and creates specials with a start and end date. The clerk never chooses a price.
- Active price today = special if one is active, otherwise base price.
- A product cannot have two overlapping specials. The API rejects the second one (HTTP 409).
- Dates are evaluated in store time, America/Chicago (config STORE\_TIMEZONE). A special runs from 00:00 on its start date to 23:59:59 on its end date, store time, and is stored in UTC.
- Each label stores a snapshot (name, PLU, unit price, total, price source), so old labels never change when prices change.
### Sell-by date
 
Each product has default shelfLifeDays. At print time the clerk can change the number of days (counted from today) or pick a date, and choose whether to print the date (default yes) and the time (default no).
 
### Grid slots and pages
 
Categories and products have a slot. Empty slots are allowed. For products, slots continue past 15: page = ceil(slot / 15), position = ((slot − 1) mod 15) + 1.
 
### Display formats
 
Scale uses the DIGI style MM.DD.YY (11.27.26). Office uses readable dates (Nov 27, 2026). Both live as shared pipes. Money is always USD, set once in config.
 
### Barcode
 
UPC-A, 12 digits, variable measure with the price inside: 2 + IIIII + C + PPPP + K.
 
- 2: number system for in-store variable-measure items.
- IIIII: PLU, left-padded with zeros to 5 digits.
- C: price check digit, GS1 General Specifications section 7.9.3 (four-digit price field).
- PPPP: total price in cents, 4 digits.
- K: standard UPC-A check digit (modulo 10).
- Reading a barcode: digits 2 to 6 are the PLU, digits 8 to 11 are the price, then validate C and K.
- Test vectors: PLU 8137 + 316 = 208137803161 · PLU 8925 + 599 = 208925105996 · PLU 9521 + 274 = 209521902743 · PLU 8090 + 579 = 208090305795 (taken from a real label, so it also checks C against a real scale).
- Before building the register module, test generated barcodes with a real decoder or scanner.
## 7. Label templates
 
**By weight:** logo and tagline, product name, ingredients (only when the product has them), net weight, unit price per lb, total price, barcode with human-readable digits, sell-by date (plus time if enabled), PLU, KEEP REFRIGERATED line, store address, website.
 
**By piece:** same layout with quantity, price each and total price.
 
- Store name, tagline, address and website come from config. KEEP REFRIGERATED is a fixed template line that can be removed.
- Output: PDF or print-friendly HTML with a print CSS page size in millimeters (size still to be chosen).
- Demo: after PRINT, a simulated printer panel shows the label. It is not a step in the clerk flow.
- Saving: the label is saved locally first (with its UUID), then synced.
## 8. Data model (MongoDB collections)
 
- Department: name, slug (MVP has one: Deli).
- Category: departmentId, name, slot.
- Product: departmentId, categoryId, plu (string), name, saleType (weight | unit), priceCents, shelfLifeDays, ingredients (optional), slot, active.
- Special: productId, priceCents, startsAt, endsAt.
- ContainerPreset: name (8 oz, 16 oz, 32 oz), tareMilliLb. Seed data, no screen.
- Label: clientId (UUID), departmentId, productId, pluSnapshot, nameSnapshot, saleType, grossMilliLb, tareMilliLb, netMilliLb, quantity, unitPriceCents, totalCents, priceSource (base | special), specialId (optional), sellByDate, printSellBy, printTime, barcode, createdBy, createdAt. No currency field.
- Ticket (Phase 3): number, status (waiting | called | cancelled), createdAt, calledAt.
- User: email, passwordHash, role (clerk | office), displayName.
Removed from the original model: Category.color and Product.nameEs.
 
## 9. API outline
 
- **Auth:** POST /auth/login, POST /auth/refresh, POST /auth/logout, POST /auth/link-station (office only, used by /setup). No register.
- **Scale (clerk or office):** GET /categories · GET /categories/:id/products?page= · GET /products/by-plu/:plu · GET /products?search=&mode=contains|startsWith · GET /container-presets · POST /labels (calculates totals, builds the barcode, saves) · GET /labels?limit= (recent, read-only).
- **Office (role office):** full CRUD on /products, /categories, /specials, plus layout endpoints for category slots and product slots.
- **Queue (Phase 3):** POST /tickets, POST /tickets/next, POST /tickets/repeat, POST /tickets/:id/cancel, GET /tickets/status.
- The API publishes an OpenAPI spec (zod to OpenAPI). The web repo generates its TypeScript types from it.
## 10. Tech stack and rules
 
**Rules**
 
- Install packages with npm install at setup time. Never copy versions from another repo. package-lock.json locks them.
- Pins: typescript \~6.0.3 in both repos, @types/node 24 in the API.
- Use libraries for boring stuff. Write by hand only the money, weight and barcode logic, with unit tests.
- Styles: SCSS, one .scss per component, no Tailwind. Shared tokens and mixins in src/styles/.
- Prettier: 4 spaces, double quotes.
- Node built-ins instead of libraries: --env-file (no dotenv), crypto.randomUUID() (no uuid).
- Environment: Windows + PowerShell, Node 24, npm 11, Angular CLI 22.3 (update before ng new).
**API (zenin-market-api)**
 
- Core (Phase 0): express 5, mongoose, zod, @asteasolutions/zod-to-openapi, swagger-ui-express, helmet, cors, pino, pino-http, pino-pretty.
- Auth (Phase 2): argon2, jose, cookie-parser, express-rate-limit, luxon (specials in America/Chicago).
- Queue (Phase 3): socket.io.
- Dev: typescript, tsx, @types/node 24, @types/express, @types/cors, @types/cookie-parser, @types/luxon, @types/swagger-ui-express, vitest, supertest, @types/supertest, mongodb-memory-server, eslint, typescript-eslint (needs TS below 6.1), prettier.
**Web (zenin-market-web)**
 
- Angular 22.3 (ng new zenin-market-web --style=scss): standalone components, signals, lazy routes, no NgRx.
- @angular/cdk (dialogs and overlays), jsbarcode (draws the barcode, digits are computed by us), ng-openapi-gen (types from the API spec).
- Later: dexie (Phase 4, IndexedDB), @angular/service-worker (Phase 4), socket.io-client (Phase 3), @playwright/test (Phase 5, optional).
- Dev: angular-eslint, eslint, prettier. Unit tests use the Angular default runner.
**Hosting:** MongoDB Atlas free tier. API on Render or Railway, web on Vercel or Netlify.
 
## 11. Architecture
 
Two separate repositories: zenin-market-api (plan and docs live in docs/) and zenin-market-web.
 
Backend: domain modules, not folders by file type.
 
- src/modules/products, categories, specials, labels (pricing and barcode logic), queue, auth.
- Each module holds routes, controller, service, repository, schema, validation and tests.
- src/shared: config, errors, middleware, money and time utils. Entry: app.ts and server.ts.
Frontend: standalone components, feature folders, lazy routes, signals.
 
- src/app/core: auth, http interceptors, guards, config.
- src/app/shared: ui components, pipes (money, weight, dates), utils.
- src/app/features: scale (category grid, product grid, detail, keypad, search, sell-by, recent labels, setup), office (products, categories, product grid, specials), queue, auth.
- Scale screens are designed at 800×480 and scaled to fit the display with letterbox. Colors and spacing come from tokens in src/styles/.
- Colors: categories use one dark blue, products one lighter blue, selected item red. Fonts: Barlow Condensed (scale and labels) and Inter (office).
## 12. Phases
 
- **Phase 0 — Setup:** two repos, Atlas cluster, TypeScript, lint, env variables, Swagger skeleton, seed script with fake data (including the linked demo station).
- **Phase 1 — Scale core:** categories and products with pages, PLU lookup, search, tare presets, weight simulator, price calculation, barcode generator, label templates, simulated printer, sell-by dialog, $99.99 limit. Unit tests for rounding and barcode. In dev, the scale runs with a dev-only auth bypass until Phase 2.
- **Phase 2 — Office and auth:** JWT, roles, station link and setup screen, office login, product and category CRUD, layout editors, specials with overlap check, active-price resolution and SPECIAL banner.
- **Phase 3 — Queue:** Ticket model, clerk controls, customer ticket, live display, queueEnabled flag.
- **Phase 4 — Offline mode:** cache the catalog locally, save each new label locally first (with its UUID), sync when the connection returns. Reserve a small sync indicator slot in the scale header.
- **Phase 5 — Polish:** tests, README with screenshots and diagrams, deploy, short demo video.
## 13. Offline: three rules from day one
 
1. Every label gets a client-generated UUID (crypto.randomUUID()).
2. The scale reads products from a local cache, not from the network on every tap.
3. Printing saves the label locally first, then syncs it.
## 14. Decisions made in the mockup review
 
- Scale has no clerk login. Station-based auth with a one-time setup; office has its own login.
- Recent labels stays, read-only. Reprint is post-MVP.
- Print preview is not a flow step. It exists only as a demo simulated-printer panel.
- Product grids paginate (15 slots per page) instead of scrolling.
- Queue buttons are hidden behind queueEnabled until Phase 3.
- English only; i18n-ready. No nameEs or category color in the MVP.
- Container presets and HOT ITEMS are not part of the MVP UI. Unit price change stays out.
- Labels gain PLU, optional sell-by time, ingredients only when present, and a KEEP REFRIGERATED line.
- Data in the mockups was made consistent: one active special (8082), sell-by from 5-day shelf life, all categories listed.
## 15. Open items
 
- Label size in millimeters for the print CSS.
- Station link details: how office creates additional stations (seed only for now).
- How many recent labels to show, and whether the list is per station.
- Whether PLU and Search jump to the product's category page or only load the detail panel.
- Session-expired state for office (refresh failure).
- Allergen information (needs a model field and an office screen).
- Test the barcode with a real decoder before any cashier module.
## 16. Portfolio checklist
 
- Fictional store and fake data only (no real brands, logos, prices or addresses).
- English README with screenshots, architecture diagram, navigation diagram and API docs link.
- Tests for money, weight and barcode logic (four test vectors).
- README story: labels are also used to identify incoming stock, not only to sell.
- Live demo link and a short demo video.
## 17. Git workflow
 
Deferred. No Git workflow yet beyond the two repositories. The earlier proposal (main and develop branches, pull requests only, squash merge, Conventional Commits) stays available for later.
