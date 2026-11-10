# Technical Tools Report

**Project:** Smart Classroom Web Application
**Version:** 0.1 (draft, in progress)  |  **Date:** 2026-11-09
**Related:** `TechStack_Attendance.md` (v2.1, §5 "Test before use")

The course asks that each suggested tool is tested in real life. This report records what was tested, how, and the result. Items not yet tested are marked **NOT YET** and are listed in section 3.

**Machine used:** Windows 11 Pro, Python 3.14.5 (64-bit) installed through the Python launcher (`py -3.14`), Git 2.45.1, Node.js present. The `python` command alone is only the Microsoft Store shortcut; use `py -3.14` or the virtual environment.

**How tested:** a throwaway spike (a 90-line FastAPI app, a pytest file, and a live-server script) in a clean virtual environment, not product code. Seven pytest checks and one live run against a real Uvicorn server.

---

## 1. Results

| Tool | What was tested | Result | Status |
|------|-----------------|--------|--------|
| Python 3.14 + `venv` + `pip` | Created a virtual environment and installed all seven packages | Works. Python 3.14.5 is newer than the 3.12+ the stack asks for; every package installed without a build step | **Tested, pass** |
| FastAPI + Uvicorn | Started a real Uvicorn server and served routes | Works; server answered all requests in the live run | **Tested, pass** |
| Jinja2 | Rendered a template through a route | Works. (Our first attempt failed because the loader was passed incorrectly, our mistake, not the tool) | **Tested, pass** |
| SQLite (`sqlite3`, version 3.50.4) | A `UNIQUE(spot_check_id, student_id)` table: a duplicate insert, then two threads inserting the same row at once | The duplicate raised `IntegrityError`; with two simultaneous inserts exactly one succeeded and one was rejected. FR-4.5 can rely on the database rule | **Tested, pass** |
| `secrets.token_urlsafe(32)` + cookie | Issued a cookie; checked the flags and decoded the token length | Cookie has `HttpOnly` and `Secure`; token decodes to 256 bits (limit is 128, NFR-5) | **Tested, pass** |
| Distance calculation (haversine with `math`) | Same point; one degree of latitude; a point 0.0009° north | 0 m; about 111 km; about 100 m. Accurate enough for a classroom radius | **Tested, pass** |
| SSE (`sse-starlette`) | Real server; one client, then 200 clients connected; one `POST /publish` pushed an event to all | 1 client: 5 ms. 200 clients: all 200 received it, median 47 ms, slowest 88 ms. The goal is under 2 seconds (NFR-3). The single-process design handles this on a laptop | **Tested, pass** |
| File upload (`python-multipart`) | Uploaded a 10 MB file and downloaded it back; uploaded a file named `../evil.txt` | Sizes matched exactly. The path trick was neutralized by taking only the file name. Streaming in 1 MB chunks works | **Tested, pass** (10 MB only) |
| pytest + `TestClient` + `httpx` | Ran seven tests, including a threaded one | 7 passed in under a second | **Tested, pass** |
| Cloudflare quick tunnel | `cloudflared` 2026.10.7 installed with `winget`. The tunnel run itself was deferred by the team on 2026-11-09 | Tested 2026-11-10: the tunnel gave a public HTTPS address and served the spike's `/phone` page from the laptop to a phone, status 200 | **Tested, pass** |
| Browser Geolocation API | — | Tested 2026-11-10 on one Android 10 phone (Chrome 156) over the tunnel's HTTPS: the permission prompt worked and two readings came back, 6.9 s for the first and 1.2 s for the second, both with 20 m accuracy. Also tested on an iPhone (iOS 18.7, Safari): one reading in 0.9 s with 9 m accuracy. Not tested in the real classroom (raw coordinates are not kept in this report) | **Pass on Android and iPhone; room not yet** |
| Safari / Chrome SSE on phones | — | Tested 2026-11-10 on one Android 10 phone (Chrome 156) over the tunnel: the page showed SSE connected (reported by the person testing). The iPhone (Safari) was also reported as working. Not confirmed: locked screen and reconnect | **Pass on Android and iPhone; locked-screen reconnect not confirmed** |
| AI service (video summary), Gemini API | A real lecture recording (`videoplayback.mp4`, 119.8 MB, 5,078 s = 84.6 minutes, a data structures lecture on asymptotic notation) through the Files API (resumable upload, wait for ACTIVE, then `generateContent` with `gemini-3.5-flash-lite`); the uploaded file was deleted afterwards. Run on 2026-11-10 | Upload 42 s; file processing 118 s; summary generation 42.9 s; **total 206 s**. 462,164 input tokens (about 91 per second of video), 184 output tokens. The summary had a title, a short paragraph and six topic bullets (Big-O, Omega, Theta, proofs, limits, iterative running times) that fit the lecture's subject; the content was not checked line by line against the video | **Tested, pass** (one run) |
| AI service (PDF review with dataset), Gemini API | One student PDF plus three made-up dataset PDFs (sent as inline base64, neutral names `doc-1` to `doc-3`; the dataset PDFs contained author names inside their text on purpose), prompt asking for feedback, an anonymous comparison and one anonymous example excerpt. Run on 2026-11-10 with the real API key | `gemini-3.5-flash-lite`: status 200 in 2.6 s, 2,271 input tokens, 276 output tokens. The review gave correct feedback (missing complexity analysis, only one test case), an anonymous comparison ("weaker than average", memoized and iterative solutions by "another student"), and a code excerpt from the best solution. **No author name, student number or file name leaked.** One factual slip: it said peers gave "complete test suites", which was not true of all of them. `gemini-2.5-flash` returned 404 ("no longer available to new users"); `gemini-3.8-flash` timed out after 120 s with no answer | **Tested, pass** (basic quality, one run) |
| MP4 upload at demo size | (a) 119.8 MB MP4 uploaded to the Gemini Files API through our own `httpx` code. (b) The same file uploaded to our own spike server (FastAPI `UploadFile`, saved in 1 MB chunks), then downloaded back through `FileResponse`; sizes and SHA-256 hashes of source, saved copy and downloaded copy compared | (a) Worked in 42 s. (b) Upload 0.6 s, download 0.5 s, **sizes equal and hashes identical**. These times are on the same laptop (loopback), so real Wi-Fi will be slower. Not yet tested over the internet or on phone data | **Tested, pass** (local) |
| Mermaid diagrams | All 12 diagrams in `Components_Attendance.md`, `Design_Attendance.md` and `UseCases_Attendance.md` parsed with the Mermaid library itself (version 12.3.0, in a clean Node folder) | All 12 parse with no syntax error (2 component flowcharts, 1 use-case flowchart, 2 class diagrams, 7 sequence diagrams). This checks syntax only; how they look on GitHub and in your Obsidian version was not checked | **Tested, pass** (syntax) |

