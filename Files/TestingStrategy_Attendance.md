# Testing Strategy

**Project:** Smart Classroom Web Application
**Version:** 1.0  |  **Date:** 2026-11-09
**Related:** `Design_Attendance.md` (classes and sequences), `SRS_Attendance.md` (v2.1), `SOW_Attendance.md` (v2.1, §5 acceptance and §9 process), `TechStack_Attendance.md`

Guiding rule, as for the rest of the project: **keep it simple.** Test the rules that can go wrong, not every line. Every class in `Design_Attendance.md` has a row in section 4.

---

## 1. Goals
1. Each SOW acceptance item passes as an automated test where possible.
2. The rules that protect attendance (window, duplicates, location, status) are tested with boundary values.
3. Secrets never leave early: spot-check times, a quiz's correct option, dataset files.
4. A slow or failing AI service breaks nothing but its own job.
5. The demo works on real phones, in the real room, over HTTPS.

## 2. Test levels

| Level | What it tests | Tools | Speed |
|-------|---------------|-------|-------|
| L1 Unit | One class or function with no database: `LocationChecker`, status rule, window check, CSV parsing | pytest | milliseconds |
| L2 Service | A service with a real temporary SQLite database, fake clock, fake AI client, temporary upload folder | pytest | tens of ms |
| L3 API | Routes, login, cookies, status codes, SSE events, through FastAPI's test client | pytest + `httpx` / `TestClient` | tens of ms |
| L4 Load | 200 simulated students answering one spot check | `asyncio` + `httpx` script | seconds |
| L5 Manual | Real phones, real browsers, real room, real AI service | checklist, recorded in the test report | minutes |

Most tests sit in L1 and L2. L3 checks wiring and security. L4 and L5 are few and aimed at the non-functional requirements.

## 3. Tools and test doubles

| Item | Use |
|------|-----|
| `pytest` | runner; fixtures; parametrized boundary cases |
| `httpx` + FastAPI `TestClient` | L3 and L4 |
| **Temporary SQLite database** | a fresh file per test from `schema.sql`, so the real SQL and the `UNIQUE` rules are tested. No mocked database |
| **`FakeClock`** | freezes and advances time; every window, session start, and timeout test uses it |
| **`FakeAiClient`** | returns canned text, or raises an error or a timeout on demand; used in all automated tests |
| **Temporary upload folder** | per test; deleted afterwards |
| **Fixed random seed hook** | `AttendanceService` takes its random source as a parameter so tests can check "M times, all inside the session" without relying on luck |

Real services are never called in automated tests. The real AI service and the real Geolocation API are tested only in L5.

Folder layout:

```
tests/
  unit/      L1
  service/   L2
  api/       L3
  load/      L4 script
  manual/    L5 checklist
  conftest.py   fixtures: db, clock, ai, uploads, client, sample course
```

## 4. Test plan per class

Class names are those of `Design_Attendance.md`. "Key cases" are the ones that must exist; more can be added.

### 4.1 Entities and repositories (C16, C17)
| Class | Level | Key cases |
|-------|-------|-----------|
| Entities (`@dataclass`) | L1 | Construction with defaults; no behavior to test beyond this |
| Repositories | L2 | Insert and get round-trip for each entity; `UNIQUE(spot_check_id, student_id)` rejects a second Response; `UNIQUE(poll_id, student_id)` rejects a second vote; `UNIQUE(device_token)` rejects a repeat; two threads inserting the same Response at once leave exactly one row; schema runs twice without error |

### 4.2 Time and scheduling (C11, C12)
| Class | Level | Key cases |
|-------|-------|-----------|
| `Clock` / `FakeClock` | L1 | `FakeClock.advance` moves `now()`; real `Clock` returns increasing times |
| `Scheduler` | L2 | With a frozen clock: before start nothing opens; at start the session opens; at end it closes, with no manual action. Exactly M spot checks exist, each inside the session and each closing before the session ends. Cancelled session never opens and has no checks. After a simulated restart (new `Scheduler` on the same database) the open session and remaining checks still run. Calling `tick()` twice at the same time creates nothing twice |

