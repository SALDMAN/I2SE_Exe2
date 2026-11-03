# Software Requirements Specification (SRS)

**Project:** Web-Based Teaching Tool
**Version:** 0.1 (draft)  |  **Date:** _<date>_
**Team:** _<names / IDs>_

Priority legend: **M** = Must, **S** = Should, **C** = Could (stretch).

---

## 1. Introduction

### 1.1 Purpose
This document specifies the requirements of a web-based teaching tool used in class by a lecturer and students. It is the basis for the use cases, component design, class and sequence diagrams, and the test strategy.

### 1.2 Scope
The system provides: (1) automatic attendance with spreadsheet output, (2) real-time and offline student comments, (3) on-the-spot polls, (4) ML/LLM summaries of recorded lectures, and (5) LLM review of homework with peer comparison. It does not integrate with Moodle, Zoom, Google Classroom, or any system holding personal data.

### 1.3 Definitions

| Term | Meaning |
|------|---------|
| Session | One class meeting started by the lecturer |
| Challenge | A short, timed request for the student's device to prove presence during a session |
| Room proof | Information obtainable only inside the room, such as the rotating code shown on the projector |
| Device token | A secret stored on the student's phone that binds the device to one student |
| Verification status | Verified / Partial / Unverified / Flagged |
| N of M | A student must pass at least N of M challenges to be Verified |
| KISS | Keep It Simple, Stupid |

### 1.4 References
README.md, `docs/02_SoW.md`, `docs/01_Tools_Report.md`.

---

## 2. Overall Description

### 2.1 Product perspective
A self-contained web application: a mobile-first student interface (installable PWA), a lecturer dashboard, and a server with a database and LLM/ML services. It runs on a laptop in the classroom (or on a hosted server) and is reached by phone browsers.

### 2.2 User classes

| User | Description |
|------|-------------|
| Student | Uses a smartphone browser; wants minimal effort |
| Lecturer | Uses a laptop dashboard and projector; wants reliable data and a spreadsheet |
| Administrator (demo only) | Configures rooms and registers demo students; may be the lecturer |

### 2.3 Operating environment
Modern mobile browsers (Chrome, Safari), a Python server (e.g. FastAPI), SQLite, a classroom Wi-Fi network, and optionally a portable router.

### 2.4 Constraints
- Web-based only, no native apps.
- Demo must run live and fit in 10 minutes.
- Browsers cannot read the Wi-Fi name, BSSID, or signal strength, and cannot run in the background when the screen is locked.
- The server sees only the client's IP address, not the router's identity directly.

### 2.5 Assumptions and dependencies
- Students have a smartphone and a data or Wi-Fi connection.
- Each classroom either has its own subnet or the team supplies a portable router (to be confirmed by the network spike).
- LLM and speech-to-text services are available.

---

## 3. Functional Requirements

### 3.1 Registration and identity

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1.1 | The system shall let a student register once with a name or ID, producing a device token stored on the phone. | M |
| FR-1.2 | The system shall bind a student to one device token and reject a second device for the same student unless the lecturer approves it. | S |
| FR-1.3 | The system shall flag a device token that is used for more than one student identity. | S |
| FR-1.4 | The lecturer shall be able to import a student list (CSV) and see which students are registered. | S |

### 3.2 Session management

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-2.1 | The lecturer shall be able to start and end a session with one action. | M |
| FR-2.2 | A session shall be linked to a room, and a room shall be linked to a network range (subnet) when available. | S |
| FR-2.3 | The lecturer shall configure the check-in window, the number of challenges (M), and the passing threshold (N). Defaults: window 10 minutes, M = 4, N = 3. | S |

