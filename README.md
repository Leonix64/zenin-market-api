# Zenin Market API

> Backend for **Zenin Market — Fine Foods & Provisions**, a fictional supermarket.
> It powers a deli scale: pick a product, weigh it, print a label with a price-embedded barcode.

![Status](https://img.shields.io/badge/status-Phase%200%20(setup)-orange)
![Node](https://img.shields.io/badge/node-%E2%89%A5%2024-339933?logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6?logo=typescript&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue)

## What is this?

Zenin Market is a portfolio project that models how a real supermarket **deli counter** works:

- A touch-screen **scale** where the clerk picks a product, weighs it and prints a label.
- A **back office** where prices and weekly specials are managed.
- A label with a **barcode that carries the price**, so the cashier scans it later and the register does not need the weight.

This repository is the **REST API**. The Angular app lives in
[`zenin-market-web`](https://github.com/TU-USUARIO/zenin-market-web).

> ⚠️ Everything here is fictional: names, brands, prices, logos and addresses.

## Features (MVP)

- Categories and products for the Deli department
- Product lookup by **PLU code** and by text search (`contains` / `startsWith`)
- Tare presets (8 oz, 16 oz, 32 oz)
- Pricing for items sold **by weight** or **by unit**
- Weekly **specials** with start and end dates (no overlaps)
- **Label** creation: totals, price snapshot and barcode generation
- JWT auth with two roles: `clerk` and `office`
- OpenAPI (Swagger) documentation
- Queue / ticket system with live updates (Phase 3)

## Tech stack

| Area | Choice |
|---|---|
| Runtime | Node.js + TypeScript |
| Framework | Express |
| Database | MongoDB Atlas + Mongoose |
| Auth | JWT |
| Docs | Swagger / OpenAPI |
| Realtime (later) | Socket.IO |
| Tests | Unit tests for money, weight and barcode logic |

## Business rules at a glance

- **Money** is stored as integer cents: `549` = $5.49.
- **Weight** is stored as integer thousandths of a pound: `575` = 0.575 lb.
- **Total (by weight)** = `Math.round(weightMilliLb * priceCents / 1000)`.
  Example: `575 × 549 / 1000 = 315.675` → `316` → **$3.16**.
- **PLU** is a string (1 to 5 digits), unique inside a department. Deli uses 4 digits.
- The clerk never chooses a price. The system uses the **active price for today** (special if active, base price otherwise).
- A label stores a **snapshot** of name, PLU, unit price and total, so old labels never change.

## Barcode format

Variable-measure **UPC-A**, 12 digits: `2 + IIIII + C + PPPP + K`

| Part | Meaning |
|---|---|
| `2` | In-store variable-measure number system |
| `IIIII` | PLU, left-padded with zeros to 5 digits |
| `C` | Price check digit (GS1 four-digit price rule) |
| `PPPP` | Total price in cents (max $99.99) |
| `K` | Standard UPC-A check digit (modulo 10) |

Example: PLU `8137` + `316` cents → `208137803161`

## API overview (planned)

```
POST   /auth/login

GET    /categories
GET    /categories/:id/products
GET    /products/by-plu/:plu
GET    /products?search=&mode=contains|startsWith
GET    /container-presets
POST   /labels
GET    /labels?limit=

CRUD   /products /categories /specials      (role: office)

POST   /tickets  /tickets/next  /tickets/repeat  /tickets/:id/cancel
GET    /tickets/status
```

Interactive docs (Swagger UI) will be available once Phase 0 is done.

## Project structure (planned)

```
src/
  modules/
    products/     routes, controller, service, repository, schema, validation, tests
    categories/
    specials/
    labels/       pricing and barcode logic
    queue/
    auth/
  shared/         config, errors, middleware, money and time utils
  app.ts
  server.ts
docs/             project plan and documentation
```

## Getting started

🚧 Setup instructions will be added during Phase 0 (`chore/backend-setup`).

## Roadmap

- [ ] **Phase 0** — Setup: repos, TypeScript, linting, env variables, Swagger skeleton, seed data
- [ ] **Phase 1** — Scale core: products, PLU lookup, search, tare, pricing, barcode, labels
- [ ] **Phase 2** — Office and auth: JWT, roles, CRUD, specials
- [ ] **Phase 3** — Queue: tickets and live display with WebSockets
- [ ] **Phase 4** — Offline mode: local catalog cache and label sync
- [ ] **Phase 5** — Polish: tests, screenshots, deploy, demo video

## Documentation

- [Project plan](docs/project-plan.pdf)

## License

[MIT](LICENSE)
