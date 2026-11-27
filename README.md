# Piggyway: running a multi-service commerce platform locally

Piggyway is a small e-commerce and guinea pig boarding business with a real stack behind it: five private repositories, four runtime services, three environments, and content that lives in a CMS that writes to the same Postgres database the API reads from.

This repository is a case study, not the code. The source belongs to the company and stays private. What follows is what the system looks like, what I did on it, and what I learned from getting it to run from nothing.

## What the product does

- A storefront for small pet products: hiders, cage liners, treats, with product variants (size and colour) and stock tracking.
- Guest and logged-in carts, checkout through Stripe, and order history.
- A guinea pig boarding side: booking requests, pickup slots, and boarding agreements that customers sign and the team manages.
- Email flows: sign-in codes, order receipts, contact form, and an inbound mail router that archives every message to object storage.

## Architecture

| Service | Technology | Role |
| --- | --- | --- |
| Storefront | Next.js 16, React 19, Tailwind 4 | Server rendered pages, API routes that act as a backend for frontend |
| API | Bun, Hono, Zod OpenAPI, Drizzle ORM | Business logic, authentication, order and boarding workflows |
| CMS | Directus 12 | Content and operations editing, writes to the same Postgres database |
| Mail router | Cloudflare Worker with R2 | Receives inbound mail, archives the raw message, notifies the API |
| Database | Postgres 16 | Single database, schema owned by API migrations and Directus system tables |
| Runtime | Cloudflare Workers, Hyperdrive, R2 | Production hosting for the API, the storefront and the CMS container |

The CMS and the API share one database on purpose: the API owns the physical tables, and Directus owns only its own system tables plus the collection metadata that points at those tables.

## My role

I was not the author of the application code. My work was to run the stack end to end, verify it behaves correctly, and fix what blocked that.

- Built a local environment from nothing: Postgres, the Bun API, Next.js, Directus in Docker, and two Cloudflare Workers under wrangler.
- Wrote the local run book: start and stop scripts, a health check script, and a mail flow test.
- Tested every user-facing flow with a real browser, not only with curl: browsing, variant selection, cart, login, account, contact form, CMS editing, mobile viewport.
- Found and fixed a data bug that made eight of twenty products unreachable (details below), and opened the pull request upstream.

## The bug worth writing about

**Symptom:** opening any product in the largest category returned `ERR_TOO_MANY_REDIRECTS`. The category filter rendered the not-found page.

**Cause:** a rename happened in code but not in the data.

The frontend had renamed the category route from `hut` to `hideout` and added a permanent redirect from `/shop/hut/:slug` to `/shop/hideout/:slug`. The production CMS had been renamed by hand. But the seed file in the backend still inserted `hut`, and the product page also redirects to whatever category slug the database reports. Any environment rebuilt from seeds therefore ping-ponged:

```
/shop/hut/<slug>        -> 308 -> /shop/hideout/<slug>
/shop/hideout/<slug>    -> 308 -> /shop/hut/<slug>     (stored slug)
```

**How I found it:** the API returned 200 with an empty list, not an error, so the failure was not in the query. I compared the three sources of truth: the redirect rules in `next.config.ts`, the canonical redirect in the product layout, and the row in `ref_product_category`. The git history showed the rename commit in the frontend one day after the last seed change, with no matching migration. Pulling the production API confirmed production stores `hideout`, so production looked fine for a reason that had nothing to do with the code being better.

**Fix:** update the seed data in both the TypeScript and SQL seeders, and add an idempotent migration so databases seeded before the rename converge. The migration skips the update when a `hideout` row already exists and updates zero rows when run twice, which matters because the same file runs against production, where the row is already renamed.

## Things that made the local build harder than expected

- `bun run db:migrate` cannot build an empty database: no migration creates one of the core tables that a later migration alters. The schema has to come from the TypeScript definitions with `drizzle-kit push` instead.
- Directus refuses to start when its own system tables were created by that push, because the migration history does not match. Those tables have to be dropped so Directus creates them itself.
- The seed writes no publication status, so the product API returns an empty list even though rows exist.
- Two Cloudflare Workers only see each other through a shared local registry: the wrangler version comes from each project lockfile and they must agree on the persist directory. Object storage writes by one worker are not visible to an already running second worker until it restarts.
- The storefront caches catalogue responses on the server for five minutes, which hides changes during verification.

## How I verify it

- A health check script that asserts every service endpoint, every page, and the row counts.
- A browser suite of 32 checks: API contracts, browsing, variant pricing, cart mutations with database assertions, login with a real session, account page, contact form, CMS to API propagation, and mobile layout.
- A mail test that delivers a raw MIME message, restarts the worker so it sees the archived object, calls the ingest webhook, and prints the stored rows.

## What I learned

- A 200 response with empty data can be a worse bug than a 500, because nothing looks broken until a user sees a blank page.
- When code and data disagree, the data usually wins quietly. Migrations that only change code are half a migration.
- Local environments rot. Anything that has to be done by hand, in order, will be done wrong later, so it belongs in a script with a health check next to it.
- Reading a stack is different from running it. Half of the problems above only appear when the services start talking to each other.

## Links

- Storefront: https://piggyway.com.au
- Public API: https://api.piggyway.com.au/doc