### 4.3 Attendance (C4, C5)
| Class | Level | Key cases |
|-------|-------|-----------|
| `LocationChecker` | L1 | Same point gives 0 m; one degree of latitude gives about 111 km; a point just inside and just outside the radius (boundary values); longitude wrap near ±180; invalid latitude or longitude raises an error |
| `AttendanceService` responses | L2 | Valid tap recorded with `method=tap`; valid location answer recorded with `method=location` and distance; location outside radius records nothing and returns `"outside"`; window not yet open, exactly at `closes_at`, and after close are rejected (boundary); unknown token rejected; student not enrolled rejected; second answer returns `"already_answered"` and leaves one row; raw coordinates never appear in the database (search every column) |
| `AttendanceService` secrecy | L2 + L3 | No payload sent to a student contains a spot check's time before it opens; payload at open contains only id and `closes_at` |
| `AttendanceService` status | L1 on the rule, L2 with the database | With M = 4, N = 3: 4/4 and 3/4 give Present; 2/4 and 1/4 give Partial; 0/4 gives Absent. Same student from two tokens gives Flagged and keeps the count. One token answering for two students gives Flagged. Override stores reason and keeps the computed status. Student enrolled after the session closed has no record. Cancelled session has no records. Non-default M and N from a course are used |

### 4.4 Course, CSV, and device binding (C3, C15)
| Class | Level | Key cases |
|-------|-------|-----------|
| `CourseService` | L2 | Course without M / N / window gets 4 / 3 / 60. Lecture without a point or radius, or with latitude, longitude, or radius out of range, is rejected. Registration returns a token of at least 128 bits (decoded length) and two registrations never match. ID not on any list is rejected. Second device refused until approved; approval swaps the token. A second login ends the first. Timetable edit leaves past sessions untouched |
| `CsvService` | L1 / L2 | 3-row file gives 3 enrollments. Malformed row rejects the whole file and reports the row number, with nothing saved. Names with commas, quotes, and Hebrew letters round-trip. Export is `utf-8-sig`, columns in the exact order of FR-6.4. Percentage excludes cancelled sessions |

### 4.5 Polls and comments (C6, C7, C2)
| Class | Level | Key cases |
|-------|-------|-----------|
| `PollService` | L2 | 1 option and 7 options rejected; 2 and 6 accepted. One vote per student; two simultaneous votes count once. Vote after close rejected. Not-enrolled student rejected. `close_ended_polls` closes polls of a closed session. Counts contain no voter names. A quiz never exposes `is_correct` before close, and exposes it after. A poll without a correct option sends no right/wrong |
| `CommentService` | L2 | Empty comment and 501 characters rejected; 500 accepted. Linked to the open session when one exists, stored with no session otherwise. Not-enrolled student rejected. Pop-up marks one comment; the pushed payload has the text and no author; dismiss and session close clear it |
| `EventHub` | L2 + L3 | A subscriber receives a published event; an event for another audience is not received; a closed connection is dropped without error; an event reaches a connected client within 2 seconds |

### 4.6 Files and AI (C8, C9, C10, C13, C14)
| Class | Level | Key cases |
|-------|-------|-----------|
| `FileService` | L2 | MP4 over the limit and a non-MP4 are rejected; same for PDF. Saved path is outside the served folder. Owner, enrolled student, and lecturer may open a file; another student, a student of another course, and an anonymous caller may not. Dataset files cannot be opened by any student |
| `AiClient` contract | L2 | `FakeAiClient` returns canned text, can raise an error, can simulate a timeout. `RealAiClient` is not run here (L5) but is checked against the same interface |
| `AiJobRunner` | L2 | Pending job becomes done with the fake text. Error and timeout both leave the job failed with a clear message; retry makes it pending again and then done. A failing job does not stop other jobs or a `Scheduler.tick()`. A pending job survives a restart and still runs. Timeout values come from settings |
| `SummaryService` | L2 | Upload returns at once with state pending. Done summary is listed for enrolled students and not for others. List is ordered by lecture date |
| `HomeworkService` | L2 | Student who never uploads is unaffected. Upload creates a job. The request to the AI contains the PDF and the dataset. Dataset files carry neutral ids and no name or student ID. Opt-in off: the PDF never joins the dataset. Opt-in on: it joins only after the review is done. A student sees only their own review. The lecturer sees every uploader's review, the same text |
| Dataset access (routes) | L3 | No student route lists, opens, or downloads a dataset document |

