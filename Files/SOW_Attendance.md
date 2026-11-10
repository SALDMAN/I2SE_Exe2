# Statement of Work (SOW)

**Project:** Smart Classroom Web Application (attendance, comments, polls, lecture summary, homework review)
**Version:** 2.1  |  **Date:** 2026-11-09
**Team:** _<names / IDs>_
**Course / Client:** _<course name, lecturer>_
**Requirements:** `SRS_Attendance.md` (v2.1). Requirement IDs below (FR-x, NFR-x) refer to it.

---

## 1. Background
Lecturers spend class time taking attendance, and a one-time check-in cannot tell whether a student stayed. They also lack quick ways to hear from students, poll the room, summarize a lecture, or review homework in depth. This project delivers a web application with five features:

1. Automatic attendance through randomly timed spot checks, with no lecturer action during the lecture (the most important feature). Students in the classroom who allow location have each check answered automatically by location match; online students, and students without a location match, answer by tap.
2. Student comments and suggestions; the lecturer can pop a comment up for the class.
3. Quick polls and quizzes created by the lecturer on the spot.
4. A lecture summary produced by an outside AI service from an uploaded MP4. Students read the summaries and download the recordings.
5. An optional, detailed homework review produced by an outside AI service, including an anonymous comparison with a dataset of other homework and anonymous examples of work done right.

## 2. Objectives

| # | Objective | SRS coverage |
|---|-----------|--------------|
| O1 | Sessions open and close automatically from the course timetable | FR-1.x, FR-3.x, NFR-1 |
| O2 | Students confirm presence at random spot checks: automatically by classroom location, or with one tap | FR-4.x, NFR-2, NFR-6 |
| O3 | Each student is bound to one device and one active login | FR-2.x, NFR-5 |
| O4 | Attendance status is computed by the N-of-M rule and can be overridden | FR-5.x |
| O5 | The lecturer sees live attendance and exports it as CSV | FR-6.x |
| O6 | Students send comments; the lecturer sees them live or afterwards and can pop one up for the class | FR-7.x |
| O7 | The lecturer runs quick polls and quizzes with live results | FR-8.x |
| O8 | The lecturer uploads a lecture video; students read the AI summary and download the recording | FR-9.x, NFR-8, NFR-9 |
| O9 | A student can optionally get an AI review of a homework PDF, with an anonymous comparison | FR-10.x, NFR-11 |

## 3. Scope

**In scope:** everything in SRS sections 3–6.

**Out of scope:** proof of physical location, integration with Moodle, Zoom, Google Meet or Classroom, building or training our own AI models, native mobile apps, push notifications, PWA installation, dedicated hardware, and production hosting beyond the course demo.

**Priority:** attendance (WP1–WP6) is the core and must be complete. Comments and polls (WP7–WP8) are small and are built fully. The two AI features (WP9–WP10) are built on top of one shared AI client; their quality depends on the outside service.

## 4. Approach and Tools

Full details and reasons: `TechStack_Attendance.md`.

- **Backend:** Python 3.12+, FastAPI on Uvicorn (one process), SQLite via the standard-library `sqlite3`, SSE (`sse-starlette`) for live updates.
- **Frontend:** server-rendered Jinja2 templates + small vanilla JS.
- **Location:** the browser's built-in Geolocation API plus a distance check in the standard library; no extra package.
- **Uploads:** `python-multipart`, files on local disk.
- **AI features:** an `AiClient` class that calls an outside AI service through `httpx`; a fake client for tests and demo fallback. Jobs run in the background from a database table.
- **Testing:** pytest + FastAPI test client; an asyncio + httpx load test script for NFR-3.
- **Demo HTTPS:** Cloudflare quick tunnel.
- **Process:** Git repository, one branch per work package, code review before merge, `pytest` must pass before every merge.

---

## 5. Work Breakdown

