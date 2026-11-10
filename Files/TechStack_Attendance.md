# Technology Stack

**Project:** Smart Classroom Web Application
**Version:** 2.1  |  **Date:** 2026-11-09
**Related:** `SRS_Attendance.md` (v2.1), `SOW_Attendance.md` (v2.1, §4 Approach and Tools)

Guiding rule: **keep it simple.** Use the Python standard library where it does the job, and add a package only when writing it ourselves would be harder. Versions are pinned in `requirements.txt` when the repository is created (WP1).

Status: v1.1 was approved by the team on 2026-11-08. The additions in v2.0 (marked **new**) cover features 2 to 5 and are not yet approved. The assignment requires each tool to be tested in real life; the "Test before use" list in section 5 is the plan for the tools report.

---

## 1. The stack

| Layer              | Choice                                                       | Package                                         |
| ------------------ | ------------------------------------------------------------ | ----------------------------------------------- |
| Language           | Python 3.12+                                                 | —                                               |
| Web server         | FastAPI on Uvicorn, one process                              | `fastapi`, `uvicorn`                            |
| Database           | SQLite through the standard library                          | — (`sqlite3`)                                   |
| Pages              | Server-rendered HTML templates + a little vanilla JavaScript | `jinja2`                                        |
| Live updates       | Server-Sent Events (SSE)                                     | `sse-starlette`                                 |
| File uploads **new** | Multipart form upload, files stored on local disk          | `python-multipart`                              |
| AI service client **new** | HTTPS calls to an outside AI service                  | `httpx` (now also a runtime package)            |
| Tests              | pytest + FastAPI's test client                               | `pytest`, `httpx`                               |
| HTTPS for the demo | Cloudflare quick tunnel                                      | `cloudflared` (a program, not a Python package) |
| Diagrams           | Mermaid inside Markdown                                      | —                                               |

**Location needs no package:** the browser's built-in Geolocation API reads the position, and the server computes the distance with `math` from the standard library. **new**

**Seven Python packages in total** (`fastapi`, `uvicorn`, `jinja2`, `sse-starlette`, `python-multipart`, `httpx`, `pytest`). Everything is free except possibly the AI service (section 2, AI service).

---

## 2. How each part is done

### Database
- Standard-library `sqlite3`, with plain SQL kept in a few repository classes (for example `StudentRepository` and `SessionRepository`).
- Each SRS §6 entity is a Python `@dataclass`. These classes are also the class diagram.
- Database rules that do real work:
  - `UNIQUE(spot_check_id, student_id)` blocks a second response to the same spot check (FR-4.5), even when two taps arrive at once.
  - `UNIQUE(device_token)` keeps every device token distinct.
  - **new:** `UNIQUE(poll_id, student_id)` blocks a second vote (FR-8.3).
- The schema is a single `schema.sql` file, run at startup.

### Scheduler (sessions, spot checks, AI jobs)
- One background loop inside the server checks once a second what is due:
  - open or close a session,
  - open or close a spot check,
  - compute statuses when a session closes,
  - **new:** close open polls when their session closes,
  - **new:** pick up AI jobs that are `pending` and run them.
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
- The server pushes "spot check open", "table changed", and **new:** "poll opened", "poll counts changed", "poll closed", "comment received", and "comment popped up" events through it.
- Students answer, vote, and comment with a normal `POST` request.
- A spot check's time is sent only at the moment it opens (NFR-6).

### Comments, polls and quizzes (new)
- All are plain tables plus `POST` endpoints and SSE events. No new package.
- A quiz is a poll whose options include one flagged correct. While the poll is open, the option list sent to students leaves out the flag (FR-8.7); it is sent only in the "poll closed" event.
- "Pop up" is a flag on a comment plus an SSE event; the student page shows a banner with the text only, no author (FR-7.6).
- "Offline" comments (FR-7.3) are comments sent while no session is open. They are stored and listed for the lecturer. Queuing on the device while it has no connection is not built.

### Location auto-answer (new)
- **Browser:** when the "spot check open" SSE event arrives and the student allowed location, the page script calls `navigator.geolocation.getCurrentPosition` and posts the result to the same response endpoint. If the call fails, is denied, or times out (about 10 seconds), the tap button is shown. Permission is asked once.
- **Server:** the great-circle (haversine) distance between the reported point and the lecture's classroom point is computed with `math` and compared with the radius the admin set for that lecture. The admin page has a "use my current location" button (the same Geolocation API) that fills the point; the admin edits it and chooses the radius. Inside the radius: the response is recorded with `method = location` and the distance. Outside: nothing is recorded and the tap button stays available. The same checks as a tap apply (token, enrollment, open window, `UNIQUE`).
- **Privacy:** the coordinates are used for the calculation only. Only the method and the distance are stored (SRS NFR-7).
- Requires HTTPS: browsers refuse location on plain `http`. The Cloudflare tunnel already provides it. During local development, `localhost` counts as secure.
- Accuracy is the weak point (SRS limitation 1), so the location test is done on real phones inside the real room (section 5).

### File uploads and recording download (new)
- Downloads of recordings and of a student's own files go through an endpoint that checks who is asking, then returns `FileResponse`. There is no public file link.
- Homework dataset documents have no download endpoint for students at all (FR-10.9).
- `python-multipart` lets FastAPI accept a form upload (MP4 or PDF).
- The server checks the file type and size (NFR-9), streams it to a folder `data/uploads/` outside the served folder, and stores the path in the database (NFR-10).
- Files are downloaded only through an endpoint that checks who is asking.
- No video or PDF processing library: the file goes to the AI service as it is. If the chosen service cannot take a video directly, the fallback is `ffmpeg` (a program) to extract audio; this is decided in the tools test.

