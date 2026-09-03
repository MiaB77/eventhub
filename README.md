# EventHub

A full-stack event ticket booking platform **plus** its end-to-end QA automation
layer. Users browse events, book tickets, manage bookings, and create their own
events; each user operates in an isolated sandbox.

| Layer | Technology |
|---|---|
| Frontend | Next.js 14 (App Router), React 18, TypeScript, Tailwind CSS, React Query v5 |
| Backend | Express.js, Prisma ORM, MySQL 8+, Swagger UI |
| Auth | JWT (7-day expiry), bcryptjs |
| E2E tests | Playwright (Chromium) |
| CI/CD | GitHub Actions |

---

## What's in this repo

| Path | Purpose |
|---|---|
| `backend/` | Express API — layered as Routes → Controllers → Services → Repositories → Prisma |
| `frontend/` | Next.js 14 app (App Router) |
| `tests/` | Playwright E2E specs |
| `docs/` | Test scenario catalogue and pyramid strategy |
| `.github/workflows/` | `ci.yml` (PR checks), `deploy.yml` (production), `playwright.yml` (E2E on `main`) |
| `.claude/` | Claude Code skills + slash-command agents for test authoring/review |
| `Dockerfile`, `docker-compose.yml` | Containerised Playwright test runner |
| `playwright.config.ts` | E2E config — targets the deployed site, Chromium only |

### Repository layout

```
eventhub/
├── package.json              # Root scripts (dev, setup, seed, test, …)
├── playwright.config.ts
├── Dockerfile / docker-compose.yml
│
├── backend/
│   ├── app.js  server.js
│   ├── prisma/               # schema.prisma, migrations, seed.js
│   └── src/
│       ├── config/           # database, env, swagger
│       ├── routes/  controllers/  services/  repositories/
│       ├── validators/  middleware/  utils/
│
├── frontend/
│   ├── app/                  # /, /login, /register, /events, /bookings, /admin
│   ├── components/           # ui/, events/, bookings/, layout/, auth/
│   ├── lib/                  # api clients, hooks, providers
│   └── types/
│
├── tests/
│   └── booking-management.spec.js
│
├── docs/
│   ├── test-scenarios.md     # 53 scenarios, TC-001 … TC-510
│   └── test-strategy.md      # scenario → pyramid-layer assignment
│
└── .claude/
    └── skills/               # eventhub-domain, generate-tests, review-tests,
                              # create-scenarios, test-strategy, playwright-best-practices
```

---

## Prerequisites

- **Node.js 18+** (CI uses 21.7.1)
- **npm**
- **MySQL 8+** — only needed to run the application locally; the E2E suite runs
  against the hosted site and needs no database.

---

## Running the application

### 1. Install dependencies

```bash
npm run setup          # installs backend/ and frontend/ deps
```

### 2. Create the database

```bash
mysql -u root -p -e "CREATE DATABASE eventhub CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

### 3. Configure environment

**`backend/.env`** (copy from `backend/.env.example`):

```env
DATABASE_URL="mysql://root:your_password@localhost:3306/eventhub"
PORT=3001
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
JWT_SECRET=change_me
SHOW_EXPLORE_LINKS=false
```

**`frontend/.env.local`** (copy from `frontend/.env.local.example`):

```env
NEXT_PUBLIC_API_URL="http://localhost:3001/api"
```

### 4. Push schema, generate client, seed

```bash
npm run db:push                         # push schema to the DB
npm run prisma:generate --prefix backend # generate the Prisma client
npm run seed                            # 10 sample events, 5 categories, 5 cities
```

> `npm run migrate` uses a migration-file workflow instead of `db:push` (interactive).

### 5. Start both servers

```bash
npm run dev
```

| Service | URL |
|---|---|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:3001 |
| Swagger UI | http://localhost:3001/api/docs |

### Root scripts

| Script | Description |
|---|---|
| `npm run dev` | Start frontend + backend concurrently |
| `npm run setup` | Install deps in `backend/` and `frontend/` |
| `npm run seed` | Insert 10 sample events |
| `npm run db:push` | Push Prisma schema (no migration files) |
| `npm run migrate` | `prisma migrate dev` (interactive) |
| `npm run build` | Production build of the frontend |
| `npm run lint` | Lint the frontend |
| `npm test` | Run the Playwright E2E suite |
| `npm run test:ui` | Playwright UI mode |
| `npm run test:report` | Open the last HTML report |

---

## End-to-end tests

The Playwright suite drives the **deployed** application — no local servers
required.

```bash
npm ci                              # install root deps
npx playwright install chromium     # one-time browser download
npm test                            # run all specs (line reporter via config)