Each work package (WP) ends with a deliverable that can be demonstrated and tested on its own. "Acceptance" lists what must pass for the WP to be accepted.

### WP1: Project foundation and data model
- **Delivers:** repository with README (problem and scope), app skeleton, database schema for the SRS §6 entities of attendance, admin setup of courses, timetables, enrollments, CSV student import, and per-course M / N / window settings, and the classroom location (point and radius, chosen by the admin, with a "use my current location" button) per lecture.
- **SRS:** FR-1.1–FR-1.4
- **Produces for later WPs:** the data model; later WPs add their own tables to `schema.sql`.
- **Acceptance:**
  - A course created with a timetable and a 3-student CSV shows 3 enrollments.
  - A course created without M / N / window gets defaults M = 4, N = 3, window = 60 s.
  - A CSV with a malformed row is rejected with the row number reported; no partial import.
  - A lecture without a classroom point or a radius, or with a latitude, longitude, or radius out of range, is rejected. There is no default radius.

### WP2: Student registration and device binding
- **Delivers:** one-time registration by student ID, device token issue and storage, one active login per student, lecturer approval for device replacement.
- **SRS:** FR-2.1–FR-2.4, NFR-5
- **Consumes:** WP1 data model.
- **Acceptance:**
  - Registration returns a token of at least 128 random bits; two registrations never return the same token.
  - Registering an ID that is not on any course list is rejected.
  - Logging in on a second browser ends the first login.
  - A second device for the same student is refused until the lecturer approves it.

### WP3: Automatic sessions and spot-check scheduling
- **Delivers:** scheduler that opens and closes sessions from the timetable, picks M hidden random times per session, opens and closes each spot check for the response window; lecturer cancel of a single session.
- **SRS:** FR-3.1, FR-3.2, FR-4.1, FR-4.2, NFR-1, NFR-6
- **Consumes:** WP1 data model.
- **Acceptance:**
  - With a frozen clock, a session opens at the timetabled start and closes at the end with no manual action.
  - Exactly M spot checks are created, all fully inside the session (each closes before the session ends).
  - No API response sent to a student contains a spot check's time before it opens.
  - A cancelled session never opens and creates no spot checks.
  - After a server restart mid-session, the open session and remaining spot checks still run.

### WP4: Student page and spot-check responses
- **Delivers:** mobile-first student page; live alert for an open spot check over SSE while the page is open; "I'm here" response endpoint with validation and duplicate rejection; automatic answer by location with fallback to the tap button.
- **SRS:** FR-2.2, FR-4.3–FR-4.10, NFR-2, NFR-4, NFR-7
- **Consumes:** WP2 device token, WP3 open spot checks.
- **Acceptance:**
  - An open spot check appears on a connected student page within 2 seconds.
  - A valid response within the window is recorded with the server timestamp; the student sees a confirmation.
  - Responses are rejected when the spot check is closed, the token is unknown, or the student is not enrolled in the course.
  - A second response to the same spot check is rejected and not recorded twice.
  - Answering takes one tap after registration, on current Chrome and Safari (Android, iOS, desktop).
  - A student whose reported location is inside the radius gets the spot check answered with no tap; the response is recorded as method "location" with its distance.
  - A reported location outside the radius is not recorded as an answer, and the tap button stays available for the rest of the window.
  - A student who denies location answers by tap with no penalty; the permission prompt appears once, with an explanation.
  - The database stores the method and distance of a location answer, never the raw coordinates.
  - A location answer fails the same validity checks as a tap (closed check, unknown token, not enrolled, duplicate).

### WP5: Attendance status and override
- **Delivers:** status computation at session close (Present / Partial / Absent / Flagged) and lecturer override with reason.
- **SRS:** FR-5.1, FR-5.2
- **Consumes:** WP3 session close event, WP4 responses.
- **Acceptance (M = 4, N = 3):**
  - 4/4 or 3/4 answered → Present; 2/4 or 1/4 → Partial; 0/4 → Absent.
  - Responses for one student from two device tokens → Flagged, with the count still shown.
  - An override changes the status and stores the lecturer's reason; the original computed status stays visible.
  - A student enrolled after a session closed gets no record for that session.

