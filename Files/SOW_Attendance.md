# Statement of Work (SOW)

**Project:** Automatic Class Attendance Web Application
**Version:** 1.1  |  **Date:** 2026-11-08
**Team:** _<names / IDs>_
**Course / Client:** _<course name, lecturer>_
**Requirements:** `SRS_Attendance.md` (v1.1). Requirement IDs below (FR-x, NFR-x) refer to it.

---

## 1. Background
Lecturers spend class time taking attendance, and a one-time check-in cannot tell whether a student stayed. This project delivers a web application that records attendance automatically through randomly timed spot checks answered from each student's registered device, for students in class and online, with no lecturer action during the lecture.

## 2. Objectives

| # | Objective | SRS coverage |
|---|-----------|--------------|
| O1 | Sessions open and close automatically from the course timetable | FR-1.x, FR-3.x, NFR-1 |
| O2 | Students confirm presence with one tap at random spot checks | FR-4.x, NFR-2, NFR-6 |
| O3 | Each student is bound to one device and one active login | FR-2.x, NFR-5 |
| O4 | Attendance status is computed by the N-of-M rule and can be overridden | FR-5.x |
| O5 | The lecturer sees live attendance and exports it as CSV | FR-6.x |

## 3. Scope

**In scope:** everything in SRS sections 3–6.

**Out of scope:** proof of physical location, comments, polls, lecture summaries, homework review, integration with Moodle/Zoom or other systems, native mobile apps, push notifications, PWA installation, dedicated hardware, and production hosting beyond the course demo.

## 4. Approach and Tools

Full details and reasons: `TechStack_Attendance.md`.

- **Backend:** Python 3.12+, FastAPI on Uvicorn (one process), SQLite via the standard-library `sqlite3`, SSE (`sse-starlette`) for live updates.
- **Frontend:** server-rendered Jinja2 templates + small vanilla JS.
- **Testing:** pytest + FastAPI test client; an asyncio + httpx load test script for NFR-3.
- **Demo HTTPS:** Cloudflare quick tunnel.
- **Process:** Git repository, one branch per work package, code review before merge, `pytest` must pass before every merge.

---

## 5. Work Breakdown

Each work package (WP) ends with a deliverable that can be demonstrated and tested on its own. "Acceptance" lists what must pass for the WP to be accepted.

### WP1: Project foundation and data model
- **Delivers:** repository, app skeleton, database schema for all SRS §6 entities, admin setup of courses, timetables, enrollments, CSV student import, and per-course M / N / window settings.
- **SRS:** FR-1.1, FR-1.2, FR-1.3
- **Produces for later WPs:** the data model (Student, Course, TimetableSlot, Enrollment, Session, SpotCheck, Response, AttendanceRecord).
- **Acceptance:**
  - A course created with a timetable and a 3-student CSV shows 3 enrollments.
  - A course created without M / N / window gets defaults M = 4, N = 3, window = 60 s.
  - A CSV with a malformed row is rejected with the row number reported; no partial import.

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
- **Delivers:** mobile-first student page; live alert for an open spot check over SSE while the page is open; "I'm here" response endpoint with validation and duplicate rejection.
- **SRS:** FR-2.2, FR-4.3–FR-4.6, NFR-2, NFR-4
- **Consumes:** WP2 device token, WP3 open spot checks.
- **Acceptance:**
  - An open spot check appears on a connected student page within 2 seconds.
  - A valid response within the window is recorded with the server timestamp; the student sees a confirmation.
  - Responses are rejected when the spot check is closed, the token is unknown, or the student is not enrolled in the course.
  - A second response to the same spot check is rejected and not recorded twice.
  - Answering takes one tap after registration, on current Chrome and Safari (Android, iOS, desktop).

### WP5: Attendance status and override
- **Delivers:** status computation at session close (Present / Partial / Absent / Flagged) and lecturer override with reason.
- **SRS:** FR-5.1, FR-5.2
- **Consumes:** WP3 session close event, WP4 responses.
- **Acceptance (M = 4, N = 3):**
  - 4/4 or 3/4 answered → Present; 2/4 or 1/4 → Partial; 0/4 → Absent.
  - Responses for one student from two device tokens → Flagged, with the count still shown.
  - An override changes the status and stores the lecturer's reason; the original computed status stays visible.
  - A student enrolled after a session closed gets no record for that session.

