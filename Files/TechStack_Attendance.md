# Technology Stack

**Project:** Automatic Class Attendance Web Application
**Version:** 1.1  |  **Date:** 2026-11-08
**Related:** `SRS_Attendance.md` (v1.0), `SOW_Attendance.md` (v1.0, §4 Approach and Tools)

Guiding rule: **keep it simple.** Use the Python standard library where it does the job, and add a package only when writing it ourselves would be harder. Versions are pinned in `requirements.txt` when the repository is created (WP1).

---

## 1. The stack

| Layer              | Choice                                                       | Package                                         |
| ------------------ | ------------------------------------------------------------ | ----------------------------------------------- |
| Language           | Python 3.12+                                                 | —                                               |
| Web server         | FastAPI on Uvicorn, one process                              | `fastapi`, `uvicorn`                            |
| Database           | SQLite through the standard library                          | — (`sqlite3`)                                   |
| Pages              | Server-rendered HTML templates + a little vanilla JavaScript | `jinja2`                                        |
| Live updates       | Server-Sent Events (SSE)                                     | `sse-starlette`                                 |
| Tests              | pytest + FastAPI's test client                               | `pytest`, `httpx`                               |
| HTTPS for the demo | Cloudflare quick tunnel                                      | `cloudflared` (a program, not a Python package) |
| Diagrams (D3)      | Mermaid inside Markdown                                      | —                                               |

**Six Python packages in total.** Everything is free.

---

## 2. How each part is done

### Database
- Standard-library `sqlite3`, with plain SQL kept in a few repository classes (for example `StudentRepository` and `SessionRepository`).
- Each SRS §6 entity is a Python `@dataclass`. These classes are also the D3 class diagram.
- Two database rules do real work:
  - `UNIQUE(spot_check_id, student_id)` blocks a second response to the same spot check (FR-4.5), even when two taps arrive at once.
  - `UNIQUE(device_token)` keeps every device token distinct.
- The schema is a single `schema.sql` file, run at startup.

### Scheduler (sessions and spot checks)
- One background loop inside the server checks once a second what is due:
  - open or close a session,
  - open or close a spot check,
  - compute statuses when a session closes.
- It keeps nothing in memory and reads everything from the database, so a server restart mid-lecture just continues.
- It gets the time from a small `Clock` class. Tests swap in a fake clock to "freeze" time.
- Spot-check times are drawn with `secrets.SystemRandom()`, so they can't be predicted.

### Students and device binding
- **Device token:** `secrets.token_urlsafe(32)`, stored in a browser cookie (`HttpOnly`, `Secure`).
  - The browser sends it automatically with every request (FR-2.2).
  - Page scripts can't read it.
- **One active login (FR-2.4):** each student row stores the current login id. A new login overwrites it, and requests carrying the old id are rejected.

### Lecturer and admin login
- FastAPI's built-in HTTP Basic login, with a username and password set in environment variables.
- No user accounts table and no extra package.

### Live updates
- The student page and the dashboard open an `EventSource` connection, which is built into every browser.
- The server pushes "spot check open" and "table changed" events through it.
- Students answer with a normal `POST` request.
- A spot check's time is sent only at the moment it opens (NFR-6).

### CSV
- Standard-library `csv`.
- **Import:** check every row first, then save them all in one transaction. A bad row rejects the whole file and reports the row number.
- **Export:** written as `utf-8-sig` so Excel shows Hebrew names correctly.

### HTTPS for the demo
- Run the server on a laptop, then run `cloudflared tunnel --url http://localhost:8000`.
- That prints a public `https://` address, which works on phones without certificate warnings.

### Load test (WP7)
- One Python script built on `asyncio` + `httpx` (already installed for tests).
- It simulates 200 students, sends 200 responses to one spot check, and reports the response times.

---

## 3. Dropped to keep it simple

| Dropped | What we lose | Effect on the SRS |
|---------|--------------|-------------------|
| Push notifications and PWA install (service worker, `pywebpush`) | Alerts when the student page is closed | FR-4.3 now alerts over SSE only; SRS v1.1 and SOW v1.1 updated to match. |
| Passkeys (WebAuthn) | Fingerprint/face check per response | FR-2.5 is a Could. Defer it. |
| ORM (SQLAlchemy) | Automatic mapping | Plain SQL is easier to read for this size. |
| Password hashing, user accounts | Several lecturer accounts | One admin login from environment variables is enough for the demo. |
| Linter, coverage tool, CI pipeline | Automatic checks on GitHub | Run `pytest` locally before each merge (SOW §9 still holds). |
| Config library | — | Read settings with `os.environ`. |
| WebSocket, SPA frameworks, Redis/Celery, PostgreSQL, cloud hosting | — | Not needed at this size. |

---

## 4. Process

- Git + GitHub, one branch per work package, a second member reviews before merging (SOW §4, §9).
- Every merge requires `pytest` to pass locally.
- `python -m venv .venv` + `pip install -r requirements.txt`.