### 4.7 Routes and login (C1)
| Class | Level | Key cases |
|-------|-------|-----------|
| Routes and login | L3 | Student routes with no or wrong cookie return 401. Lecturer and admin routes without HTTP Basic return 401; with wrong credentials 401; with right ones 200. A student calling a lecturer route gets 403. Cookie is `HttpOnly` and `Secure`. Old login id is rejected after a new login. Each route returns the right status for valid, invalid, and duplicate input. SSE route streams events and rejects anonymous callers |

## 5. Traceability to the SRS and SOW

| WP | Main automated tests | Level |
|----|----------------------|-------|
| WP1 | Defaults, CSV import, location validation | L2 |
| WP2 | Registration, token length, second device, single login | L2, L3 |
| WP3 | Frozen-clock session, exactly M checks, hidden times, cancel, restart | L2 |
| WP4 | Tap and location answers, rejection cases, duplicate, 2-second push | L2, L3 |
| WP5 | Status rule, Flagged, override | L1, L2 |
| WP6 | Live table, CSV columns and encoding, percentage | L2, L3 |
| WP7 | Comment limits, linking, pop-up | L2, L3 |
| WP8 | Poll limits, one vote, quiz secrecy, close | L2 |
| WP9 | Upload, job states, failure and retry, restart, access to the recording | L2, L3 |
| WP10 | Dataset hiding, neutral ids, opt-in, own review only | L2, L3 |
| WP11 | Load test, HTTPS, no coordinates stored, no key in the repository | L4, L5 |

Each SRS requirement ID appears in at least one test name or docstring (for example `test_fr_4_5_second_response_rejected`), so a search for an ID finds its test.

## 6. Non-functional tests

| Requirement | Test |
|-------------|------|
| NFR-3 (200 students, 2 seconds) | L4: the script registers 200 test students, opens one spot check, sends 200 answers at once, and records each response time. Pass: every response acknowledged within 2 seconds and none lost. Run once on the laptop before the demo, results go in the report |
| NFR-4 (browsers) | L5: current Chrome and Safari on Android, iOS, and desktop: register, receive a check, answer by tap, answer by location |
| NFR-5 (token, HTTPS) | L2 token length and uniqueness; L5 the demo address is `https://` |
| NFR-6 (no early secrets) | L2 and L3 payload checks (section 4.3, 4.5) |
| NFR-7 (no coordinates, no IP) | L2: after location answers, scan every database column; L5: read the schema |
| NFR-8 (AI cannot break the rest) | L2: a failing and a very slow fake AI client while polls, comments, and attendance tests run |
| NFR-9, NFR-10 (limits, access) | L2 `FileService` cases |
| NFR-11 (homework privacy) | L2 and L3 cases in 4.6 |

## 7. Manual tests (L5)

Recorded in the test report with date, device, browser, and result.

1. **Location in the real room.** Open the student page on an Android Chrome phone and an iPhone Safari, through the tunnel, inside the classroom. Note the reported error against the lecture point and set the radius from it. Check the permission prompt appears once and an explanation is shown.
2. **Location denied.** Deny permission: the tap button works on every check.
3. **Cross-device run.** One full short session with at least three real devices: automatic open, checks answered by location and by tap, dashboard live, CSV opens in Excel with Hebrew names.
4. **Real AI service.** One short MP4 and one PDF with a small dataset against the real provider: result is readable; a revoked key gives a failed job with a clear message.
5. **Recording download.** As an enrolled student on a phone, then as a non-enrolled user.
6. **Tunnel.** The public address works on a phone with no certificate warning.

## 8. Process
- Every work package ends with its automated acceptance tests passing.
- `pytest` must pass locally before every merge (SOW §9). A second team member reviews the code and the tests.
- A bug found by hand gets a failing test first, then the fix.
- The suite is split in fast tests (L1 to L3, run every time) and the load script (run before the demo).
- Test data is made up: test students and prepared sample PDFs. No real student data.

## 9. What is not tested, on purpose
- The quality of AI output. Only that a job completes, fails clearly, and stays isolated.
- Location accuracy beyond the one real-room measurement.
- Other browsers than current Chrome and Safari.
- Anything the tech stack dropped: push notifications, passkeys, offline queuing.
- Line coverage as a number: no coverage tool (tech stack §3). The class table in section 4 is the coverage check.

## 10. Exit criteria for the demo
1. All L1 to L3 tests pass.
2. The load test meets NFR-3.
3. Every L5 item has a recorded result, and failures are either fixed or listed in the report as known limits.
4. The fake AI client is ready as a fallback, and the demo can switch to it with one setting.