### 3.3 Automatic attendance (main feature)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-3.1 | The dashboard shall display a room code that changes every 10 to 20 seconds (as text and as a QR code). | M |
| FR-3.2 | A student shall be able to check in by opening the app, with the stored device token sent automatically, and entering or scanning the current room code. | M |
| FR-3.3 | The server shall accept a check-in only if the device token is known, the session is open, and the code is the current (or immediately previous) code. | M |
| FR-3.4 | The server shall reject duplicate check-ins for the same challenge. | M |
| FR-3.5 | The server shall record for each check-in: student, device token, timestamp, client IP, and which signals matched. | M |
| FR-3.6 | During a session the server shall issue random presence challenges (M per session), each open for a limited time (default 30 seconds), each requiring a fresh room proof. | S |
| FR-3.7 | The system shall notify students of a challenge through the open page (WebSocket) and, when available, a push notification. | S |
| FR-3.8 | When a room has a configured network range, the server shall record whether the client IP is inside it as an advisory signal. The network signal shall never produce a Verified status on its own. | S |
| FR-3.9 | The system may use an audio token (tone decoded by the phone's microphone) as an additional room proof. | C |
| FR-3.10 | The system may use browser geolocation as a weak, advisory signal. | C |
| FR-3.11 | The server shall compute a verification status per student (see 3.3.1). | M |
| FR-3.12 | The lecturer shall be able to manually override any student's status, with the override recorded. | M |
| FR-3.13 | Check-ins made while the phone is offline shall be queued locally with their original timestamp and sent when the connection returns; the server shall accept them only within the grace period (default 5 minutes after the window closes). | S |
| FR-3.14 | The dashboard shall show attendance updating live as students check in. | M |

#### 3.3.1 Verification rule

| Status | Condition |
|--------|-----------|
| **Verified** | Known device token and at least N of M challenges passed with valid room proof inside their windows |
| **Partial** | Valid room proof for at least one challenge, but fewer than N (late arrival or early departure) |
| **Unverified** | Only weak signals (network or location) and no valid room proof |
| **Flagged** | Anomalies: one device used for several students, many responses from one IP within the same second, impossible timing, or repeated invalid codes |

The first valid challenge response is recorded as the student's arrival time.

### 3.4 Spreadsheet export

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-4.1 | The lecturer shall be able to export a session's attendance as CSV and XLSX. | M |
| FR-4.2 | Each row shall contain: student name/ID, arrival time, challenges passed / total, status, signals matched, and manual-override note. | M |
| FR-4.3 | The export shall work offline from the locally stored session data when the server is on the lecturer's laptop. | S |
| FR-4.4 | The system should export a summary across sessions (attendance percentage per student). | C |

### 3.5 Comments and suggestions

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-5.1 | A student shall be able to send a text comment during a session. | M |
| FR-5.2 | Comments shall appear on the lecturer's dashboard in real time. | M |
| FR-5.3 | Comments written offline shall be queued and delivered when the connection returns. | S |
| FR-5.4 | The lecturer shall be able to mark comments as read or answered. | C |
| FR-5.5 | The system may offer anonymous comments, with the sender still known to the server for abuse control. | C |

### 3.6 Polls

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-6.1 | The lecturer shall be able to create a poll with a question and 2 to 6 options in under 30 seconds. | M |
| FR-6.2 | Students shall see the active poll and vote once. | M |
| FR-6.3 | Results shall update live on the lecturer's screen. | M |
| FR-6.4 | The lecturer shall be able to close a poll and optionally show the results to students. | S |
| FR-6.5 | The system may draft a poll question from the current learning material using an LLM. | C |

### 3.7 Lecture video summary

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-7.1 | The lecturer shall be able to upload a recording (audio or video file) exported from Zoom Meet or Google Classroom. | S |
| FR-7.2 | The system shall transcribe the recording and produce a structured summary (main topics, key points, questions raised) using an LLM. | S |
| FR-7.3 | The processing shall run in the background, with visible progress. | S |
| FR-7.4 | The lecturer shall be able to download the summary. | C |

### 3.8 Homework deep review

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-8.1 | A student shall be able to submit homework as text or a file. | S |
| FR-8.2 | The system shall produce an LLM review of each submission against a lecturer-provided rubric. | S |
| FR-8.3 | The system shall compare a submission to the other students' submissions (using embeddings and an LLM), reporting strengths, gaps, and similarity, without revealing other students' identities to students. | S |
| FR-8.4 | The lecturer shall see per-student reviews and a class-level overview. | S |
| FR-8.5 | Unusually high similarity between submissions shall be flagged for the lecturer. | C |

---

## 4. External Interface Requirements

### 4.1 User interfaces
- **Student:** a single mobile-first page with large buttons (check in, comment, vote, submit homework). It shall be usable with one hand and installable as a PWA.
- **Lecturer:** a dashboard with the session control, room code/QR for the projector, live attendance table, polls, comments, and uploads.

### 4.2 Software interfaces
- HTTPS REST endpoints and a WebSocket channel between phones/dashboard and server.
- LLM API through one internal `LLMClient` class (provider swappable).
- Speech-to-text (e.g. Whisper) through one internal service class.
- Spreadsheet generation through a library (e.g. openpyxl).

### 4.3 Hardware and network interfaces
- Classroom Wi-Fi, or a portable router or laptop hotspot running the server.
- A laptop and projector for the dashboard and room code.

---

## 5. Non-Functional Requirements

| ID | Category | Requirement | Priority |
|----|----------|-------------|----------|
| NFR-1 | Usability | A student shall complete a check-in in at most 2 taps/steps after the first registration. | M |
| NFR-2 | Usability | A lecturer shall start a session and display the room code in at most 3 actions. | M |
| NFR-3 | Performance | The system shall handle at least 60 simultaneous students with check-in responses under 2 seconds. | M |
| NFR-4 | Performance | Live updates (attendance, polls, comments) shall appear within 2 seconds. | S |
| NFR-5 | Reliability | Attendance data shall not be lost if the connection drops; it is queued and synced. | S |
| NFR-6 | Compatibility | The app shall work on current Chrome and Safari on Android and iOS. | M |
| NFR-7 | Security | Device tokens shall be random, unguessable, and sent only over a secure channel when hosted online. | S |
| NFR-8 | Security | LLM API keys shall stay on the server and never reach the browser. | M |
| NFR-9 | Privacy | The system shall store the minimum data needed (student ID, timestamps, signals), use demo identities in the demo, and allow deletion of a session's data. | M |
| NFR-10 | Privacy | Precise location shall not be stored; only a yes/no match against the room is kept. | S |
| NFR-11 | Maintainability | Code shall follow the class design from the diagrams; LLM and storage are behind interfaces so they can be swapped and mocked in tests. | M |
| NFR-12 | Demo-readiness | The full demo flow shall run within 10 minutes using a prepared dataset and a backup recording. | M |

---

## 6. Data Requirements

Main entities (to be detailed in the class diagram):

| Entity | Key fields |
|--------|-----------|
| Student | id, name, device token, registered at |
| Room | id, name, network range (optional) |
| Session | id, room, start, end, window, M, N, status |
| Challenge | id, session, opens at, closes at, code |
| CheckIn | id, challenge, student, device token, timestamp, client IP, signals, valid |
| AttendanceRecord | session, student, arrival, passed/total, status, override note |
| Poll / Vote | question, options, student, choice |
| Comment | session, student, text, timestamp, state |
| Recording / Summary | file, transcript, summary |
| Submission / Review | student, content, rubric, review, similarity |

---

## 7. Known Limitations

1. A web app cannot prove that a specific person is in the room. It can only show that a bound device produced room-bound responses at random moments. A student could hand over a phone, or relay a live code to someone outside. Risk is reduced by random challenges, device binding, anomaly flags, and the manual override, not removed.
2. The network signal is advisory because Wi-Fi leaks beyond the room and VPNs or mobile data change the apparent address.
3. Browsers cannot read the router's identity, so room matching depends on the server seeing the client's IP and on the room having its own subnet or a router the team controls.
4. Locked phones suspend web pages, so only explicit challenge responses count as presence.

---

## 8. Open Issues (for team decision)

| # | Question | Owner | Status |
|---|----------|-------|--------|
| OI-1 | Primary room proof: rotating QR/code only, or add audio token? | Team | Open |
| OI-2 | Values of N and M, and the policy for a student with a dead phone | Team / client | Open |
| OI-3 | Do the classrooms have separate subnets? Does the network allow device-to-device traffic? (network spike) | Team | Open |
| OI-4 | Hosted server or laptop on the classroom network for the demo? | Team | Open |
| OI-5 | LLM provider and speech-to-text choice | Team | Open |
| OI-6 | Retention period for attendance logs | Client | Open |

---

## 9. Traceability (summary)

| Objective (SoW) | Requirements |
|-----------------|--------------|
| O1 Automatic attendance | FR-1.x, FR-2.x, FR-3.x, FR-4.x, NFR-1, 2, 5, 9, 10 |
| O2 Comments | FR-5.x |
| O3 Polls | FR-6.x |
| O4 Video summary | FR-7.x |
| O5 Homework review | FR-8.x |
| O6 KISS / demo | NFR-1, 2, 6, 12 |

Each requirement ID will be referenced by the use cases (`04_Use_Cases.md`) and the test cases (`07_Test_Strategy.md`).
