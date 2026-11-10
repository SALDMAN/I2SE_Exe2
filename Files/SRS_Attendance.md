# Software Requirements Specification (SRS)

**Project:** Smart Classroom Web Application (attendance, comments, polls, lecture summary, homework review)
**Version:** 2.1  |  **Date:** 2026-11-09
**Team:** _<names / IDs>_

Priority legend: **M** = Must, **S** = Should, **C** = Could.

Items marked **[assumed]** are the team's working assumptions. They have not been confirmed by the lecturer or the team and should be reviewed (see section 8).

---

## 1. Introduction

### 1.1 Purpose
This document specifies the requirements of a web application that helps a lecturer run a class: it records attendance automatically, collects student comments and quick polls, summarizes a lecture video, and reviews homework.

### 1.2 Scope
The application has five features, in priority order:

1. **Attendance** (most important). Sessions open and close automatically from the timetable. Every student gets the same short, randomly timed spot checks on their own registered device. A student in the classroom who allows location has each check answered automatically when their location matches the lecture location. Every other student (online, location turned off, or location not matching) answers by tapping. The lecturer views and exports the results as a spreadsheet (CSV).
2. **Comments and suggestions.** Students send short comments to the lecturer, live during a lecture or afterwards. The lecturer can pop a comment up for the whole class.
3. **Quick polls and quizzes.** The lecturer creates a poll on the spot about the current material, with or without a correct answer. Students vote and the results are shown live.
4. **Lecture summary.** The lecturer uploads an MP4 of a lecture and an outside AI service produces a summary. Students read the summary of each previous lecture and download the MP4 from the summary page.
5. **Homework review.** A student may upload a homework PDF and receive a detailed review from an outside AI service, including an anonymous comparison with a dataset of other homework and anonymous examples of homework done right.

Out of scope: proving a student's physical location (location is supporting evidence only), integration with Moodle, Zoom, Google Meet or Classroom, building or training our own AI models, native mobile apps, push notifications, and production hosting beyond the course demo.

### 1.3 Definitions

| Term | Meaning |
|------|---------|
| Course | A taught subject with a weekly timetable and enrolled students |
| Session | One lecture meeting of a course, created automatically from the timetable |
| Spot check | A short, randomly timed request for the student to confirm presence |
| Response window | The time a spot check stays open (default 60 seconds) |
| Device token | A random secret stored on the student's device that binds the device to one student |
| N of M | A student must answer at least N of the M spot checks in a session |
| Attendance status | Present / Partial / Absent / Flagged |
| Comment | A short text message from a student to the lecturer |
| Poll | A question with 2 to 6 answer options, created by the lecturer, one vote per student |
| AI service | An outside service that takes a file and returns generated text. We call it; we do not build it |
| AI job | One request to the AI service for a lecture summary or a homework review |
| Dataset | The set of other homework PDFs that a review is compared against. Reachable only by the AI service and the lecturer |
| Classroom location | The point (latitude, longitude) and radius that define where one lecture (timetable slot) is held, set by the administrator |
| Location match | A student's reported location is inside the classroom radius when a spot check opens |
| Quiz | A poll in which the lecturer marks one option as correct |

---

## 2. Overall Description

### 2.1 Product perspective
A self-contained web application: a mobile-first student page, a lecturer dashboard, and a server with a database and local file storage. Features 4 and 5 call an outside AI service over the internet.

### 2.2 User classes

| User | Description |
|------|-------------|
| Student | Uses a phone or computer browser; attends in class or online; wants minimal effort |
| Lecturer | Wants reliable attendance data with no actions during the lecture; optionally runs polls, reads comments, and uploads videos |
| Administrator | Enters courses, timetables, student lists, and homework assignments; may be the lecturer |

### 2.3 Constraints
- Web-based only; no native apps and no dedicated hardware.
- No use of IP address, Bluetooth, Wi-Fi scanning, or room codes as presence evidence.
- Browser location is used only to auto-answer spot checks, and only for a student who allows it. A student who denies it is treated as an online student.
- Browsers cannot run pages in the background on a locked phone, so a spot check is answered (by location or by tap) only while the student page is open.
- Features 4 and 5 depend on an outside AI service; the choice of provider is open (section 8).
- Keep it simple: the team prefers dropping a feature detail over adding complexity.

### 2.4 Assumptions
- Every student has a device with a browser and an internet connection during the lecture.
- Students keep the student page open during the lecture to receive spot checks and polls.
- The classroom is within the accuracy of phone location (tens of meters); indoor accuracy can be worse. **[assumed]**
- The AI service accepts the files we send (an MP4 or PDFs) or can be fed text derived from them. **[assumed]**

---

## 3. Functional Requirements