### WP6: Lecturer dashboard and attendance export
- **Delivers:** course and session lists, live attendance table, per-student attendance percentage, CSV export per session and per course. This page is the base for the lecturer parts of WP7–WP10.
- **SRS:** FR-6.1–FR-6.4
- **Consumes:** WP5 attendance records.
- **Acceptance:**
  - The table updates within 2 seconds of a student response.
  - The table shows per student how each check was answered (location or tap).
  - Exported CSV columns are exactly: student ID, name, checks answered / total, status, override note.
  - Names containing commas, quotes, or Hebrew characters export correctly and open in Excel.
  - Attendance percentage excludes cancelled sessions.

### WP7: Comments and suggestions
- **Delivers:** comment box on the student page, comment endpoint, live comment list on the lecturer dashboard, stored list for comments sent outside a session, pop-up of one chosen comment on all student pages.
- **SRS:** FR-7.1–FR-7.4, FR-7.6, FR-7.7 (FR-7.5 optional)
- **Consumes:** WP2 identity, WP4 student page, WP6 dashboard.
- **Acceptance:**
  - A comment sent during an open session appears on the dashboard within 2 seconds, linked to that session.
  - A comment sent when no session is open is stored and appears in the course comment list.
  - A comment over 500 characters, an empty comment, or one from a student not enrolled in the course is rejected.
  - When the lecturer pops a comment up, it appears on connected student pages within 2 seconds without the author's name, and disappears on dismiss or when the session closes.

### WP8: Quick polls and quizzes
- **Delivers:** one-step poll creation by the lecturer with an optional correct option (quiz), poll shown on student pages over SSE, one vote per student, live counts for the lecturer, close by lecturer or by session end, result and right/wrong feedback to students.
- **SRS:** FR-8.1–FR-8.8
- **Consumes:** WP3 sessions, WP4 student page, WP6 dashboard.
- **Acceptance:**
  - A poll with a question and 2–6 options opens at once and appears on connected student pages within 2 seconds.
  - A second vote by the same student is rejected and not counted; two simultaneous votes count once.
  - The lecturer's counts update live and show no voter names.
  - After the poll closes, or its session closes, new votes are rejected.
  - A poll with fewer than 2 or more than 6 options is rejected.
  - In a quiz, no response sent to a student before the quiz closes contains the correct option.
  - After a quiz closes, each student sees the correct answer and whether their own answer was right; nothing is saved as a grade.

### WP9: AI client and lecture summary
- **Delivers:** the shared `AiClient` (real and fake), the background job runner, MP4 upload by the lecturer, job state display, summary display and retry, student list of previous lectures with summaries, and the MP4 download link.
- **SRS:** FR-9.1–FR-9.5, FR-9.7, FR-9.8 (FR-9.6 optional), NFR-8, NFR-9, NFR-10
- **Consumes:** WP3 sessions, WP6 dashboard.
- **Acceptance:**
  - With the fake client, an uploaded MP4 ends with a stored summary, shown to the lecturer and the course's students.
  - The upload request returns immediately; the job state goes pending → done.
  - When the client raises an error or times out, the job shows failed with a clear message, and retry works.
  - While a job is running or failed, attendance, comments, and polls keep working.
  - A non-MP4 file or a file over the size limit is rejected with a clear message.
  - An enrolled student sees the list of previous lectures with summaries and can download the MP4 from the summary page; a student of another course, or a visitor with the direct file path, gets refused.
  - A pending job survives a server restart and still runs.
  - The same flow works once against the real AI service on a short sample MP4 (manual check, recorded).