## 2. Findings that change the plan

1. **Python 3.14 works with the whole stack.** The handoff said Python was not installed; that was out of date. Pin the versions found below in `requirements.txt` so every team member gets the same ones.
2. **Installed versions (all worked together):** `fastapi 0.162.2`, `starlette 1.10.0`, `uvicorn 0.54.0`, `jinja2 3.1.6`, `sse-starlette 3.7.0`, `python-multipart 0.0.33`, `httpx 0.28.1`, `pytest 9.2.0`, `pydantic 2.14.3`.
3. **Deprecation warning about `httpx`:** this Starlette version prints "Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead." Tests still pass. Before pinning, check whether to use `httpx2` for the tests; the AI client and load test can still use `httpx`. The tech stack should be updated once this is decided.
4. **Windows quirk:** `python` on its own is the Store shortcut. Team instructions must say `py -3.14 -m venv .venv`, then use the venv's Python.
5. **Load numbers so far are encouraging:** 200 SSE clients received a push in under 100 ms. The full 200-student load test (answers in, not only pushes out) is still WP11.

6. **The AI service works for the PDF review (T5).** Gemini API with `gemini-3.5-flash-lite` took 2.6 s for 4 small PDFs and kept the dataset anonymous even when authors' names were written inside the PDFs. This is one run on made-up text; it shows the idea works, not that reviews are always right.
7. **Model names move.** Within one session, `gemini-2.5-flash` was already closed to new users and `gemini-3.8-flash` timed out. The model name must be a setting, not hard-coded, and the job runner's timeout and failed state (NFR-8) are needed in practice, not only in theory.
8. **The anonymity rule must be in the prompt, and the names in PDFs are the real risk.** File names are neutral, but names can appear inside the content. The prompt must forbid revealing them, and WP10 needs a test that scans the review for known names (SRS FR-10.10).
10. **The video summary works (T4), but it is slow for full lectures.** An 85-minute lecture took 206 s end to end, of which 118 s was the provider processing the upload. A 3-hour lecture could take far longer than the 5-minute timeout in NFR-8, so the default summary timeout is raised to 10 minutes (SRS NFR-8 updated). The job design (background job, pending / done / failed) is confirmed necessary: a request could not wait this long.
11. **Video cost in tokens is large:** about 462,000 input tokens for 85 minutes. This ran on the key's free tier without an error, but a few long videos in a row could hit a daily or per-minute token limit. The demo uses a short MP4 (SRS NFR-9) and the fake client as fallback.
12. **A 120 MB video fits our 200 MB limit.** An 85-minute lecture of this quality is about 120 MB, so the limit holds for normal lectures. Longer or better-quality recordings may exceed it.
13. **Free-tier limits remain unconfirmed.** The call succeeded on the key as it is; no rate-limit errors appeared in the few calls made.

