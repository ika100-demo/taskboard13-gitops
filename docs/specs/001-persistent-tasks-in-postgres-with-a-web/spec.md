---
spec_id: 001-persistent-tasks-in-postgres-with-a-web
title: Persistent tasks in Postgres with a web task board
status: done
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks in Postgres with a web task board

## Problem

The taskboard13 Task API keeps tasks in memory, so every restart or redeploy of the API empties the board, and nobody can rely on a task still being there. The only way to see or add tasks is to call the API by hand, because the web component has no features yet. This spec makes the board usable: tasks are kept in a Postgres database that belongs to the system, and people see and create tasks in the web UI.

## Stories

As a team member, I want tasks I created to still be on the board after the API restarts or is redeployed, so that I can rely on the board.
**Repos:** taskboard13-gitops (declares the Postgres database and gives the API access to it), taskboard13-api

As a team member, I want to open the task board in my browser and see every task with its due date, so that I know what is on the board without calling the API.
**Repos:** taskboard13-web, taskboard13-gitops (wires the web to the API)

As a team member, I want to add a task with a title and an optional due date in the browser, so that I can record work without calling the API.
**Repos:** taskboard13-web, taskboard13-api

## Acceptance criteria

- **AC-001.1** Given tasks A, B and C were created in that order, when the API is restarted or redeployed, then `GET /tasks` returns `200` with A, B and C in that order, each with the same `id`, `title` and `due_date` as when it was created.
- **AC-001.2** Given tasks exist, when a person opens the web UI's start page, then it lists every task the API returns, oldest first, each showing its title and its due date as `YYYY-MM-DD`, or "No due date" when it has none; given no tasks exist, the page shows "No tasks yet".
- **AC-001.3** Given a task whose title is `<b>Buy milk</b>`, when a person opens the start page, then the title is shown literally as the text `<b>Buy milk</b>` and is not interpreted as markup.
- **AC-001.4** Given the start page is open, when a person enters the title "Buy milk" and the due date 2026-12-01 and submits the form, then "Buy milk" with 2026-12-01 appears at the end of the list without a manual page reload, the form is cleared, and `GET /tasks` on the API also returns the task.
- **AC-001.5** Given the start page is open, when a person submits the form with a title only, then the task appears at the end of the list with "No due date".
- **AC-001.6** Given the start page is open, when a person submits the form with an empty or whitespace-only title or a title longer than 200 characters, then a message next to the title field says what is wrong, the entered values stay in the form, and no task is created (the list and `GET /tasks` are unchanged).
- **AC-001.7** Given the database is unreachable, when a client calls `GET /tasks` or `POST /tasks`, then the API answers `503` with a JSON body holding an `error` message and creates no task, and the API's readiness endpoint reports not ready until the database is reachable again.
- **AC-001.8** Given the API cannot list tasks (unreachable or answering an error), when a person opens the start page or submits the form, then the page shows "Tasks could not be loaded. Please try again." (or "The task could not be saved. Please try again." for the form) instead of an empty list, and keeps the entered values in the form.

## Non-goals

- Authentication and access control: the web UI and the API stay open with no login, like the API today, so anyone who can reach them can read and add tasks; login, user accounts and per-user boards are a later spec.
- Editing, completing, deleting or reordering tasks, in the API or in the web UI.
- Filtering, searching, sorting options or pagination.
- Fields beyond title and due date.
- Moving tasks from the current in-memory API into the database: in-memory tasks are lost on every restart anyway, so there is nothing to migrate.
- Database backups, restore and copying data between environments.
- Live updates of the list when another person adds a task (a reload shows it, AC-001.2).

## Assumptions

- **A1** The API keeps its existing contract (`POST /tasks`, `GET /tasks`, fields `id`, `title`, `due_date`, `201`/`200`/`422` as today); only the storage changes (other option: a new versioned API; affects AC-001.1, AC-001.4).
- **A2** One Postgres database per environment, declared in the system (in-cluster, no paid service); environments never share task data (other option: one shared database; affects AC-001.1).
- **A3** The board is the web UI's start page `/`; the form sits above the list on the same page (other option: a separate `/tasks` page or a dialog; affects AC-001.2, AC-001.4).
- **A4** The web UI shows the list oldest first, the API's order, and due dates exactly as `YYYY-MM-DD` (other option: newest first, or a localized date format; affects AC-001.2, AC-001.4, AC-001.5).
- **A5** The web UI checks title rules (1 to 200 characters, not whitespace only) before sending and also shows the API's `422` as the same field message, so both paths behave as AC-001.6 (other option: rely on the API only; affects AC-001.6).
- **A6** The due date is entered with a date picker, so an invalid date cannot be submitted from the web UI; the API's `422` for `due_date` stays as today (other option: free text with its own error message; affects AC-001.4).
- **A7** When the database is down the API answers `503` and reports not ready, rather than falling back to memory (other option: keep serving from memory; affects AC-001.7).
- **A8** Error and empty-state wording in English as written in AC-001.2, AC-001.6 and AC-001.8 (other option: other wording or languages; affects those criteria).
- **A9** The web UI reaches the API over the system's internal wiring; the browser never needs the API's public address (other option: the browser calls the exposed API directly; affects AC-001.2, AC-001.4).

## Open questions

- ~~Tasks now persist and the web UI is exposed with no login, so anyone who can reach the web UI or the API can read every task and add tasks. Is that acceptable for this spec? (suggested: yes, open as the API is today; login is a later spec; affects AC-001.2, AC-001.4 and the Non-goals)~~ Answered 2026-11-13: yes, open for now; folded into Non-goals (authentication and access control).

## Changelog

- 2026-11-13 created
- 2026-11-13 amended: access question answered (open, no login, for now); added authentication and access control as a non-goal; criteria unchanged
- 2026-11-13 approved
- 2026-11-13 building
- 2026-11-13 done
