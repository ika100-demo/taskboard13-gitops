---
plan_id: 001-persistent-tasks-in-postgres-with-a-web
spec: 001-persistent-tasks-in-postgres-with-a-web
feature: Tasks persist in the system's Postgres database and people see and add them on the web task board
gitops_app: ika100-demo/taskboard13-gitops
status: completed
repos:
- id: taskboard13-gitops
  shape: gitops-app
  summary: Declare the Postgres database, give the API access to it and wire the web to the API
  acs:
  - AC-001.1
  - AC-001.2
  - AC-001.4
  order: before-merge
  gitops:
  - resource: postgres
    type: database
  - uses: postgres
    component: taskboard13-api
  - env:
      TASKBOARD13_API_URL: http://taskboard13-api
    component: taskboard13-web
  depends_on: []
  done: true
- id: taskboard13-api
  shape: service-python
  summary: Keep tasks in Postgres so they survive restarts, and answer 503 while the database is unreachable
  acs:
  - AC-001.1
  - AC-001.7
  criteria:
  - Given tasks A, B and C were created with POST /tasks in that order, when the service restarts against
    the same database, then GET /tasks returns 200 with A, B and C in that order, each with the same id,
    title and due_date as in its 201 response (system AC-001.1) (replaces AC-001.7)
  - 'Given the service starts with DATABASE_URL pointing at an empty database, when a client sends GET
    /tasks and then POST /tasks with {"title": "Buy milk"}, then GET answers 200 with [] and POST answers
    201, without any manual schema step (system AC-001.1)'
  - Given the database is unreachable, when a client sends GET /tasks, then the response is 503 with a
    JSON body whose error is a non-empty string (system AC-001.7)
  - Given the database is unreachable, when a client sends POST /tasks with a valid body, then the response
    is 503 with a JSON body whose error is a non-empty string, and once the database is reachable again
    GET /tasks does not contain that task (system AC-001.7)
  - Given the database is unreachable, when the readiness endpoint GET /ready is called, then it answers
    503, while GET /health still answers 200; when the database is reachable again, GET /ready answers
    200 without a restart of the service (system AC-001.7)
  arguments: |
    Database settings come from the injected DATABASE_URL (and PG*), never from a file in the repo.
    The service creates or migrates its own schema at start; creation order must survive restarts
    (store an ordering column, do not rely on row order). id stays 32 lowercase hex chars.
    A request body that fails validation still answers 422 even while the database is down.
  depends_on: []
  done: true
- id: taskboard13-web
  shape: web-nextjs
  summary: Task board start page that lists tasks and adds a task through the Task API
  acs:
  - AC-001.2
  - AC-001.3
  - AC-001.4
  - AC-001.5
  - AC-001.6
  - AC-001.8
  criteria:
  - Given GET /tasks on the API returns tasks, when a person opens the start page /, then it lists every
    returned task in the API's order (oldest first), each with its title and its due_date as YYYY-MM-DD,
    or the text "No due date" when due_date is null (system AC-001.2)
  - Given GET /tasks on the API returns [], when a person opens the start page, then it shows "No tasks
    yet" (system AC-001.2)
  - Given the API returns a task titled <b>Buy milk</b>, when a person opens the start page, then the
    page shows the literal text <b>Buy milk</b> and no <b> element is created from it (system AC-001.3)
  - 'Given the start page is open, when a person enters the title "Buy milk", picks the due date 2026-12-01
    in the date picker and submits the form above the list, then the web sends POST /tasks with {"title":
    "Buy milk", "due_date": "2026-12-01"} and, on 201, "Buy milk" with 2026-12-01 appears at the end of
    the list without a page reload and the form is cleared (system AC-001.4)'
  - Given the start page is open, when a person submits the form with a title and no due date, then the
    web sends POST /tasks with due_date null and the task appears at the end of the list with "No due
    date" (system AC-001.5)
  - Given the start page is open, when a person submits an empty or whitespace-only title or one longer
    than 200 characters, then a message next to the title field says what is wrong, the entered values
    stay in the form, the list is unchanged and no POST /tasks is sent (system AC-001.6)
  - Given the API answers POST /tasks with 422 naming the field title, when a person submits the form,
    then the same kind of message appears next to the title field, the entered values stay in the form
    and the list is unchanged (system AC-001.6)
  - Given the API is unreachable, too slow (over 5 s) or answers a status the contract does not list,
    when a person opens the start page, then it shows "Tasks could not be loaded. Please try again." instead
    of a list; when a person submits the form, then it shows "The task could not be saved. Please try
    again." and keeps the entered values (system AC-001.8)
  arguments: |
    The API base URL comes from the server-side env TASKBOARD13_API_URL (http://taskboard13-api in the
    cluster); the browser never calls the API directly: list loading and form submission go through
    the web's own server side (server component, route handler or server action).
    The start page / replaces the template's landing page.
  depends_on: []
  done: true
gitops_pin: []
---

## Decomposition

Three entries, all at level 0 (no `depends_on`): the contract below is the existing API contract plus a `503`, so the web builds against it in parallel with the API's storage change. The gitops entry runs first (`before-merge`) so that the database and the wiring exist when the components are deployed.

## taskboard13-gitops — Declare the Postgres database, give the API access to it and wire the web to the API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.4 · **Depends on:** []