### 3.1 Course setup

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1.1 | The administrator shall create a course with a weekly timetable (day, start time, end time). | M |
| FR-1.2 | The administrator shall import the course's student list from CSV. | S |
| FR-1.3 | The administrator shall set per course the values of M and N and the response window. Defaults: M = 4, N = 3, window = 60 seconds. | S |
| FR-1.4 | When creating a lecture (a timetable slot), the administrator shall set its classroom location: a point (latitude, longitude) and a radius chosen by the administrator. There is no default radius. The page offers a "use my current location" button to fill the point, and the administrator can edit the values. **[assumed]** that the location belongs to each lecture, not to the whole course. | M |

### 3.2 Student registration and device binding

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-2.1 | A student shall register once with their student ID, producing a device token stored on the device. | M |
| FR-2.2 | Every request from the student page shall carry the device token automatically; the student never types it. | M |
| FR-2.3 | A student shall be bound to one device token. A new device shall replace the old one only after lecturer approval. | S |
| FR-2.4 | Only one active session per student is allowed; a new login ends the previous one. | M |
| FR-2.5 | The system may require passkey (WebAuthn) user verification on each spot-check response. | C |

### 3.3 Automatic sessions

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-3.1 | The system shall open a session automatically at the timetabled start time and close it at the timetabled end time, with no lecturer action. | M |
| FR-3.2 | The lecturer may cancel a single session (e.g. a holiday) from the dashboard. | S |

### 3.4 Spot checks

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-4.1 | When a session opens, the server shall choose M random times within the session, keeping them hidden from students. | M |
| FR-4.2 | At each chosen time, the server shall open a spot check for the response window. | M |
| FR-4.3 | The student page shall show the spot check with a single "I'm here" button, alerted through the open page (SSE), unless FR-4.7 answers it automatically. | M |
| FR-4.4 | The server shall accept a response only if the device token is valid, the student is enrolled in the course, and the spot check is currently open. | M |
| FR-4.5 | The server shall reject a second response from the same student to the same spot check. | M |
| FR-4.6 | The server shall record each response: student, spot check, device token, server timestamp, and how it was answered (tap or location) with the distance from the classroom for location answers. | M |
| FR-4.7 | When a spot check opens and the student has allowed location, the student page shall read the device location and send it, answering the spot check without any tap. | M |
| FR-4.8 | The server shall accept a location answer only if the reported location is within the classroom radius, using the same checks as FR-4.4. Otherwise the answer is not recorded and the student page shows the normal "I'm here" button for the rest of the window. | M |
| FR-4.9 | A student who denies or has turned off location shall use the tap button on every spot check, like an online student. No penalty applies. | M |
| FR-4.10 | The system shall ask for location permission once, at the first session the student opens, and shall explain why. | S |

### 3.5 Attendance status

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-5.1 | When a session closes, the server shall compute each enrolled student's status using the rule in 3.5.1. | M |
| FR-5.2 | The lecturer shall be able to override any student's status, with the override and its reason recorded. | M |

#### 3.5.1 Status rule

| Status | Condition |
|--------|-----------|
| **Present** | Answered at least N of M spot checks from the bound device |
| **Partial** | Answered at least 1 but fewer than N spot checks |
| **Absent** | Answered no spot checks |
| **Flagged** | Responses for the same student came from more than one device token, or the student's device token was used for another student |

A Flagged status is shown together with the underlying count (e.g. "Flagged, 4/4").

### 3.6 Lecturer dashboard and attendance export

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-6.1 | The dashboard shall show, per session, each student's checks answered / total, how each was answered (location or tap), and status, updating live. | M |
| FR-6.2 | The dashboard shall show per student their attendance percentage across all sessions of the course. | S |
| FR-6.3 | The lecturer shall export a session's attendance, and the whole course's attendance, as CSV. | M |
| FR-6.4 | Each exported row shall contain: student ID, name, checks answered / total, status, override note. | M |

### 3.7 Comments and suggestions

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-7.1 | An enrolled student shall be able to send a text comment of up to 500 characters to the lecturer of a course. | S |
| FR-7.2 | While a session is open, a comment shall appear on the lecturer dashboard within 2 seconds (SSE), and shall be linked to that session. | S |
| FR-7.3 | A comment sent while no session is open shall be stored and shown to the lecturer in the course's comment list. **[assumed]** This is how "offline" is understood: the student does not need the lecturer to be online. A comment typed with no network connection is not queued on the device. | S |
| FR-7.4 | The lecturer shall see each comment with the student's name and the time received. **[assumed]** | S |
| FR-7.5 | The lecturer shall be able to filter the comment list by session and export it as CSV. | C |
| FR-7.6 | During an open session the lecturer shall be able to pop up one chosen comment on every connected student page within 2 seconds (SSE). The popped-up comment shows the text without the author's name. **[assumed]** | S |
| FR-7.7 | The lecturer shall be able to dismiss the popped-up comment; it also disappears when the session closes. | S |

