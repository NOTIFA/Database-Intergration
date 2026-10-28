# Oracle Ledger

An inventory/products ledger: an Express REST API backed by **MongoDB**
(Mongoose), and a **React** frontend for managing stock (add / edit / delete
/ view products).

```
oracle-ledger/
├── backend/    Express REST API + MongoDB (Mongoose) — Task 3
└── frontend/   React (Vite) UI that calls the API — Task 1
```

## Run it locally

The commands below use Bash (for example, Git Bash). Make sure your local
MongoDB server is already running, or configure the API to use a MongoDB Atlas
connection string as described below.

```bash
cd backend
cp .env.example .env
npm ci
npm start
```

The API reads its database address from `MONGODB_URI` in `backend/.env`.
`.env.example` defaults to a local MongoDB (`mongodb://127.0.0.1:27017/oracle_ledger`,
start one with `docker compose up -d`).

To use **MongoDB Atlas** instead, put your connection string in `backend/.env`:

```
MONGODB_URI=mongodb+srv://<username>:<password>@<cluster-host>/oracle_ledger?retryWrites=true&w=majority
```

Common mistakes that stop the API from connecting:

- The line must start with `MONGODB_URI=` (a bare `mongodb+srv://...` line is ignored).
- Remove the `<` `>` around the password; they are only placeholders.
- Add the database name (`/oracle_ledger`) before the `?`.
- URL-encode special characters in the password (`@ : / ? # %`).
- In Atlas, allow your IP under **Network Access**, and make sure the database
  user exists under **Database Access**.

Keep real credentials in `.env`; do not commit that file.

In another Bash terminal, install and start the frontend:

```bash
cd frontend
npm ci
npm run dev
```

Open the URL Vite prints (usually `http://localhost:5173`).

### Optional demo data

To load the sample categories, suppliers, and products, stop the API and run
this once from the `backend` folder:

```bash
npm run seed
```

**Seeding deletes all existing products, categories, and suppliers before
inserting the demo data. Do not run it against a database containing data you
want to keep.** The API works without seeding; it creates default category and
supplier records when a product is created without them.

To build a static frontend bundle, run `npm run build` in `frontend/` (output
in `frontend/dist/`), then `npm run preview` to serve it.

If your API runs somewhere other than `http://localhost:3000`, copy
`frontend/.env.example` to `frontend/.env` and change `VITE_API_BASE`. If the
frontend runs from a different origin, set `FRONTEND_ORIGIN` in `backend/.env`
to that exact origin (scheme, hostname, and port), then restart the API.

## What you'll see

- A **connection dot** in the header that turns green once the app reaches the API.
- A **New entry** form on the left — add a product's name, price, and quantity;
  it POSTs to the API and the ledger refreshes.
- A **ledger table** on the right showing everything fetched from
  `GET /api/products`, with inline **Edit** (PUT) and **Delete** (DELETE)
  buttons per row.
- Clear **loading**, **error**, and **empty** states in the table instead of a
  blank screen while data is in flight or missing.

## How the frontend is put together (Task 1 objectives)

| Objective | Where |
|---|---|
| Project set up with a JS framework | React 18 + Vite (`frontend/`) |
| Functional components | Every component in `src/components/` is a function component; no classes |
| State management | `src/hooks/useProducts.js` — a custom hook holding products/status/error/connection and the CRUD actions; local `useState` in `ProductForm`/`ProductRow` for form-level and per-row state |
| API calls + loading states | `src/api/products.js` is the only module that calls `fetch`; `useProducts` exposes a `status` of `"loading" \| "ready" \| "error"` that `ProductTable` renders directly |
| Reusable UI components | `ConnectionStatus`, `ProductForm`, `ProductRow`, `ProductTable` all take props and have no hidden dependencies on where they're used |

```
frontend/src/
├── main.jsx                 # React root
├── App.jsx                  # wires the hook to the page
├── styles.css                # same visual identity as the original design
├── api/
│   └── products.js           # fetch wrapper — the only place URLs/HTTP live
├── hooks/
│   └── useProducts.js        # data + loading/error/connection state, CRUD actions
└── components/
    ├── Header.jsx
    ├── ConnectionStatus.jsx
    ├── ProductForm.jsx        # controlled form, own submit/loading state
    ├── ProductTable.jsx       # loading / error / empty / list states
    ├── ProductRow.jsx         # view mode <-> inline edit mode
    └── Footer.jsx
```

## Task 3 — Database Integration

The API now persists to MongoDB via Mongoose instead of an in-memory array.

| Objective | Where |
|---|---|
| MongoDB with Mongoose | `config/db.js` connects on boot; every model/query uses Mongoose |
| Models + relationships between collections | `models/Product.js` references `models/Category.js` and `models/Supplier.js` via `ObjectId` refs; `GET /api/products` populates both so responses include the category/supplier name, not just an id |
| Indexing and optimization | Unique index on `Product.sku`, a text index on `Product.name` (powers `?search=`), a compound `{ category, quantity }` index for filtered/sorted listing, unique indexes on `Category.name` and `Supplier.name`; list queries use `.lean()` and pagination (`?page`, `?limit`) instead of loading the whole collection |
| Data validation before saving | Schema-level Mongoose validators (`required`, `min`, `minlength`/`maxlength`, a custom integer check on quantity, an email-format validator on suppliers) run on every create/update; a `pre("save")` hook additionally confirms a product's `category`/`supplier` ids actually exist before saving; all of it is translated into clean `400`/`404`/`409` JSON responses by `middleware/errorHandler.js` |

New endpoints, same response shape as before (`{ success, data }` / `{ success, error }`):

```
GET    /api/categories        POST /api/categories
GET    /api/categories/:id    PUT  /api/categories/:id     DELETE /api/categories/:id
GET    /api/suppliers         POST /api/suppliers
GET    /api/suppliers/:id     PUT  /api/suppliers/:id      DELETE /api/suppliers/:id
```

`POST`/`PUT` on `/api/products` now accept optional `categoryId` and
`supplierId` fields. The existing React frontend never sends these, so the
API auto-assigns an `"Uncategorized"` category and `"Unknown Supplier"` the
first time they're needed — the old frontend keeps working unmodified. If you
want category/supplier pickers in the UI, that's a small addition to
`ProductForm.jsx` (a `<select>` populated from `GET /api/categories` /
`GET /api/suppliers`) — say the word and I'll add it.

## Note on CORS

The API allows browser requests only from the origin configured by
`FRONTEND_ORIGIN` (defaults to `http://localhost:5173`). CORS is a browser
policy, **not authentication**: this coursework/demo API has no login or
authorization, and non-browser clients can still call it. Do not expose it
publicly or store sensitive data without adding authentication and
authorization.

## Note on `npm audit`

Run `npm audit` in each project folder to check dependency advisories. Review
the advisories and their impact before applying fixes; avoid `npm audit fix
--force`, which may introduce breaking dependency upgrades.