### WP10: Homework review
- **Delivers:** homework assignments and an admin dataset upload, optional student PDF upload with opt-in to the dataset, review via the shared AI client, review display, retry.
- **SRS:** FR-10.1–FR-10.10, NFR-8, NFR-10, NFR-11
- **Consumes:** WP1 courses, WP2 identity, WP9 `AiClient` and job runner.
- **Acceptance:**
  - A student who never uploads anything is unaffected and not blocked.
  - An uploaded PDF produces a job; with the fake client the review text appears for that student only.
  - The review request contains the student's PDF and the assignment's dataset.
  - A PDF is added to the dataset only when the student ticked the opt-in, and only after its own review finished.
  - A student cannot open another student's PDF or review.
  - The lecturer sees the list of students who uploaded, and each student's review, the same text the student sees.
  - No student page or endpoint lists, opens, or downloads a dataset document.
  - The files sent to the AI service carry neutral identifiers; a review shown to a student contains no other student's name or ID.
  - Examples of work done right in a review come only from sample solutions or opted-in submissions.
  - A failed job shows a clear message, and retry works. A non-PDF or a file over the size limit is rejected.
  - The same flow works once against the real AI service on 1 PDF plus a small dataset (manual check, recorded).

### WP11: Performance, security, privacy verification and final documents
- **Delivers:** load test, security and privacy checklist, final bug fixes, user guide, testing report, conclusion.
- **SRS:** NFR-3, NFR-5, NFR-7, NFR-8, NFR-10, NFR-11
- **Consumes:** WP1–WP10.
- **Acceptance:**
  - 200 simulated students answer the same spot check; every response is acknowledged within 2 seconds and none are lost.
  - All traffic is HTTPS in the deployed demo.
  - The database stores no IP address, raw location coordinates, or sensor data.
  - Location is requested only with the student's permission.
  - No AI key is in the repository; uploaded files are not reachable by direct URL.

---

## 6. Deliverables

These map to the course assignment list.

| # | Deliverable | Course item | Due |
|---|-------------|-------------|-----|
| D1 | Git repository with README: problem and scope (WP1) | 1 | _<date>_ |
| D2 | Technical tools report: each tool tested in real life: `ToolsReport_Attendance.md` (in progress; plan in `TechStack_Attendance.md` §5) | 2 | _<date>_ |
| D3 | This SOW | 3 | _<date>_ |
| D4 | SRS (`SRS_Attendance.md`) | 4 | _<date>_ |
| D5 | Main use cases (client stories): `UseCases_Attendance.md`, 13 use cases agreed 2026-11-09 (draft written) | 5 | _<date>_ |
| D6 | Main software components: `Components_Attendance.md`, 17 components in two levels (draft written) | 6 | _<date>_ |
| D7 | Low-level class and sequence diagrams: `Design_Attendance.md`, 2 class diagrams and 7 sequence diagrams (draft written) | 7 | _<date>_ |
| D8 | Testing strategy covering the classes of D7: `TestingStrategy_Attendance.md` (draft written) | 8 | _<date>_ |
| D9 | Demo web application (KISS), WP1–WP10, with source code and automated tests | 9 | _<date>_ |
| D10 | Test report including load test results (WP11) | 8 | _<date>_ |
| D11 | Live demo in class, and user guide | 10 | _<date>_ |
| D12 | Conclusion with a short description of the results | 11 | _<date>_ |

## 7. Milestones

| Milestone | Contents | Target |
|-----------|----------|--------|
| M1 | WP1 + WP2 accepted: courses and registered students exist; D1–D4 written | _<week>_ |
| M2 | WP3 + WP4 accepted: spot checks run end-to-end in a live session; D5–D7 written | _<week>_ |
| M3 | WP5 + WP6 accepted: statuses, dashboard, export; D8 written | _<week>_ |
| M4 | WP7 + WP8 accepted: comments and polls | _<week>_ |
| M5 | WP9 + WP10 accepted: AI features | _<week>_ |
| M6 | WP11 accepted, D9–D12 done, final demo | _<week>_ |