### 3.8 Quick polls

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-8.1 | During an open session the lecturer shall create a poll in one step: a question, 2 to 6 text options, and optionally one option marked correct. A poll with a correct option is a quiz. | S |
| FR-8.2 | A new poll shall open for votes immediately and appear on the student page (SSE). | S |
| FR-8.3 | An enrolled student shall be able to vote once per poll; a second vote shall be rejected. The vote is a single tap. | S |
| FR-8.4 | The lecturer shall see the vote counts per option update live. The lecturer sees counts only, not who voted for what. **[assumed]** | S |
| FR-8.5 | The lecturer shall close a poll; a poll also closes when its session closes. After closing, votes are rejected. | S |
| FR-8.6 | When a poll closes, students shall see the results. In a quiz, each student also sees the correct answer and whether their own answer was right. **[assumed]** | S |
| FR-8.7 | The correct option of a quiz shall never be sent to students before the quiz closes. | S |
| FR-8.8 | Quiz results are not saved as grades or scores across lectures. **[assumed]** | S |

### 3.9 Lecture summary

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-9.1 | The lecturer shall upload one MP4 file for a session, up to the size limit in NFR-9. | S |
| FR-9.2 | After upload, the server shall create an AI job and send the video to the AI service. The upload request shall return immediately; the work runs in the background. | S |
| FR-9.3 | The page shall show the job state: pending, done, or failed. | S |
| FR-9.4 | When the job is done, the summary text shall be stored and shown to the lecturer and to the course's enrolled students. | S |
| FR-9.5 | When the AI service fails or times out, the job shall be marked failed with a clear message, and the lecturer shall be able to retry. | S |
| FR-9.6 | The lecturer shall be able to download the summary as a text file. | C |
| FR-9.7 | An enrolled student shall see the list of previous lectures of the course that have a summary, and open each summary. | S |
| FR-9.8 | Each summary page shall have a link to download the MP4 recording of that lecture. Only the course's enrolled students and the lecturer can download it. The recording is download only; the app has no video player. **[assumed]** | S |

### 3.10 Homework review

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-10.1 | The administrator or lecturer shall create a homework assignment for a course (title) and may upload a dataset for it: sample solutions and other homework PDFs. | S |
| FR-10.2 | A student shall be able, but is never required, to upload one PDF for an assignment of a course they are enrolled in. | S |
| FR-10.3 | At upload the student shall choose whether the PDF may be added to the assignment's dataset. The default is no. | S |
| FR-10.4 | After upload, the server shall create an AI job that sends the PDF and the assignment's dataset to the AI service for a detailed review that includes an anonymous comparison with the dataset (how the student's work compares to the others) and, where useful, examples of homework done right. | S |
| FR-10.5 | The student shall see the job state (pending, done, failed) and, when done, the review text. A student sees only their own review. | S |
| FR-10.6 | When the AI service fails or times out, the job shall be marked failed with a clear message, and the student shall be able to retry. | S |
| FR-10.7 | A student's PDF shall be added to the dataset only if the student chose so at upload, and only after that upload's own review has finished. **[assumed]** | S |
| FR-10.8 | The lecturer shall see which students uploaded and the same individual review each student got. There is no separate class-wide comparison. | S |
| FR-10.9 | The dataset shall be reachable only by the AI service and the lecturer. No page or endpoint shall let a student list, open, or download a dataset document. | M |
| FR-10.10 | Examples of homework done right shall be shown to the student only as anonymous excerpts inside the review. They shall come only from sample solutions or from students who opted in at upload (FR-10.3). They shall carry no name or student ID. The files sent to the AI service shall use neutral identifiers, not names. **[assumed]** | S |

---

## 4. External Interface Requirements

- **Student page:** one mobile-first page; shows "no active session", or the active session with any open spot check (large "I'm here" button), the open poll or quiz, a popped-up comment when the lecturer shows one, a comment box, previous lectures with their summaries and MP4 download links, and homework upload with the review.
- **Lecturer dashboard:** course list, timetable with the classroom location of each lecture, session list, attendance table with the method per check, export buttons, comment list with a "pop up" action, poll and quiz creation and live results, video upload, homework overview.
- **Communication:** HTTPS for all traffic; SSE (Server-Sent Events) for live spot checks, polls, comments, and dashboard updates.
- **AI service:** one outbound HTTPS client behind a small interface, so the provider can be changed or replaced by a fake for tests and demos. Request and response format depend on the provider chosen.

---

## 5. Non-Functional Requirements

