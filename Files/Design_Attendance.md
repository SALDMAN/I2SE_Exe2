# Low-Level Design: Class and Sequence Diagrams

**Project:** Smart Classroom Web Application
**Version:** 1.0  |  **Date:** 2026-11-09
**Related:** `Components_Attendance.md` (C1–C17), `SRS_Attendance.md` (v2.1), `UseCases_Attendance.md`

Method and field names here are the plan for the code. If the code changes a name, update this file. The class diagrams are the base for the testing strategy (course item 8): every class below gets tests.

Diagrams use Mermaid and render on GitHub and in Obsidian.

---

## 1. Class diagrams

### 1.1 Entities (component C17)

One `@dataclass` per SRS §6 entity. Fields are the plan, not the full schema.

```mermaid
classDiagram
    class Student {
        id
        student_number
        name
        device_token
        login_id
        registered_at
    }
    class Course {
        id
        name
        m
        n
        window_seconds
    }
    class TimetableSlot {
        id
        course_id
        weekday
        start_time
        end_time
        lat
        lng
        radius_m
    }
    class Enrollment {
        course_id
        student_id
    }
    class Session {
        id
        course_id
        slot_id
        start
        end
        status
    }
    class SpotCheck {
        id
        session_id
        opens_at
        closes_at
    }
    class Response {
        spot_check_id
        student_id
        device_token
        timestamp
        method
        distance_m
    }
    class AttendanceRecord {
        session_id
        student_id
        answered
        total
        status
        override_status
        override_note
    }
    class Comment {
        id
        course_id
        session_id
        student_id
        text
        created_at
        popped_up
    }
    class Poll {
        id
        session_id
        question
        status
    }
    class PollOption {
        id
        poll_id
        text
        is_correct
    }
    class PollVote {
        poll_id
        student_id
        option_id
    }
    class LectureSummary {
        id
        session_id
        video_path
        status
        summary_text
        error
    }
    class HomeworkAssignment {
        id
        course_id
        title
    }
    class HomeworkSubmission {
        id
        assignment_id
        student_id
        pdf_path
        add_to_dataset
        status
        review_text
        error
    }
    class DatasetDocument {
        id
        assignment_id
        neutral_id
        pdf_path
        source
    }

    Course "1" --> "many" TimetableSlot
    Course "many" -- "many" Student : Enrollment
    TimetableSlot "1" --> "many" Session
    Session "1" --> "many" SpotCheck
    SpotCheck "1" --> "many" Response
    Student "1" --> "many" Response
    Session "1" --> "many" AttendanceRecord
    Student "1" --> "many" AttendanceRecord
    Course "1" --> "many" Comment
    Session "1" --> "many" Poll
    Poll "1" --> "2..6" PollOption
    Poll "1" --> "many" PollVote
    Session "1" --> "0..1" LectureSummary
    Course "1" --> "many" HomeworkAssignment
    HomeworkAssignment "1" --> "many" HomeworkSubmission
    HomeworkAssignment "1" --> "many" DatasetDocument
```

Database rules that back the model: `UNIQUE(spot_check_id, student_id)` on Response, `UNIQUE(poll_id, student_id)` on PollVote, `UNIQUE(device_token)` on Student.

### 1.2 Services and background classes (components C1–C16)