Dependencies: WP2 and WP3 depend only on WP1 and can run in parallel. WP4 needs WP2 and WP3; WP5 needs WP4; WP6 needs WP5. WP7 and WP8 need WP4 and WP6. WP9 needs WP6; WP10 needs WP9 for the shared AI client. The AI work (testing the outside service, `AiClient`) can start early, in parallel with WP1–WP6, because it does not depend on attendance.

## 8. Roles

| Role | Responsibility | Member |
|------|----------------|--------|
| Project lead | Schedule, coordination, client contact, README, conclusion | _<name>_ |
| Backend | WP1, WP3, WP5, WP8 | _<name>_ |
| Frontend | WP4 student page, WP6 dashboard, WP7 | _<name>_ |
| Auth / security | WP2, WP11 | _<name>_ |
| AI integration | WP9, WP10, AI service test (tools report) | _<name>_ |
| QA | Acceptance tests for every WP, testing strategy (D8) | _<name>_ |

If the team is smaller, roles are combined; the work packages stay the same.

## 9. Acceptance Process
A work package is accepted when all its acceptance items pass as automated tests (or a recorded manual check where automation is impossible, e.g. Safari on iOS, or a call to the real AI service), and a second team member has reviewed the code. The project is accepted when all WPs are accepted and the live demo runs a full session: automatic open → spot checks → responses → statuses → CSV export, plus a comment, a poll, a lecture summary, and a homework review.

## 10. Review Focus (risks not covered by any single requirement)

1. **Student's page is closed or the phone is locked during a spot check or poll:** it is simply missed; the page must say so clearly on next open. (WP4, WP8)
2. **Clock differences between server and devices:** all window decisions use the server clock only. (WP3, WP4)
3. **Server restart mid-session:** open sessions, pending spot checks, and pending AI jobs survive. (WP3, WP9)
4. **Student changes or loses their phone before a lecture:** the replacement flow must not block attendance permanently. (WP2)
5. **Timetable edits after the semester starts:** past sessions and their records stay unchanged. (WP1, WP3)
6. **Outside AI service is slow, wrong, unavailable, or out of free quota:** the failure stays inside the job; the fake client keeps the demo running. This is the largest schedule risk, so the AI service is tested first. (WP9, WP10)
7. **Student privacy for homework:** opt-in only, own review only, no AI key in the repository. (WP10, WP11)
8. **Large video upload on demo Wi-Fi:** the demo uses a short MP4 and the size limit is enforced. (WP9)
9. **Location accuracy indoors:** a phone may report a position outside the radius while inside the room. The fallback is the tap button, so nobody loses attendance, and the radius is tuned in the room on demo day. The location test runs early on real phones. (WP1, WP4)
10. **Location can be faked, and some browsers ask permission often:** an answer by location is evidence only; permission behavior on Safari iOS is checked on a real phone. (WP4, WP11)
11. **Recording files are large and private:** download goes through an access check, never a public file link. (WP9)

## 11. Assumptions and Constraints
- As listed in SRS §2.3 and §2.4.
- The team has no budget; only free tools and a free or university-provided host are used. The AI service must be usable on a free tier, or the fake client is used. **[assumed]**
- Students in the demo are test identities; the homework dataset in the demo is prepared sample PDFs.
- The AI features in the demo are basic: simple prompts on short samples, no tuning. The fake client is the fallback.

## 12. Open Items
The open questions listed in SRS §8 also apply to this SOW: the AI provider (chosen by the tools test), team size and dates. Decided: the demo AI is basic, and the admin sets the classroom point and radius for each lecture; the lecturer sees the same individual homework reviews as students, with no class-wide comparison; students see summaries and download recordings; students who deny location answer by tap; the dataset is hidden from students.

## 13. Sign-off

| Name | Role | Signature | Date |
|------|------|-----------|------|
| | Lecturer / client | | |
| | Project lead | | |
