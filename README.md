# Todo / Shopping List / Water Tracker API

A beginner-friendly but production-structured REST API.

> **Note:** this sandbox has no internet access, so `npm install` couldn't be
> run here. Follow the steps below on your own machine — everything is
> standard, well-tested tooling (Express, Prisma, Zod, Jest).

## 1. Prerequisites

- Node.js 18+ installed
- A PostgreSQL database running somewhere (local install, or a free one from
  Supabase / Neon / Railway if you don't want to install Postgres yourself)

## 2. Setup

```bash
cd todo-api
npm install
cp .env.example .env
```

Open `.env` and put your real database connection string in `DATABASE_URL`.
It looks like:

```
DATABASE_URL="postgresql://myuser:mypassword@localhost:5432/todo_db?schema=public"
```

## 3. Create the database tables

Prisma reads `prisma/schema.prisma` and creates the actual tables for you:

```bash
npx prisma migrate dev --name init
npx prisma generate
```

You should see `Task`, `ShoppingItem`, and `WaterLog` tables get created.

## 4. Run the API

```bash
npm run dev
```

You'll see `todo-api listening on http://localhost:3000`.

## 5. Try it out

```bash
# Create a task
curl -X POST http://localhost:3000/tasks -H "Content-Type: application/json" \
  -d '{"title":"Buy groceries","priority":"HIGH"}'

# List tasks
curl http://localhost:3000/tasks?page=1&limit=10

# Update a task
curl -X PUT http://localhost:3000/tasks/<id> -H "Content-Type: application/json" \
  -d '{"status":"DONE"}'

# Delete a task
curl -X DELETE http://localhost:3000/tasks/<id>

# Shopping list
curl -X POST http://localhost:3000/shopping-list -H "Content-Type: application/json" \
  -d '{"name":"Milk","quantity":2}'
curl -X PATCH http://localhost:3000/shopping-list/<id>/toggle   # tick as bought

# Water tracker
curl -X POST http://localhost:3000/water/log -H "Content-Type: application/json" \
  -d '{"glasses":1}'
curl http://localhost:3000/water/today
```

## 6. Run the tests

```bash
npm test
```

This runs Jest against the **service layer only** (per the assignment),
with the repository layer mocked out so no real database is touched.
Coverage report prints to the terminal and to `coverage/` — thresholds
are set to 90% in `jest.config.js`.

## How the code is organized (and why)

```
src/
  errors/         custom error classes (NotFoundError, ValidationError, ...)
  validations/     Zod schemas — the ONLY place request shape is validated
  repositories/    the ONLY layer allowed to import Prisma / talk to the DB
  services/        business logic, calls repositories, throws custom errors
  routes/          thin — parse with Zod, call a service, send a response
  middlewares/      errorHandler (turns errors into JSON + status codes),
                    asyncHandler (catches promise rejections in routes)
  app.ts            wires everything together
  server.ts         starts the HTTP server
tests/
  *.service.test.ts  unit tests for the service layer, repository is mocked
```

**Route → Service → Repository**, one direction only. Routes never touch
Prisma. Services never touch `req`/`res`. This is what makes the service
layer trivial to unit test (mock the repository, no real DB needed) and
easy to reason about as the app grows.

**Why transactions on every write**, even single-statement ones? Because if
you ever need to add a second write to the same operation (e.g., writing to
an audit-log table whenever a task is deleted), the transaction wrapper is
already there — you just add the second repository call inside it. Nothing
in the route or service *signature* has to change.

**Error flow:** a repository throws Prisma's own errors → a service catches
"not found" conditions and throws `NotFoundError`/`ValidationError` →
`asyncHandler` catches anything thrown/rejected in a route → `errorHandler`
(registered last in `app.ts`) turns it into the right JSON + status code.