```mermaid
classDiagram
    class Scheduler {
        tick()
        run_forever()
    }
    class Clock {
        now()
    }
    class FakeClock {
        set(time)
        advance(seconds)
    }
    class AttendanceService {
        open_due_sessions(now)
        update_spot_checks(now)
        close_due_sessions(now)
        answer_by_tap(student, spot_check_id)
        answer_by_location(student, spot_check_id, lat, lng)
        compute_statuses(session_id)
        override_status(record_id, status, reason)
        cancel_session(session_id)
    }
    class LocationChecker {
        distance_m(lat1, lng1, lat2, lng2)
        is_inside(lat, lng, slot)
    }
    class CourseService {
        create_course(name, m, n, window)
        add_lecture(course_id, day, start, end, lat, lng, radius)
        import_students(course_id, csv_file)
        register_device(student_number)
        approve_new_device(student_id)
    }
    class PollService {
        create_poll(session_id, question, options, correct_index)
        vote(student, poll_id, option_id)
        close_poll(poll_id)
        close_ended_polls(now)
        counts(poll_id)
    }
    class CommentService {
        send(student, text)
        list(course_id, session_id)
        pop_up(comment_id)
        dismiss(comment_id)
    }
    class SummaryService {
        upload_video(session_id, file)
        mark_done(summary_id, text)
        mark_failed(summary_id, error)
        list_for_student(student, course_id)
    }
    class HomeworkService {
        create_assignment(course_id, title)
        add_dataset_document(assignment_id, file, source)
        submit(student, assignment_id, file, add_to_dataset)
        mark_done(submission_id, text)
        mark_failed(submission_id, error)
    }
    class FileService {
        save(file, kind, max_size)
        open_for(user, file_id)
    }
    class AiJobRunner {
        run_pending()
    }
    class AiClient {
        <<interface>>
        summarize_video(path)
        review_homework(pdf_path, dataset)
    }
    class RealAiClient
    class FakeAiClient
    class EventHub {
        subscribe(audience)
        publish(audience, event)
    }
    class CsvService {
        parse_students(file)
        export_session(session_id)
        export_course(course_id)
    }
    class Repository {
        <<SQL only>>
        get / insert / update / query
    }

    Scheduler --> Clock
    Scheduler --> AttendanceService
    Scheduler --> PollService
    Scheduler --> AiJobRunner
    FakeClock --|> Clock
    AttendanceService --> LocationChecker
    AttendanceService --> EventHub
    AttendanceService --> Repository
    PollService --> EventHub
    PollService --> Repository
    CommentService --> EventHub
    CommentService --> Repository
    CourseService --> Repository
    CourseService --> CsvService
    SummaryService --> FileService
    SummaryService --> Repository
    HomeworkService --> FileService
    HomeworkService --> Repository
    AiJobRunner --> AiClient
    AiJobRunner --> SummaryService
    AiJobRunner --> HomeworkService
    RealAiClient ..|> AiClient
    FakeAiClient ..|> AiClient
```

The Routes (C1) call only the services; they hold no rules. There is one repository class per entity group (`StudentRepository`, `SessionRepository`, and so on); the diagram shows them as one box.

**Key design decisions**
1. `AttendanceService` has two entry points, `answer_by_tap` and `answer_by_location`. Both end in one private check (token, enrollment, open window, duplicate), so the rules cannot drift apart.
2. `LocationChecker` has no state and no repository. It can be tested with plain numbers.
3. `Scheduler.tick()` takes the time from `Clock` only. Tests give it a `FakeClock`. `AttendanceService` takes its random source as a constructor argument (default `secrets.SystemRandom()`), so tests can check the spot-check times deterministically.
4. `AiClient` is an interface with a real and a fake implementation. Nothing else knows which one is used.
5. Services never return a secret early: spot-check times and a quiz's `is_correct` are removed by the service from anything sent to a student (NFR-6, FR-8.7).

---

## 2. Sequence diagrams

### S1: Scheduler opens a session and runs spot checks (UC3, UC4)

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant C as Clock
    participant A as AttendanceService
    participant R as Repositories
    participant H as EventHub

    loop every second
        S->>C: now()
        C-->>S: t
        S->>A: open_due_sessions(t)
        A->>R: find slots starting now with no session
        A->>R: insert Session (open)
        A->>R: insert M SpotCheck rows at hidden random times
        S->>A: update_spot_checks(t)
        A->>R: find checks with opens_at <= t < closes_at
        A->>H: publish(students of course, "spot_check_open", id and closes_at)
        A->>R: mark checks past closes_at as closed
        S->>A: close_due_sessions(t)
        A->>R: find sessions with end <= t
        A->>A: compute_statuses(session_id)
    end