npx playwright test tests/booking-management.spec.js   # single file
npx playwright test -g "TC-102"                        # single test by title
npx playwright test --headed                           # watch it run
npm run test:report                                    # open HTML report
```

### Configuration (`playwright.config.ts`)

- `baseURL`: `https://eventhub.rahulshettyacademy.com`
- Chromium only, `fullyParallel: false`, `retries: 0`
- `timeout: 30s`, `expect.timeout: 15s` (widened for live-site latency)
- `screenshot: only-on-failure`, `video: retain-on-failure`, `reporter: html`

### Current specs — `tests/booking-management.spec.js`

| Test | Checks |
|---|---|
| TC-001 | Booking card renders on the bookings list |
| TC-002 | Booking detail page shows every section |
| TC-003 | Cancel from detail page → toast + redirect |
| TC-004 | "Clear all bookings" → empty state |
| TC-006 | "View My Bookings" link navigates to the list |
| TC-102 | Booking ref starts with the event title's first letter (uppercase) |

> Tests share one live account and run serially — each is self-contained
> (login → clear state → act → assert). Test account: `rahulshetty1@gmail.com` / `Magiclife1!`.

### Docker

```bash
docker compose run --rm tests     # runs the suite in the Playwright image;
                                  # report is written to ./playwright-report
```

---

## CI/CD

| Workflow | Trigger | Does |
|---|---|---|
| `ci.yml` | PRs to `main` | Backend checks, Prisma schema-drift check, frontend checks |
| `playwright.yml` | Push to `main`, manual | Runs the full E2E suite, uploads the HTML report (30-day retention) |
| `deploy.yml` | Push to `main`, manual | Pre-deploy checks, then production deploy |

---

## Test documentation

- **`docs/test-scenarios.md`** — 53 scenarios (TC-001 … TC-510) across happy path,
  business rules, security, negative, edge, and UI-state lenses.
- **`docs/test-strategy.md`** — assigns each scenario to the optimal test-pyramid
  layer (E2E / API / component / unit).

## Claude Code workflow

`.claude/skills/` ships domain knowledge and QA agents used via slash commands:

| Command | Role |
|---|---|
| `/create-scenarios <area>` | Functional Tester — generate scenario docs |
| `/test-strategy <scenarios>` | Test Architect — assign pyramid layers |
| `/generate-tests <feature>` | Test Automation Engineer — write Playwright specs |
| `/review-tests <file>` | Code Reviewer — review test quality |

The `eventhub-domain` and `playwright-best-practices` skills are auto-loaded as
reference. See `CLAUDE.md` for project conventions.

---

## API reference

Base URL: `http://localhost:3001`

### Events

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/events` | List events — `category`, `city`, `search`, `page`, `limit` |
| `GET` | `/api/events/:id` | Single event |
| `POST` | `/api/events` | Create event |
| `PUT` | `/api/events/:id` | Update event |
| `DELETE` | `/api/events/:id` | Delete event (cascades bookings) |

### Bookings

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/bookings` | List bookings (paginated, filter by status) |
| `GET` | `/api/bookings/:id` | Booking by ID |
| `GET` | `/api/bookings/ref/:ref` | Booking by reference code |
| `POST` | `/api/bookings` | Create booking (atomically decrements seats) |
| `DELETE` | `/api/bookings/:id` | Cancel one booking (restores seats) |
| `DELETE` | `/api/bookings` | Clear all of the current user's bookings |

**`POST /api/bookings`**

```json
{
  "eventId": 1,
  "customerName": "Rahul Shetty",
  "customerEmail": "rahul@example.com",
  "customerPhone": "9876543210",
  "quantity": 2
}
```

Response includes a unique `bookingRef` (`<EventInitial>-XXXXXX`), `totalPrice`,
and `status: "confirmed"`.

---

## Key business rules

- Max **6** user-created events, max **9** bookings per user (FIFO pruning on overflow)
- Booking ref's first character = event title's first character (uppercase)
- Seats decrement on booking, restore on cancellation
- Refund eligibility: 1 ticket = eligible, >1 = not eligible (client-side)
- Cross-user booking access returns "Access Denied"
- Seeded ("static") events are immutable

---

## Playwright selectors

Key UI elements expose `data-testid` attributes:

| `data-testid` | Element |
|---|---|
| `event-card` | Event card in listings |
| `book-now-btn` | "Book Now" on an event card |
| `customer-name` / `customer-email` / `customer-phone` | Booking form fields |
| `confirm-booking-btn` | Submit booking |
| `booking-ref` | Reference on the confirmation card |
| `booking-card` | Booking card in "My Bookings" |
| `cancel-booking-btn` | Cancel booking |
| `confirm-dialog-yes` | Confirm button in any confirmation dialog |
| `admin-event-form` / `event-title-input` / `add-event-btn` | Admin event form |
| `event-table-row` / `edit-event-btn` / `delete-event-btn` | Admin events table |
| `nav-events` / `nav-bookings` | Navbar links |