### AI service client (new)
- Dataset files are sent with neutral identifiers (for example `doc-1`), never with a student's name or ID, so the review can quote an anonymous excerpt (FR-10.10).
- A small class `AiClient` with two methods: `summarize_video(path)` and `review_homework(pdf_path, dataset_paths)`. Calls use `httpx` with a timeout (NFR-8).
- A job is a row with status `pending`, `done`, or `failed`, plus result text or error message. The scheduler loop runs pending jobs, so an upload request returns at once and a slow service never blocks other features.
- Two implementations behind the same interface: the real client, and `FakeAiClient` that returns canned text. The fake is used in tests and as the demo fallback.
- Provider choice is open (SRS §8) and the tools test decides it. The demo needs only basic output: simple prompts, short samples, no tuning. Candidates to test: a general LLM API for the PDF review, and a speech-to-text or video-capable API for the summary. The key is read from an environment variable and never committed.
- Cost: the free tier of the chosen service is expected to be enough for the demo. If not, the fake client is used. **[assumed]**

### CSV
- Standard-library `csv`.
- **Import:** check every row first, then save them all in one transaction. A bad row rejects the whole file and reports the row number.
- **Export:** written as `utf-8-sig` so Excel shows Hebrew names correctly. Used for attendance (and optionally comments).

### HTTPS for the demo
- Run the server on a laptop, then run `cloudflared tunnel --url http://localhost:8000`.
- That prints a public `https://` address, which works on phones without certificate warnings.

### Load test (WP11)
- One Python script built on `asyncio` + `httpx` (already installed for tests).
- It simulates 200 students, sends 200 responses to one spot check, and reports the response times.

---

## 3. Dropped to keep it simple

| Dropped | What we lose | Effect on the SRS |
|---------|--------------|-------------------|
| Push notifications and PWA install (service worker, `pywebpush`) | Alerts when the student page is closed | FR-4.3 alerts over SSE only. |
| Passkeys (WebAuthn) | Fingerprint/face check per response | FR-2.5 is a Could. Defer it. |
| ORM (SQLAlchemy) | Automatic mapping | Plain SQL is easier to read for this size. |
| Password hashing, user accounts | Several lecturer accounts | One admin login from environment variables is enough for the demo. |
| Linter, coverage tool, CI pipeline | Automatic checks on GitHub | Run `pytest` locally before each merge (SOW §9 still holds). |
| Config library | — | Read settings with `os.environ`. |
| WebSocket, SPA frameworks, Redis/Celery, PostgreSQL, cloud hosting | — | Not needed at this size. |
| **new:** Job queue (Celery, RQ) | Distributed workers | The existing 1-second loop runs AI jobs from a database table. |
| **new:** Cloud file storage (S3), video/PDF libraries | Scalable storage, local processing | Local folder; files go to the AI service unchanged. |
| **new:** Own ML or vector search for the homework comparison | Our own similarity engine | The AI service does the comparison against the dataset. |
| **new:** Device-side offline queue for comments | Typing a comment with no connection | "Offline" means the lecturer need not be present. |
| **new:** Continuous or background location tracking | Noticing a student who leaves between checks | Location is read only at each spot check, with the page open. Leaving is noticed by the next check. |
| **new:** Built-in video player and streaming | Watching the recording inside the app | A download link only (FR-9.8). |

---

## 4. Process

- Git + GitHub, one branch per work package, a second member reviews before merging (SOW §4, §9).
- Every merge requires `pytest` to pass locally.
- `python -m venv .venv` + `pip install -r requirements.txt`.
- `.gitignore` includes `.venv`, `data/`, `.agents`, and any file with keys.

---

## 5. Test before use (input to the tools report)

Each tool is run once in a small real test before the team relies on it.

| Tool | What to test | Pass means |
|------|--------------|------------|
| Python 3.12+ | Install on the team machines | `python --version` works |
| FastAPI + Uvicorn | Hello-world endpoint and a Jinja2 page | Page loads on a phone on the same network |
| SQLite | Create the schema, insert, hit a `UNIQUE` rule | Duplicate insert is rejected |
| SSE | A page receives a pushed event from the server | Event arrives within 2 seconds on Chrome and Safari |
| File upload | Upload a 10 MB PDF and a short MP4 | Files appear in `data/uploads/` with the right size |
| Cloudflare tunnel | Open the public address on a phone | HTTPS page, no certificate warning |
| AI service (video) | Send a short MP4, read the summary | Readable summary within the timeout |
| AI service (PDF) | Send one PDF plus 3 dataset PDFs | Review mentions the comparison |
| Fake AI client | Run the full flow without network | Same screens, canned text |
| Geolocation (new) | Read position on an Android Chrome phone and an iPhone Safari, inside the real classroom, through the tunnel | Position arrives in under 10 seconds; the error against the real room point is measured and the radius is chosen from it |
| Recording download (new) | Download a stored MP4 as an enrolled student, then try as a non-enrolled user | First works, second is refused |
| Anonymous excerpt (new) | Send neutral-ID dataset PDFs and read the review | No name or student ID in the review text |