```

Everything is read from the database each tick, so a server restart changes nothing. Spot-check times are drawn with `secrets.SystemRandom()` when the session opens and are never sent before the check opens.

### S2: In-class student answered automatically by location (UC3)

```mermaid
sequenceDiagram
    participant P as Student page
    participant R as Routes
    participant A as AttendanceService
    participant L as LocationChecker
    participant DB as Repositories
    participant H as EventHub

    H-->>P: SSE "spot_check_open" (id, closes_at)
    P->>P: getCurrentPosition()
    P->>R: POST /spot-checks/{id}/respond (device token cookie, lat, lng)
    R->>A: answer_by_location(student, id, lat, lng)
    A->>DB: load check, session, slot, enrollment
    A->>A: window open? enrolled? token matches?
    A->>L: is_inside(lat, lng, slot)
    L-->>A: inside, distance
    A->>DB: insert Response (method=location, distance)
    A->>H: publish(lecturer, "table_changed")
    A-->>R: result "recorded"
    R-->>P: 200 recorded
    P->>P: show confirmation
```

Alternatives:
- `is_inside` is false: no Response is inserted, the result is `"outside"`, and the page shows the "I'm here" button (S3).
- Location denied, unavailable, or slow: the page never sends coordinates and shows the button directly.
- Window closed, token unknown, or not enrolled: result `"rejected"` with the reason.
- Duplicate: the `UNIQUE(spot_check_id, student_id)` rule stops it and the result is `"already_answered"`.

The coordinates exist only inside this call. Only `method` and `distance_m` are stored.

### S3: Tap answer (UC4)

```mermaid
sequenceDiagram
    participant P as Student page
    participant R as Routes
    participant A as AttendanceService
    participant DB as Repositories
    participant H as EventHub

    P->>R: POST /spot-checks/{id}/respond (device token cookie)
    R->>A: answer_by_tap(student, id)
    A->>DB: load check, session, enrollment
    A->>A: window open? enrolled? token matches?
    A->>DB: insert Response (method=tap)
    A->>H: publish(lecturer, "table_changed")
    A-->>R: "recorded"
    R-->>P: 200 recorded
```

A request with no coordinates is a tap. The checks are the same as in S2.

### S4: Session closes and status is computed (UC5)

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant A as AttendanceService
    participant DB as Repositories
    participant H as EventHub
    participant LP as Lecturer page

    S->>A: close_due_sessions(t)
    A->>DB: set session status = closed
    A->>DB: load enrolled students and all Response rows
    loop each enrolled student
        A->>A: count answers, check device tokens
        A->>A: status = Present / Partial / Absent / Flagged
        A->>DB: upsert AttendanceRecord
    end
    A->>H: publish(lecturer, "table_changed")
    H-->>LP: SSE "table_changed"
    LP->>LP: reload table
    Note over LP,A: Later: override_status(record_id, status, reason)<br/>keeps the computed status and stores the override
```

Rules: Present if `answered >= N`; Partial if `1 <= answered < N`; Absent if `answered == 0`; Flagged if one student answered from more than one token or one token answered for more than one student. A cancelled session creates no records.

### S5: Quiz from creation to result (UC8, UC9)

```mermaid
sequenceDiagram
    participant LP as Lecturer page
    participant SP as Student page
    participant R as Routes
    participant P as PollService
    participant DB as Repositories
    participant H as EventHub

    LP->>R: POST /polls (question, options, correct option)
    R->>P: create_poll(session_id, question, options, correct_index)
    P->>DB: insert Poll and PollOption rows
    P->>H: publish(students, "poll_open", question and options, no is_correct)
    H-->>SP: SSE "poll_open"
    SP->>R: POST /polls/{id}/vote (option)
    R->>P: vote(student, id, option)
    P->>DB: insert PollVote
    P->>H: publish(lecturer, "poll_counts", counts only)
    LP->>R: POST /polls/{id}/close
    R->>P: close_poll(id)
    P->>DB: set status = closed
    P->>H: publish(students, "poll_closed", results and correct option)
    H-->>SP: SSE "poll_closed"
    SP->>SP: show results, right or wrong
```

