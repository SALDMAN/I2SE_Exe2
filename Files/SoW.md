# Statement of Work (SoW)

**Project:** Web-Based Teaching Tool (Vibe Coding & Deep Designing)
**Course:** Introduction to Software Engineering (OOP, OOD) – Ex2
**Version:** 0.1 (draft)  |  **Date:** _<date>_
**Team:** _<names / IDs>_  |  **Client:** _<lecturer / course staff>_

---

## 1. Purpose

This document defines what the team will deliver, how, when, and under which assumptions and constraints. It is the agreement between the team and the client on the project's scope and acceptance.

## 2. Background

Existing online teaching tools are fragmented and do not support real in-class learning well. The client asks for a simpler, LLM-ready set of tools, delivered as a web application that works in a real classroom and can be demonstrated live in 10 minutes.

## 3. Objectives

| ID | Objective |
|----|-----------|
| O1 | Provide **automatic attendance** with a spreadsheet output for the lecturer (main objective). |
| O2 | Let students send comments and suggestions in real time or offline. |
| O3 | Let lecturers create on-the-spot (KISS) polls. |
| O4 | Provide an ML/LLM summary of a recorded class video. |
| O5 | Provide a deep LLM review of homework, including comparison to other students' work. |
| O6 | Deliver a KISS, web-based, smartphone-friendly app that runs in a live class demo. |

## 4. Scope of Work

### 4.1 In scope
- Design documentation: README, tools report, SoW, SRS, use cases, components, class and sequence diagrams, test strategy.
- A working demo web application covering at least O1, and O2 to O5 as far as is realistic within the demo time.
- Real-life testing of every tool that is proposed.
- A live in-class demo (10 minutes) and a short results summary.

### 4.2 Out of scope
- Integration with Moodle, Zoom, Google Classroom, or any external platform (recordings and files are imported manually).
- Integration with personal or sensitive data systems; the demo uses demo identities only.
- Native mobile apps.
- Production-grade security, scalability, monitoring, and deployment.
- Face recognition, Bluetooth/NFC beacons, and other methods not feasible in a browser.

## 5. Deliverables

| # | Deliverable | Format | Location |
|---|-------------|--------|----------|
| D1 | Repository with problem definition and scope | README.md | `/README.md` |
| D2 | Technical tools report (with real-life test results) | Markdown | `docs/01_Tools_Report.md` |
| D3 | Statement of Work | Markdown | `docs/02_SoW.md` |
| D4 | Software Requirements Specification | Markdown | `docs/03_SRS.md` |
| D5 | Main use cases (client stories) | Markdown | `docs/04_Use_Cases.md` |
| D6 | Main software components | Markdown + diagram | `docs/05_Components.md` |
| D7 | Class and sequence diagrams | Markdown + diagrams | `docs/06_Class_and_Sequence_Diagrams.md` |
| D8 | Testing strategy (covering the classes in D7) | Markdown | `docs/07_Test_Strategy.md` |
| D9 | Demo web application (vibe-coded or composed) | Source code | `src/` |
| D10 | Live demo (10 min) and demo script | Presentation | `demo/demo_script.md` |
| D11 | Conclusion and short description of results | Markdown | `docs/08_Conclusion.md` |

## 6. Approach

1. **Design first.** Requirements, use cases, and diagrams are written before the main implementation, and they act as the prompts for AI-assisted coding.
2. **Compose over build.** Reuse existing libraries and code wherever possible (QR generation, spreadsheet export, speech-to-text, LLM APIs).
3. **Spike before committing.** Each key technical decision (attendance verification, real-time channel, offline support, LLM provider) is tested with a short, real experiment and recorded in the tools report.
4. **Reliable core, optional extras.** The attendance feature must work end to end before any other feature is polished. Risky features are stretch goals.

## 7. Technical Direction (Draft)

