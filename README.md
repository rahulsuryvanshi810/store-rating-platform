# Store Rating Platform

Customers rate registered stores from 1 to 5. One login system, three roles, different
functionality per role.

**Backend:** Express (Node.js) · **Database:** PostgreSQL · **Frontend:** React (Vite)

---

## Login credentials

| Role | Email | Password |

| --- | --- | --- |
| System Administrator | `admin@storeratings.com` | `Admin@12345` |
| Normal User | `aarav.sharma@example.com` | `User@12345` |
| Store Owner | `anil.deshpande@example.com` | `Owner@12345` |

Other seeded accounts use the same passwords: `priya.iyer@`, `rohan.das@`, `meera.nair@`,
`vikram.menon@` (normal users) and `sunita.pillai@`, `farhan.qureshi@`, `divya.narayanan@`
(store owners), all `@example.com`.

---

## Run it

### Option A — full stack in Docker

Everything including the database, built and served in production mode:

```bash
docker compose -f docker-compose.prod.yml up -d --build
docker compose -f docker-compose.prod.yml exec api npm run db:setup
```

Open <http://localhost:8080>.

Nginx serves the built React app and proxies `/api` to the Node container, so both run on one
origin. Stop with `docker compose -f docker-compose.prod.yml down`, add `-v` to also drop the
database volume.

### Option B — local development

Requires Node.js 18+ (developed on Node 22).

```bash
docker compose up -d                  # PostgreSQL only

cd server
cp .env.example .env
npm install
npm run db:setup                      # create schema, load demo data
npm run dev                           # http://localhost:4000

cd ../client
npm install
npm run dev                           # http://localhost:5173
```

Vite proxies `/api` to the backend, so no CORS setup is needed.

Database scripts: `db:migrate` (schema only), `db:seed` (demo data), `db:setup` (both),
`db:reset` (drop, recreate, reseed).

---

## Roles

**System Administrator** — dashboard with total users, stores and ratings; add stores, normal
users, admins and store owners; view all stores (name, email, address, rating) and all users
(name, email, address, role); filter every listing by name, email, address and role; view full
user details, including average rating and owned stores for store owners.

**Normal User** — sign up and log in; update password; view and search all stores by name and
address; each row shows the store name, address, overall rating, the user's own rating and an
action to submit or modify it; submit ratings from 1 to 5 and change them at any time.

**Store Owner** — log in and update password; dashboard with average rating, total ratings and
rating distribution; a list of every customer who rated their stores.

All tables sort ascending and descending on key fields, and all listings are paginated server
side.

---

## Database schema

Three tables with identity primary keys and `created_at` / `updated_at` maintained by triggers.

**users** — `id`, `name` `VARCHAR(60)`, `email` `CITEXT` unique, `password_hash`, `address`
`VARCHAR(400)`, `role` enum (`SYSTEM_ADMIN`, `NORMAL_USER`, `STORE_OWNER`).

**stores** — `id`, `name` `VARCHAR(60)`, `email` `CITEXT` unique, `address` `VARCHAR(400)`,
`owner_id` nullable FK to `users` (`ON DELETE SET NULL`).

**ratings** — `id`, `user_id` FK, `store_id` FK (both `ON DELETE CASCADE`), `score` `SMALLINT`.
Unique on `(user_id, store_id)`, so one rating per user per store and re-rating is an upsert.

Check constraints mirror every API validation rule, so the data stays valid regardless of write
path. Indexes on `users.role`, `lower(users.name)`, `stores.owner_id`, `lower(stores.name)`,
`ratings.store_id`, `ratings.user_id`. The `store_rating_summary` view exposes average rating and
count per store. Uses `citext` for case-insensitive email.

---

## Validation

Enforced in the browser, in the API and by database constraints.

| Field | Rule |
| --- | --- |
| Name | 20 to 60 characters |
| Address | Maximum 400 characters |
| Password | 8 to 16 characters, at least one uppercase letter and one special character |
| Email | Standard email format |
| Rating | Whole number, 1 to 5 |

The name rule applies to store names as well as user names.

---

## API

Base path `/api`. Protected routes expect `Authorization: Bearer <token>`.

| Method | Path | Access | Description |
| --- | --- | --- | --- |
| `POST` | `/auth/register` | Public | Register a normal user |
| `POST` | `/auth/login` | Public | Sign in |
| `GET` | `/auth/me` | Signed in | Current user |
| `PATCH` | `/auth/password` | Signed in | Change password |
| `GET` | `/admin/dashboard` | Admin | Totals, top stores, distribution |
| `GET` | `/admin/users` | Admin | Users, filters `name` `email` `address` `role` |
| `POST` | `/admin/users` | Admin | Create a user with any role |
| `GET` | `/admin/users/:id` | Admin | User detail, plus owned stores for owners |
| `GET` | `/admin/owners` | Admin | Owners available for assignment |
| `GET` | `/admin/stores` | Admin | Stores, filters `name` `email` `address` |
| `POST` | `/admin/stores` | Admin | Create a store, optionally assigning an owner |
| `GET` | `/stores` | Normal user | Stores with overall and own rating, filters `name` `address` |
| `PUT` | `/stores/:storeId/rating` | Normal user | Submit or update a rating |
| `GET` | `/owner/dashboard` | Owner | Average, totals, distribution |
| `GET` | `/owner/raters` | Owner | Customers who rated the owner's stores |

Listings accept `page`, `limit` (max 100), `sortBy` and `order`. Sort keys are whitelisted per
endpoint, so they cannot be used for injection. Responses:

```json
{
  "data": [],
  "meta": { "page": 1, "limit": 10, "totalItems": 0, "totalPages": 1 },
  "sort": { "sortBy": "name", "order": "asc" }
}
```

Validation failures return `{ "message": "Validation failed", "errors": { "field": "reason" } }`.

---

## Deploying elsewhere

The compose file in Option A is the deployment unit and runs anywhere Docker does. For a managed
host, build the two images and set these on the API service:

```
NODE_ENV=production
CLIENT_ORIGIN=https://your-domain
DATABASE_URL=postgresql://user:password@host:5432/store_ratings
DATABASE_SSL=true
JWT_SECRET=<long random string>
BCRYPT_ROUNDS=12
```

Run `npm run db:migrate` once against the production database. Skip `db:seed` unless you want
the demo accounts.

To host the frontend separately from the API, set `VITE_API_BASE_URL` to the full API URL at
build time and set `CLIENT_ORIGIN` so CORS allows the site. Whatever serves the static files must
fall back to `index.html`, since the app uses HTML5 routing.

In place already: Helmet headers, restricted CORS origin, 100 kB body limit, bcrypt hashing, JWT
verified against the database on every request, role checks on every protected route,
parameterised queries, whitelisted sort columns.

---

## Troubleshooting

**`Unable to reach the database`** — Postgres is not up or `DATABASE_URL` is wrong. Check
`docker compose ps`.

**Port already in use** — another Postgres on 5432 or service on 8080. Change the published port
in the compose file and in `DATABASE_URL`.

**Signed out unexpectedly** — the token lives in `localStorage`; clearing site data ends the
session, and any 401 outside login clears it automatically.

**Migration fails on `citext`** — the database user cannot create extensions. Run
`CREATE EXTENSION citext;` once as a superuser and re-run.