Alternatives: a second vote, a vote after close, or a vote from a non-enrolled student is rejected. When the session ends first, `close_ended_polls(now)` (called by the Scheduler) closes the poll the same way. A poll with no correct option sends no right or wrong feedback.

### S6: Lecture summary through the AI job (UC10, UC11)

```mermaid
sequenceDiagram
    participant LP as Lecturer page
    participant R as Routes
    participant SS as SummaryService
    participant F as FileService
    participant DB as Repositories
    participant S as Scheduler
    participant J as AiJobRunner
    participant AI as AiClient
    participant SP as Student page

    LP->>R: POST /sessions/{id}/video (MP4)
    R->>SS: upload_video(session_id, file)
    SS->>F: save(file, "video", max_size)
    SS->>DB: insert LectureSummary (pending)
    R-->>LP: 202 accepted, state pending

    S->>J: run_pending()
    J->>DB: take one pending summary
    J->>AI: summarize_video(path)
    alt service answers in time
        AI-->>J: summary text
        J->>SS: mark_done(id, text)
    else error or timeout
        AI-->>J: error
        J->>SS: mark_failed(id, message)
    end

    SP->>R: GET /courses/{id}/lectures
    R->>SS: list_for_student(student, course_id)
    SS-->>SP: lectures with summaries
    SP->>R: GET /lectures/{id}/recording
    R->>F: open_for(student, file_id)
    F-->>SP: the MP4 (enrolled student), or 403 otherwise
```

The upload reply does not wait for the AI. A failed job can be retried; the `pending` state survives a server restart because it lives in the database.

### S7: Homework submission and review (UC12, UC13)

```mermaid
sequenceDiagram
    participant SP as Student page
    participant R as Routes
    participant HS as HomeworkService
    participant F as FileService
    participant DB as Repositories
    participant J as AiJobRunner
    participant AI as AiClient

    SP->>R: POST /assignments/{id}/submit (PDF, add_to_dataset)
    R->>HS: submit(student, id, file, add_to_dataset)
    HS->>F: save(file, "pdf", max_size)
    HS->>DB: insert HomeworkSubmission (pending)
    R-->>SP: 202 accepted

    J->>DB: take pending submission
    J->>DB: load dataset documents with neutral ids
    J->>AI: review_homework(pdf_path, dataset)
    AI-->>J: review text with anonymous comparison and excerpts
    J->>HS: mark_done(id, text)
    opt add_to_dataset is true
        HS->>DB: insert DatasetDocument (source = student submission)
    end

    SP->>R: GET /submissions/{id}
    R->>HS: get_for_student(student, id)
    HS-->>SP: own review only
```

Privacy rules: the dataset documents are sent with neutral ids such as `doc-1`, never with names or student IDs. No route returns dataset files to a student. The student's own PDF joins the dataset only when the opt-in is set, and only after the review is done.

---

## 3. Coverage of use cases

| Use case | Class diagram | Sequence |
|----------|---------------|----------|
| UC1 | CourseService, CsvService, TimetableSlot | not drawn (plain CRUD) |
| UC2 | CourseService, Student | not drawn (plain CRUD) |
| UC3 | AttendanceService, LocationChecker, Scheduler | S1, S2 |
| UC4 | AttendanceService | S1, S3 |
| UC5 | AttendanceService, CsvService | S4 |
| UC6, UC7 | CommentService, EventHub | not drawn (same event pattern as S5) |
| UC8, UC9 | PollService | S5 |
| UC10, UC11 | SummaryService, FileService, AiJobRunner, AiClient | S6 |
| UC12, UC13 | HomeworkService, FileService, AiJobRunner, AiClient | S7 |

The comment pop-up (UC7) is the same pattern as S5: the lecturer posts, the service stores the flag, the hub pushes an event to students.