### WP6: Lecturer dashboard and export
- **Delivers:** course and session lists, live attendance table, per-student attendance percentage, CSV export per session and per course.
- **SRS:** FR-6.1–FR-6.4
- **Consumes:** WP5 attendance records.
- **Acceptance:**
  - The table updates within 2 seconds of a student response.
  - Exported CSV columns are exactly: student ID, name, checks answered / total, status, override note.
  - Names containing commas, quotes, or Hebrew characters export correctly and open in Excel.
  - Attendance percentage excludes cancelled sessions.

### WP7: Performance, security and privacy verification
- **Delivers:** load test, security and privacy checklist, final bug fixes.
- **SRS:** NFR-3, NFR-5, NFR-7
- **Consumes:** WP1–WP6.
- **Acceptance:**
  - 200 simulated students answer the same spot check; every response is acknowledged within 2 seconds and none are lost.
  - All traffic is HTTPS in the deployed demo.
  - The database stores no IP address, location, or sensor data.

---

## 6. Deliverables

| # | Deliverable | Due |
|---|-------------|-----|
| D1 | SRS (`SRS_Attendance.md`) and technology stack (`TechStack_Attendance.md`) | _<date>_ |
| D2 | This SOW | _<date>_ |
| D3 | Design documents: use cases, class and sequence diagrams | _<date>_ |
| D4 | Working application (WP1–WP6) with source code and automated tests | _<date>_ |
| D5 | Test report including load test results (WP7) | _<date>_ |
| D6 | Live demo and user guide (student and lecturer) | _<date>_ |

## 7. Milestones

| Milestone | Contents | Target |
|-----------|----------|--------|
| M1 | WP1 + WP2 accepted: courses and registered students exist | _<week>_ |
| M2 | WP3 + WP4 accepted: spot checks run end-to-end in a live session | _<week>_ |
| M3 | WP5 + WP6 accepted: statuses, dashboard, export | _<week>_ |
| M4 | WP7 accepted, final demo | _<week>_ |

WP2 and WP3 depend only on WP1 and can run in parallel. WP4 needs WP2 and WP3; WP5 needs WP4; WP6 needs WP5.

## 8. Roles

| Role | Responsibility | Member |
|------|----------------|--------|
| Project lead | Schedule, coordination, client contact | _<name>_ |
| Backend | WP1, WP3, WP5 | _<name>_ |
| Frontend | WP4 student page, WP6 dashboard | _<name>_ |
| Auth / security | WP2, WP7 | _<name>_ |
| QA | Acceptance tests for every WP | _<name>_ |

## 9. Acceptance Process
A work package is accepted when all its acceptance items pass as automated tests (or a recorded manual check where automation is impossible, e.g. Safari on iOS), and a second team member has reviewed the code. The project is accepted when all WPs are accepted and the live demo runs a full session: automatic open → spot checks → responses → statuses → CSV export.

## 10. Review Focus (risks not covered by any single requirement)

1. **Student's page is closed or the phone is locked during a spot check:** the check is simply missed; the page must say so clearly on next open. (WP4)
2. **Clock differences between server and devices:** all window decisions use the server clock only. (WP3, WP4)
3. **Server restart mid-session:** open sessions and pending spot checks survive. (WP3)
4. **Student changes or loses their phone before a lecture:** the replacement flow must not block attendance permanently. (WP2)
5. **Timetable edits after the semester starts:** past sessions and their records stay unchanged. (WP1, WP3)

## 11. Assumptions and Constraints
- As listed in SRS §2.3 and §2.4.
- The team has no budget; only free tools and a free or university-provided host are used.
- Students in the demo are test identities.

## 12. Sign-off

| Name | Role | Signature | Date |
|------|------|-----------|------|
| | Lecturer / client | | |
| | Project lead | | |
