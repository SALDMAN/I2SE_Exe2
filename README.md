# I2SE_Exe2
# Vibe Coding & Deep Designing: Web-Based Teaching Tool

> **Course:** Introduction to Software Engineering (OOP, OOD)
> **Assignment:** Ex2 – Vibe Coding & Deep Designing
> **Status:** 🟡 Phase 1 – Problem definition & scope

---

## 1. Overview

Current online tools for teaching are not "doing it" for real in-class learning. This project designs an **end-to-end, simple, LLM-ready web application for teaching in class**: a lightweight smartphone-friendly web app for students, and a control dashboard for the lecturer.

This assignment is a **design challenge**. Implementation is secondary, and reusing existing code and tools is preferred (find-and-compose or vibe coding). The important parts are the problem definition, the design, and the ability to **compose a working web app and demo it live in class (10 minutes)**.

## 2. Problem Statement

Lecturers need better, simpler tools that:

- Remove friction from routine class management (especially **attendance**).
- Let students interact with the lecturer during class, in real time or offline.
- Use LLMs/ML to turn class material (videos, homework) into useful feedback.

Existing tools are fragmented, heavy, and not built with LLM-based analysis in mind.

## 3. Goals

| # | Goal |
|---|------|
| G1 | Provide an automatic, low-friction attendance mechanism (**main feature**). |
| G2 | Enable real-time and offline student feedback. |
| G3 | Let lecturers launch KISS polls in seconds. |
| G4 | Provide ML/LLM summaries of recorded lectures. |
| G5 | Provide deep LLM review of homework, including comparison to peers. |
| G6 | Be demo-ready in class: simple, web-based, no installation. |

## 4. Features & Priority

| Priority | Feature | Description |
|----------|---------|-------------|
| **P0 – Must** | **Automatic attendance** | Students mark attendance from their phone. The lecturer receives an online/offline spreadsheet (CSV/XLSX) with the details. |
| P1 | Live polls | Lecturer creates an on-the-spot poll (KISS) about the current material; students vote; results are shown live. |
| P1 | Comments & suggestions | Students send comments in real time, or offline (queued and synced later). |
| P2 | Lecture video summary | ML/LLM summary of a class recording (e.g. exported from Zoom Meet / Google Classroom), uploaded as a file. |
| P2 | Homework deep review | LLM-based review of submissions, including comparison to other students' work. For students and teachers. |

> The demo must at minimum showcase **P0**, plus as many P1/P2 features as are realistic within 10 minutes.

## 5. Scope

### In scope
- A web-based student app (mobile-first, no install) and a lecturer web dashboard.
- The five features listed above.
- Design artifacts: SoW, SRS, use cases, components, class and sequence diagrams, test strategy.
- A working KISS demo that runs in the classroom.

### Out of scope
- Integration with Moodle, Zoom, or any external LMS/video platform (files are imported manually).
- Integration with personal/private data systems (no sensitive personal data; use demo identities only).
- Production-grade security, scalability, and deployment.
- Native iOS/Android apps.

> **Design rule:** only design what can be implemented.

## 6. Requirements (High Level)

- **Works in class:** must be demonstrable live on real student devices.
- **KISS:** students use a simple smartphone-friendly web app.
- **Offline tolerance:** attendance and comments should survive a poor connection (sync later).
- **Exportable output:** attendance is delivered as a spreadsheet.
- **LLM-ready:** data is stored in a clean, structured form that LLMs can consume.

Detailed requirements will be in the SRS (`docs/03_SRS.md`).

## 7. Proposed Architecture (Draft)

```
 Students (phones, browser / PWA)          Lecturer (browser dashboard)
            │                                         │
            └──────────────►  Web Backend (REST + WebSocket)  ◄────┘
                                  │         │          │
                          Attendance   Polls &      LLM / ML
                          + Export     Comments     Services
                                  │         │          │
                                  └──── Database / File Storage
```

Final components will be defined in `docs/05_Components.md` after the tools report.

## 8. Technology Candidates (To Be Tested)

Each tool must be **tested in real life** before being adopted. Results go to `docs/01_Tools_Report.md`.

| Area | Candidates | Status |
|------|-----------|--------|
| Frontend / PWA | React, Vue, plain HTML + JS | ⬜ |
| Backend | Python (FastAPI / Flask), Node.js (Express) | ⬜ |
| Real-time | WebSocket, Server-Sent Events, Firebase | ⬜ |
| Database | SQLite, Firebase / Supabase | ⬜ |
| Spreadsheet export | CSV, openpyxl, SheetJS | ⬜ |
| LLM | OpenAI / Anthropic / Gemini API, local models | ⬜ |
| Speech-to-text (video summary) | Whisper, cloud STT | ⬜ |
| Hosting / demo access | localhost + tunnel (ngrok), Render, Vercel | ⬜ |

## 9. Deliverables & Roadmap

- [ ] 1. Create repository; define problem and scope (this README)
- [ ] 2. Technical tools report (with real-life tests)
- [ ] 3. Statement of Work (SoW)
- [ ] 4. Software Requirements Specification (SRS)
- [ ] 5. Main use cases (client stories)
- [ ] 6. Main software components
- [ ] 7. Low-level class diagrams and sequence diagrams
- [ ] 8. Testing strategy (covering the classes from step 7)
- [ ] 9. Demo implementation (vibe code vs. find-and-compose)
- [ ] 10. Live demo in class (10 min)
- [ ] 11. Conclusion: short description of results

## 10. Repository Structure (Planned)

```
.
├── README.md
├── docs/
│   ├── 01_Tools_Report.md
│   ├── 02_SoW.md
│   ├── 03_SRS.md
│   ├── 04_Use_Cases.md
│   ├── 05_Components.md
│   ├── 06_Class_and_Sequence_Diagrams.md
│   ├── 07_Test_Strategy.md
│   └── 08_Conclusion.md
├── src/
│   ├── backend/
│   └── frontend/
├── tests/
└── demo/
    └── demo_script.md
```

## 11. Demo Plan (10 Minutes)

| Time | Step |
|------|------|
| 0:00 – 1:00 | Problem and solution overview |
| 1:00 – 4:00 | **Live attendance:** students join from their phones; the lecturer exports the spreadsheet |
| 4:00 – 6:00 | Live poll and real-time comments |
| 6:00 – 8:00 | Lecture video summary (pre-processed sample) |
| 8:00 – 9:30 | Homework deep review and peer comparison |
| 9:30 – 10:00 | Conclusions and lessons learned |

## 12. Getting Started

_To be completed once the implementation exists._

```bash
# Example (placeholder)
git clone <repo-url>
cd <repo>
# install dependencies
# run the app
```

## 14. License & Credits

_TBD. List any reused code, libraries, and AI tools used (vibe coding) here._
