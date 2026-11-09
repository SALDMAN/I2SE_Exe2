# Software Requirements Specification (SRS)

**Project:** Automatic Class Attendance Web Application
**Version:** 1.1  |  **Date:** 2026-11-08
**Team:** _<names / IDs>_

Priority legend: **M** = Must, **S** = Should, **C** = Could.

---

## 1. Introduction

### 1.1 Purpose
This document specifies the requirements of a web application that tracks student attendance in lectures, both in the classroom and online.

### 1.2 Scope
The system records attendance automatically, with no action required from the lecturer during a lecture. Students confirm presence by answering short, randomly timed spot checks from their own registered device. The lecturer views and exports the results.

Out of scope: proving a student's physical location, comments, polls, lecture summaries, homework review, and integration with external systems (Moodle, Zoom, etc.).

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

---

## 2. Overall Description

### 2.1 Product perspective
A self-contained web application: a mobile-first student page, a read-only lecturer dashboard, and a server with a database.

### 2.2 User classes

| User | Description |
|------|-------------|
| Student | Uses a phone or computer browser; attends in class or online; wants minimal effort |
| Lecturer | Wants reliable attendance data with no actions during the lecture |
| Administrator | Enters courses, timetables, and student lists once per semester; may be the lecturer |

### 2.3 Constraints
- Web-based only; no native apps and no dedicated hardware.
- No use of IP address, geolocation, Bluetooth, Wi-Fi scanning, or room codes as presence evidence.
- Browsers cannot run pages in the background on a locked phone, so only explicit spot-check responses count.

### 2.4 Assumptions
- Every student has a device with a browser and an internet connection during the lecture.
- Students keep the student page open during the lecture to receive spot checks.

---

## 3. Functional Requirements

### 3.1 Course setup

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1.1 | The administrator shall create a course with a weekly timetable (day, start time, end time). | M |
| FR-1.2 | The administrator shall import the course's student list from CSV. | S |
| FR-1.3 | The administrator shall set per course the values of M and N and the response window. Defaults: M = 4, N = 3, window = 60 seconds. | S |

### 3.2 Student registration and device binding

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-2.1 | A student shall register once with their student ID, producing a device token stored on the device. | M |
| FR-2.2 | Every spot-check response shall carry the device token automatically; the student never types it. | M |
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
| FR-4.3 | The student page shall show the spot check with a single "I'm here" button, alerted through the open page (SSE). | M |
| FR-4.4 | The server shall accept a response only if the device token is valid, the student is enrolled in the course, and the spot check is currently open. | M |
| FR-4.5 | The server shall reject a second response from the same student to the same spot check. | M |
| FR-4.6 | The server shall record each response: student, spot check, device token, server timestamp. | M |

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

### 3.6 Lecturer dashboard and export

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-6.1 | The dashboard shall show, per session, each student's checks answered / total and status, updating live. | M |
| FR-6.2 | The dashboard shall show per student their attendance percentage across all sessions of the course. | S |
| FR-6.3 | The lecturer shall export a session's attendance, and the whole course's attendance, as CSV. | M |
| FR-6.4 | Each exported row shall contain: student ID, name, checks answered / total, status, override note. | M |

---

## 4. External Interface Requirements

- **Student page:** one mobile-first page; shows "no active session", or the active session and any open spot check with a large "I'm here" button.
- **Lecturer dashboard:** course list, session list, attendance table, export button. No controls are needed during a lecture.
- **Communication:** HTTPS for all traffic; SSE (Server-Sent Events) for live spot checks and dashboard updates.

---

## 5. Non-Functional Requirements

| ID | Category | Requirement | Priority |
|----|----------|-------------|----------|
| NFR-1 | Usability | The lecturer shall perform zero actions during a lecture. | M |
| NFR-2 | Usability | A student shall answer a spot check with one tap after first registration. | M |
| NFR-3 | Performance | The system shall handle at least 200 simultaneous students per session; a spot check shall reach students and a response shall be acknowledged within 2 seconds. | M |
| NFR-4 | Compatibility | The student page shall work on current Chrome and Safari on Android, iOS, and desktop. | M |
| NFR-5 | Security | Device tokens shall be random (at least 128 bits) and sent only over HTTPS. | M |
| NFR-6 | Security | Spot-check times shall never be sent to students before the spot check opens. | M |
| NFR-7 | Privacy | The system shall store only student ID, name, device token, and response timestamps; it shall collect no location, IP-based, or device-sensor data. | M |

---

## 6. Data Requirements

| Entity | Key fields |
|--------|-----------|
| Student | id, student number, name, device token, registered at |
| Course | id, name, M, N, response window |
| TimetableSlot | course, weekday, start time, end time |
| Enrollment | course, student |
| Session | id, course, start, end, status (scheduled / open / closed / cancelled) |
| SpotCheck | id, session, opens at, closes at |
| Response | spot check, student, device token, timestamp |
| AttendanceRecord | session, student, answered / total, status, override note |

---

## 7. Known Limitations

1. The system does not prove where a student is; students in class and online are treated the same.
2. The system cannot prove a student is paying attention; it only shows the bound device answered at random moments.
3. A student could hand their device to someone else. Device binding, one active session, and the optional passkey reduce this risk; they do not remove it.
4. There are no push notifications: a student whose page is closed or whose phone is locked misses that spot check.