## 3. Still to test

| # | Test | What is needed | Who |
|---|------|----------------|-----|
| T1 | Cloudflare quick tunnel (done 2026-11-10, see above) | `cloudflared` is installed. Start the spike server, run `cloudflared tunnel --url http://localhost:8000`, open `/phone` on the printed address. Claude Code blocks starting a public tunnel without an explicit permission rule, so a person runs it or adds the rule | Anyone |
| T2 | Geolocation on Android Chrome and iPhone Safari, in the real classroom | A page that reads the position and shows it (through the tunnel); two phones; the room | Needs the team, in the room |
| T3 | SSE on phones | The same spike page, opened on both phones; push an event | With T2 |
| T4 | AI service, video | Done 2026-11-10 with Gemini and an 85-minute recording (see section 1). Repeat with a short 2 to 3 minute clip for the demo | Done |
| T5 | AI service, PDF review | Done 2026-11-10 with Gemini (see section 1). A second run with a real homework assignment is still worth doing | Done |
| T6 | Upload at demo size | The real short MP4 | Anyone |
| T7 | Mermaid rendering | Syntax checked (all 12 pass). Still open: look at the files once in Obsidian and on GitHub after pushing | Anyone |
| T8 | Fake AI client | Build it in WP9 and run the whole flow offline | Developer |

## 4. Where the spike code is
The spike is throwaway and lives in a temporary scratch folder, not in the repository. If the team wants it kept as evidence for the report, copy `spike_app.py`, `test_tools.py`, and `run_live.py` into the repository under `spikes/` before the scratch folder is cleaned.

## 5. AI service options (desk research, 2026-11-09, nothing tested yet)

Figures come from the providers' documentation pages unless marked. Free-tier limits change often and some sources disagree, so the numbers must be checked in the provider's console before use.

| Option | Video summary | Homework PDF review | Free tier | Notes |
|--------|---------------|---------------------|-----------|-------|
| **A. Gemini API (one provider for both)** | Takes an MP4 directly through its Files API. Docs: 2 GB per file on the free tier, about 100 tokens per second of video at low resolution, up to 3 hours at low resolution on 1M-token models | PDF input expected to be supported; **to be tested** | The docs point to AI Studio for the live limits; the numbers could not be confirmed | One key, one client, simplest fit for "basic". Check whether free-tier data may be used by the provider: relevant to student homework (use sample PDFs in the demo) |
| **B. Groq Whisper + a text model** | Whisper accepts MP4 files; docs: 25 MB on the free tier, 100 MB paid, first audio track only. Two steps: transcript, then a text model writes the summary. `ffmpeg` can shrink a larger file to FLAC | A text model on text pulled out of the PDFs, or a PDF-capable model | Third-party pages say about 20 requests per minute and 2,000 per day for Whisper; not confirmed by the official page | Two calls instead of one. `ffmpeg` is not installed here |
| **C. Claude API** | Does not take video or audio directly, so it would need option B's transcript first | Takes PDFs directly: docs say up to 128 MB per request, 600 pages (100 pages on smaller context windows), no separate PDF fee, about 1,500 to 3,000 tokens per page | No free credits found in the search | Paid by tokens; a demo with a few short PDFs should cost little but is not free |
| **D. Fake client only** | Canned text | Canned text | Free | Always available; already planned as the fallback |

**Suggested path:** try option A first, because one provider and one key cover both features. If the free tier is too limited or the PDF review is poor, use B for the video and C or A for the PDFs. The `AiClient` interface stays the same in every case: `summarize_video(path)` and `review_homework(pdf_path, dataset)` hide how many calls happen inside.

**Decision needed from the team:** which provider to try first, and who creates the account and API key. Keys go in an environment variable and never in the repository.

## 6. Conclusion so far
Every tool that can be tested on one computer passed. The remaining risk is in the three real-world items: phone location accuracy in the room (T2), the HTTPS tunnel (T1), and the AI service (T4, T5). Those should be done before the demo app is built around them.