- **Architecture:** a single Python project (FastAPI) serving server-rendered pages with a small amount of JavaScript, installable as a PWA. Final choice depends on the tools report.
- **Storage:** SQLite.
- **Real-time:** WebSocket.
- **Attendance verification:** layered evidence (device binding, room-bound rotating code, classroom network/router location when available, and random presence challenges during the lecture). See the SRS for the exact rules.
- **LLM and ML:** accessed through a single swappable client class.

## 8. Milestones and Schedule

_Dates to be filled in against the course calendar._

| Milestone | Content | Target date |
|-----------|---------|-------------|
| M1 | Repository, README, scope | _TBD_ |
| M2 | Tools report with real-life spikes (including the classroom network test) | _TBD_ |
| M3 | SoW and SRS approved | _TBD_ |
| M4 | Use cases, components, class and sequence diagrams | _TBD_ |
| M5 | Test strategy | _TBD_ |
| M6 | Demo implementation (attendance end to end) | _TBD_ |
| M7 | Remaining features and demo rehearsal | _TBD_ |
| M8 | Live demo and conclusion | _TBD_ |

## 9. Roles and Responsibilities

| Role | Responsibility | Person |
|------|----------------|--------|
| Project lead | Planning, scope, integration | _TBD_ |
| Requirements and design | SRS, use cases, diagrams | _TBD_ |
| Backend | Services, database, LLM integration | _TBD_ |
| Frontend | Student and lecturer pages | _TBD_ |
| QA and testing | Test strategy, test execution | _TBD_ |
| Client / lecturer | Review, feedback, acceptance | _TBD_ |

## 10. Assumptions

- Students have a smartphone with a modern browser and can scan a QR code or open a link.
- The classroom has Wi-Fi, or the team can bring a portable router or hotspot.
- A laptop and projector are available for the demo.
- LLM and speech-to-text API access (or a local alternative) is available.
- Demo data uses fake or consented identities.

## 11. Constraints

- The demo must run live in class and fit within 10 minutes.
- Only features that can actually be implemented are designed.
- The app must be web-based and KISS for students.
- No external platform integrations.

## 12. Risks and Mitigation

| ID | Risk | Likelihood | Impact | Mitigation |
|----|------|-----------|--------|------------|
| R1 | Campus network blocks device-to-device traffic or does not separate rooms by subnet | Medium | High | Run the early network spike; fall back to a portable router and the rotating code |
| R2 | A friend relays a live code to someone outside the room | Medium | Medium | Random presence challenges, device binding, anomaly flags, lecturer override; documented as a known limitation |
| R3 | Browser limits (iOS background suspension, microphone or location permissions) | High | Medium | Count only explicit challenge responses; keep audio and location as optional signals |
| R4 | Wi-Fi or internet failure during the demo | Medium | High | Offline queue, local server, a recorded backup demo |
| R5 | LLM or transcription latency and cost | Medium | Medium | Pre-process sample inputs for the demo; cache results |
| R6 | Privacy concerns about attendance and presence logging | Medium | Medium | Collect the minimum, use demo identities, state retention rules in the SRS |
| R7 | Time overrun | Medium | High | Priority order (attendance first); stretch goals clearly marked |

## 13. Acceptance Criteria

The project is accepted when:

1. The attendance feature works live in class: students check in from their phones and the lecturer exports a spreadsheet containing arrival time, verification status, and the signals used.
2. Poll and comment features work live, as far as the demo time allows.
3. Video summary and homework review are shown with at least a pre-processed sample.
4. All deliverables D1 to D11 are present in the repository and consistent with each other.
5. The tools report documents a real test for every tool that was adopted.
6. The 10-minute demo is completed.

## 14. Change Management

Scope changes are proposed in writing (an issue in the repository), assessed for impact on schedule and risk, and accepted by the client before the plan is updated. Features are traded off against the priority order in the SRS.

## 15. Sign-off

| Name | Role | Signature | Date |
|------|------|-----------|------|
| _TBD_ | Client / lecturer | | |
| _TBD_ | Team representative | | |