| ID | Category | Requirement | Priority |
|----|----------|-------------|----------|
| NFR-1 | Usability | For attendance the lecturer shall perform zero actions during a lecture. Polls, comments, and uploads are optional lecturer actions. | M |
| NFR-2 | Usability | A student shall answer a spot check or a poll with one tap after first registration. | M |
| NFR-3 | Performance | The system shall handle at least 200 simultaneous students per session; a spot check, poll, or comment shall reach its audience and a response shall be acknowledged within 2 seconds. | M |
| NFR-4 | Compatibility | The student page shall work on current Chrome and Safari on Android, iOS, and desktop. | M |
| NFR-5 | Security | Device tokens shall be random (at least 128 bits) and sent only over HTTPS. | M |
| NFR-6 | Security | Spot-check times shall never be sent to students before the spot check opens. | M |
| NFR-7 | Privacy | The system shall store only: student ID and name, device token, response timestamps, comments, poll votes, uploaded files with their AI results. For location answers it stores only the result (match, and distance). Raw coordinates are used to compute the match and are not stored. Location is requested only with the student's permission. No IP-based or other sensor data is collected. | M |
| NFR-8 | Reliability | A slow, failing, or unreachable AI service shall never block or break attendance, comments, or polls. AI calls run in the background with a timeout (default 10 minutes for a summary, 2 minutes for a review; an 85-minute lecture took about 3.5 minutes in the tools test). **[assumed]** | M |
| NFR-9 | Limits | Uploads are limited to 200 MB for MP4 and 10 MB for PDF; other files and larger files are rejected with a clear message. The demo uses a short MP4. **[assumed]** | S |
| NFR-10 | Security | Uploaded files shall be stored outside the web-served folder, and each file shall be downloadable only by its owner and the lecturer. | S |
| NFR-11 | Privacy | A student's homework shall never be shown to other students in full or with a name; it enters the dataset only by explicit opt-in (FR-10.3), and appears to others only as an anonymous excerpt (FR-10.10). Data sent to the AI service is limited to the files named in FR-9.2 and FR-10.4. | M |

---

## 6. Data Requirements

| Entity | Key fields |
|--------|-----------|
| Student | id, student number, name, device token, registered at |
| Course | id, name, M, N, response window |
| TimetableSlot | course, weekday, start time, end time, classroom latitude, longitude, radius |
| Enrollment | course, student |
| Session | id, course, start, end, status (scheduled / open / closed / cancelled) |
| SpotCheck | id, session, opens at, closes at |
| Response | spot check, student, device token, timestamp, method (tap / location), distance (location only) |
| AttendanceRecord | session, student, answered / total, status, override note |
| Comment | id, course, session (optional), student, text, created at, popped up (yes / no) |
| Poll | id, session, question, status (open / closed) |
| PollOption | id, poll, text, is correct (quiz only) |
| PollVote | poll, student, option; one vote per student per poll |
| LectureSummary | id, session, video file path, status (pending / done / failed), summary text, error message |
| HomeworkAssignment | id, course, title |
| HomeworkSubmission | id, assignment, student, PDF file path, add to dataset (yes/no), status (pending / done / failed), review text, error message |
| DatasetDocument | id, assignment, PDF file path, neutral id, source (sample solution / other homework / student submission) |

---

## 7. Known Limitations

1. The system does not prove where a student is. Browser location can be faked or be inaccurate indoors, so a location answer is evidence, not proof. A student in class whose location does not match falls back to tapping, like an online student.
2. The system cannot prove a student is paying attention; it only shows the bound device answered at random moments.
3. A student could hand their device to someone else. Device binding and one active session reduce this risk; the optional passkey would reduce it further. None removes it.
4. There are no push notifications: a student whose page is closed or whose phone is locked misses that spot check or poll. Location is also read only while the page is open, so a locked phone answers nothing, by location or by tap.
5. Summaries and reviews are generated by an outside AI service. They can be wrong or incomplete, are not graded, and are advisory only. Quality and speed depend on the provider.
6. Homework "comparison with other students' work" compares against a prepared dataset, not live against classmates. **[assumed]**

---

## 8. Open Questions

1. Which AI service and model will be used, and what does it accept (video, PDF, text)? The tools test decides; the team does not know yet.
2. Team size, deadline, and role assignments (see SOW).
3. Why the team rejected IP address as an in-room signal. It stays excluded either way.

Decided:
- The lecturer's homework review is the same individual review the student gets, not a class-wide comparison (FR-10.8).
- Students see lecture summaries and can download the MP4 (FR-9.4, FR-9.8).
- The AI in the demo is basic: simple prompts on short samples, with a fake service as fallback. Attendance, comments and polls run fully.
- The administrator sets the classroom point and the radius for each lecture when creating it (FR-1.4).
- Attendance is one system for everyone: in-class students with a location match are answered automatically; a student who denies location is treated as online (FR-4.7–FR-4.9).
- Polls and quizzes share one tool (FR-8.1); comments go to the lecturer, who can pop one up for the class (FR-7.6).
- The dataset is reachable only by the AI service and the lecturer; examples shown to students are anonymous (FR-10.9, FR-10.10).