Declares one CloudNativePG `postgres` database (type `database`) per environment (A2: environments never share data; rendered per instance), gives `taskboard13-api` `DATABASE_URL` and `PG*` through `uses`, and sets `TASKBOARD13_API_URL=http://taskboard13-api` on `taskboard13-web` (the API's Service, port 80, in the same namespace; A9). It owns the infrastructure half of AC-001.1 (a database outlives API restarts) and the wiring half of AC-001.2 and AC-001.4 (the web reaches the API). Both components are already exposed; no exposure, replica or resource change is needed. The first database brings the CloudNativePG operator into `operators/`; `/system:ship` applies `operators/root.yaml` on the local cluster.

## taskboard13-api — Keep tasks in Postgres so they survive restarts, and answer 503 while the database is unreachable

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.7 · **Depends on:** []

Replaces the in-memory `TaskStore` with a Postgres-backed store behind the same `POST /tasks` / `GET /tasks` contract (A1), so tasks survive restarts and redeploys (AC-001.1). Its own spec `001-create-and-list-tasks-in-memory` criterion AC-001.7 ("restart returns `[]`") becomes false and is superseded by the first criterion here. It owns the whole of AC-001.7: `503` with `{"error": ...}` on both endpoints while the database is unreachable, no task created, and `/ready` not ready until the database is back (A7: no memory fallback).

## taskboard13-web — Task board start page that lists tasks and adds a task through the Task API

**Shape:** `web-nextjs` · **Criteria:** AC-001.2, AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.8 · **Depends on:** []

Turns the template's start page `/` into the task board: the form (title, optional due date via a date picker, A6) above the list (A3), oldest first and dates as `YYYY-MM-DD` (A4), titles rendered as text (AC-001.3), client-side title rules plus the API's `422` shown as the same field message (A5), and the English messages of AC-001.2, AC-001.6 and AC-001.8 (A8). It calls the API from its server side only (A9).

## Contract

Between `taskboard13-web` (caller, server side only) and `taskboard13-api`. Base URL: env `TASKBOARD13_API_URL` (cluster: `http://taskboard13-api`). No auth (non-goal), no prefix, JSON in and out (`Content-Type: application/json`). The existing contract of the API's spec `001-create-and-list-tasks-in-memory` stays unchanged except for the `503` added below.

### `GET /tasks`

- `200` — JSON array of `{"id": string, "title": string, "due_date": "YYYY-MM-DD" | null}`, creation order, oldest first; `[]` when none. Values identical to the `201` responses, also across API restarts and redeploys.
- `503` — database unreachable: `{"error": "<non-empty message>"}`.

### `POST /tasks`

Request: `{"title": string, "due_date": "YYYY-MM-DD" | null}`; `title` 1 to 200 characters with at least one non-whitespace character, stored as sent; `due_date` absent or `null` means no due date; unknown fields ignored. The web always sends both fields (`due_date: null` when none).

- `201` — the created task `{"id": "<32 lowercase hex>", "title", "due_date"}`; it is in every later `GET /tasks`.
- `422` — validation failed, nothing stored; FastAPI body `{"detail": [{"loc": ["body", "<field>", ...], "msg": "<text>", "type": "...", "input": ...}]}`. Validation runs before any database access, so an invalid body answers `422` even while the database is down. The web maps an entry whose `loc` contains `"title"` to the title field message; a `422` for any other field is treated as an unlisted response.
- `503` — database unreachable: `{"error": "<non-empty message>"}`, nothing stored.

### Probes (API)

- `GET /health` — `200` while the process runs, independent of the database (so Kubernetes does not restart the API during a database outage).
- `GET /ready` — `200` when a database round trip (`SELECT 1`) succeeds, `503` otherwise; it turns `200` again by itself when the database is back.

### Errors

- API errors outside validation use `{"error": "<human-readable message>"}` with status `503` (database unreachable or timing out) or `500` (unexpected); validation errors use the FastAPI `422` body above. Messages never contain connection strings or credentials.
- The web treats every response the contract does not list for that call (any other status, a non-JSON or malformed body, a `422` not naming `title`) like an unreachable API: on load "Tasks could not be loaded. Please try again.", on submit "The task could not be saved. Please try again." with the entered values kept. It never shows an empty list or "No tasks yet" in that case, and never retries a `POST` by itself (no duplicate tasks).

### Timeouts

- Web → API: 5 s total per call (`GET /tasks` and `POST /tasks`), connect included; a timeout or refused connection is handled like an unlisted response (above). No automatic retry; the person retries.
- API → database: connect timeout 3 s and statement timeout 5 s; when either hits, the request answers `503` within about 5 s, so it stays inside the web's 5 s budget in the common case. `/ready`'s check uses a 2 s timeout.
- When the database is down the API keeps running (`/health` 200, `/ready` 503) and reconnects on later requests without a restart; Kubernetes takes the API out of its Service while not ready, so the web then gets a refused or timed-out connection, which it shows as the AC-001.8 messages.

## gitops-app PR

The gitops entry lands first, as one PR on `compose/001-persistent-tasks-in-postgres-with-a-web` from `/system:ship`: `cplat resource add postgres`, `uses: postgres` on the API, `TASKBOARD13_API_URL` on the web, then render. Both components run in `dev` only today, and dev tracks `main` (the system ship pins both builds in the same compose PR), so no staging or prod pin is planned; staging follows later through `/instance:promote`.

After Argo reconciles dev, verify: the CloudNativePG cluster for `postgres` is healthy in `taskboard13-dev`; the API pod has `DATABASE_URL` and `/ready` answers `200`; create a task on `http://taskboard13-api.taskboard13-dev.localhost:8088/tasks`, restart the API deployment and see it still returned (AC-001.1); open `http://taskboard13-web.taskboard13-dev.localhost:8088/`, see the task listed and add one with the form (AC-001.2, AC-001.4).
