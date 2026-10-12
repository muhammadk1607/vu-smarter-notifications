# Research: What a logged-in VU LMS student session exposes

Resolves issue #2 (sub-issue of the map #1). Read-only reconnaissance of
`https://vulms.vu.edu.pk/` performed with one student account the owner
authorised for research, on **2026-10-12**.

Method: a plain HTTP client (`curl`) to probe the login form, then a real
browser (Playwright/Chromium, headless) to log in and navigate. No item was
ever submitted, booked, selected or posted; only the login form was POSTed.
The session was signed out at the end. Credentials were passed via environment
variables only and appear nowhere in this file, in git, or in the issue. The
student's name/ID, instructor names and enrolment specifics are redacted; raw
pages and screenshots were kept out of the repo.

Primary sources are the live pages and responses themselves; each claim cites
the page/endpoint it came from.

---

## Bottom line

- **A real browser is required**, not a plain HTTP client. Login is gated by
  **Google reCAPTCHA v3** (invisible, score-based). A `curl` POST with a blank
  token is rejected as "Incorrect username or password". Headless Chromium
  logs in fine (the v3 score threshold is lenient — login succeeded even with a
  `HeadlessChrome` user-agent).
- The LMS is **ASP.NET Web Forms** (IIS 10, .NET 4.0). Almost all navigation is
  `__doPostBack` against a single form with `__VIEWSTATE` / `__EVENTVALIDATION`,
  not plain GET URLs. A scraper must either drive a browser or faithfully replay
  the postback + ViewState dance.
- There is a **single-active-session limit**: a second concurrent login for the
  same student bumps the other to `MultiLoginMessage.aspx` ("Multiple Login
  Error"). This is the single most important operational constraint for a
  notifier (see Implications).
- **Personal Deadlines** (assignments, quizzes, GDBs, results) and their
  **submission/attempt status** are all visible on per-course list pages and are
  aggregated on a "To Do Calendar". **Handled detection is feasible** from the
  Submit / Status columns.
- **Institutional Windows**: *Course Selection* lives inside the LMS;
  *Datesheet* ("Make your Date Sheet") is an **external** site; *announcements*
  live on the Notice Board and per-course announcement lists.
- A bespoke client-side **anti-scraping framework** (`js/security-protection.js`)
  runs in the browser (honeypots, DevTools/proxy/extension detection, fetch/XHR
  interception, auto-logout on "suspicious activity", 30-min idle timeout). It is
  entirely client-side, so a non-browser HTTP client is not subject to it — but
  reCAPTCHA v3 at login is the hard blocker that forces a browser anyway.

---

## 1. Login mechanism

Login page is served at `/` (which is `LMS_LP.aspx`).
Source: GET `https://vulms.vu.edu.pk/` and the post-logout redirect target.

- **Server stack** (response headers): `Server: Microsoft-IIS/10.0`,
  `X-AspNet-Version: 4.0.30319`, `X-Powered-By: ASP.NET` + `ARR/3.0`.
- **Form**: one `<form method="post" action="./" id="ctl00">` (classic Web Forms).
- **Fields** (from the login HTML):
  - `txtStudentID` — text, `maxlength=12` (the student roll number).
  - `txtPassword` — password, `maxlength=15`.
  - `ibtnLogin` — submit button, value "Sign In", `onclick="return ValidateFields();"`.
  - Hidden: `__VIEWSTATE`, `__VIEWSTATEGENERATOR` (value `4C74136D`),
    `__EVENTVALIDATION`, `g-recaptcha-response`, and `action=beta_LMS`.
- **CAPTCHA**: Google **reCAPTCHA v3** (invisible). The page loads
  `https://www.google.com/recaptcha/api.js?render=<sitekey>` (public site key
  `6LdSLNUUAAAAAAcFN7NNNDEH-DdQ5GyZQDlhTYjn`). On submit the script fills the
  hidden `g-recaptcha-response` with a token the server scores. There is **no
  visible checkbox or image challenge** (that is v2); v3 runs silently. Observed
  calls: `recaptcha/api2/anchor`, `.../reload`, `.../clr`.
- **OTP / 2FA**: none observed. Username + password + reCAPTCHA token only.
- **IP check**: the login form carries a hidden `lblInvalidIp` feedback element,
  suggesting the backend may reject logins from disallowed IPs. Not triggered
  during this research (logins from the research host succeeded), but worth
  flagging — a server-side IP allow/deny could silently block a cloud host.
- **Success**: POST redirects (GET) to `Home.aspx?id=<GUID>`, title
  "VU - Learning Management System". The `id` GUID appears to be a per-login
  identifier carried in the landing URL.
- **Failure**: the login page re-renders with
  `<span id="lblError">Incorrect username or password. Please try again.</span>`.
  A missing/invalid reCAPTCHA token produces this same generic message (no
  distinct "captcha failed" text).

**`curl` login result**: POSTing the correct ViewState/EventValidation +
credentials with an empty `g-recaptcha-response` returned HTTP 200 and
re-rendered the login page with the "Incorrect username or password" error and a
fresh `ASP.NET_SessionId`. Conclusion: the reCAPTCHA v3 token is mandatory, so a
plain HTTP login is not viable.

## 2. Cookies, session lifetime, concurrent-session limit

- **Cookies** (Set-Cookie on `/`):
  - `ASP.NET_SessionId` — session cookie, `HttpOnly; SameSite=Lax; path=/`.
  - `ARRAffinity` — Azure/ARR load-balancer stickiness cookie,
    `Domain=.vulms.vu.edu.pk`. Needed so the session stays pinned to one backend.
- **Session regenerates on login** (new `ASP.NET_SessionId` after the login POST).
- **Idle lifetime**: the anti-scrape script configures `sessionTimeout: 30 min`
  of inactivity; classic ASP.NET session timeout is typically the same order.
  Treat ~30 min idle as the working assumption.
- **Concurrent-session limit = 1**: during testing, when two of the research
  sessions overlapped, navigating landed on
  `https://vulms.vu.edu.pk/MultiLoginMessage.aspx` (title "Multiple Login Error").
  VU enforces one active session per student; a new login invalidates/conflicts
  with the previous one. **This is a hard constraint for a notifier.**
- **Logout**: `__doPostBack('ctl00$lnkLogout', '')` or GET
  `https://vulms.vu.edu.pk/SignOut.aspx` (title "Sign Out"). Verified: after
  SignOut, requesting `Home.aspx` redirects to the login page `LMS_LP.aspx`, so
  the session is dead.

## 3. Navigation model (matters for implementation)

The top navigation and in-page actions are ASP.NET postbacks, e.g.
`__doPostBack('ctl00$lbtnActivityCalendar','')`,
`...lbtnGradeBook`, `...lbtnCourseSelection`, `...lbtnStudentServices`,
`...lbtnLectureSchedule`, `...lbtnHome`, `...lnkLogout`.
A few pages are plain GET (`/NoticeBoard/NoticeBoard.aspx`,
`/Settings/MyLogins.aspx`, `/Profile/StudentProfile.aspx`,
`/Settings/ChangePassword.aspx`).

The Home course grid (`gvCourseList`) uses per-course **ImageButtons** that post
back, e.g. `ctl00$MainContent$gvCourseList$ctl00$ibtnAssignments`,
`...$ibtnQuizzes`, `...$ibtnGDB`, `...$ibtnnnouncements` (Announcements),
`...$ibtnCourseHome`.

Implication: driving this in Node is far simpler with Playwright (click the real
controls) than reconstructing ViewState postbacks by hand. Note that
`page.evaluate(() => __doPostBack(...))` fails on the AJAX/UpdatePanel pages
(MS Ajax `__doPostBack` touches `arguments.callee`, which throws under
Playwright's strict-mode eval) — **click the anchors/buttons instead**.

## 4. Personal Deadlines — where they live, dates & status

### 4a. Home course grid (overview / "new item" signal)
`Home.aspx` lists the student's enrolled courses (7 courses, Fall 2026, redacted).
Each course row shows activity icons with a **red count badge**
(`badge m-badge--danger`) indicating pending/new items, e.g. a "1" on the
Assignment and Quiz icons. Elements: `ibtnAssignments`/`lblAssignment`,
`ibtnQuizzes`/`lblQuiz`, `ibtnGDB`/`lblGDB`, `ibtnnnouncements`/
`lblAnnouncement`, `imgActivitySession`/`lblActivity`. This badge is a cheap
"something changed" signal per course.

### 4b. "To Do Calendar" (aggregated deadlines)
`ActivityCalendar/ActivityCalendar.aspx` (title "To Do Calendar") is a month/week
calendar (header `Sun..Sat`, cells tagged `data-date="YYYY-MM-DD"`). Below it are
three lists rendered into `divToDoAssignmentsGV`, `divToDoQuizzesGV`,
`divToDoGDBGV` (Assignments / Quizzes / Graded Discussion Board), plus a
"Pending Fee" item and a "Pending" status word. These list divs are populated
dynamically and were empty at the time (nothing currently due). This is the best
*single* consolidated view of upcoming Personal Deadlines, but per-course pages
(below) are the authoritative source and expose status.

### 4c. Per-course activity list pages (authoritative, with status)
Reached from the Home grid ImageButtons. Columns observed:

- **Assignments** — `/Assignments/StudentAssignmentListView.aspx` (title
  "Assignment"). Columns: *(title) | Due Date | Total Marks | Submit | Result |
  Discuss*. The **Submit** column reflects submission state/action; **Result**
  shows marks once graded.
- **Quizzes** — `/Quiz/QuizList.aspx` (title "Quiz List"). Columns:
  *(title) | Start Date | End Date | Total Marks | Open/Close | Status | Result*.
  **Status** reflects attempt state; **Open/Close** whether the quiz window is
  live. A banner warns that *taking* a quiz needs the "VU Quiz Firewall" Chrome
  extension — this is only for attempting a quiz; **reading the list does not
  start an attempt** and needs no extension.
- **GDB** — `/GDB/Default.aspx` (title "Graded Discussion Board"). Columns:
  *Total Marks | Start Date | End Date | GDB Status | Submit Status | View GDB |
  Result*. **GDB Status** = open/closed; **Submit Status** = whether the student
  has posted. (For the sampled course: "No GDB exists for this course.")
- **MDB (Moderated Discussion Board)**: **no MDB control exists** in the current
  LMS course grid — only GDB. MDB appears to have been retired; the notifier
  should treat GDB as the discussion deadline and not rely on an MDB page.
- Dates render in formats like `16 October 2026` and `DD/MM/YYYY` / time
  `HH:MM AM/PM` depending on page.

### 4d. Results (result release)
`GradeBook/GradeBook.aspx` (title "Grade Book") holds declared results. Sections
seen: "Student Grade Book" and "Midterm Result (FALL 2026)". Table columns:
*Semester, Course Code, Title, Credit Hours, Score, Grade Point, Grade Points,
Equivalent Percentage, Percentile, Grade*. Where no result is declared it shows
"No Record". Per-activity "Result" columns (above) also surface marks once
released. So "result release" is observable both in the Grade Book and as the
Result column flipping to a value on each activity list.

## 5. Handled detection

Yes — submission/attempt status is visible without starting anything:
- **Assignment** → the **Submit** column on `StudentAssignmentListView.aspx`
  (submitted vs. still-submittable) and the **Result** column once marked.
- **Quiz** → the **Status** column on `QuizList.aspx` (attempted vs not) plus
  **Open/Close**.
- **GDB** → the **Submit Status** column on `GDB/Default.aspx`.
- **Datesheet booked** → not visible inside the LMS (booking is on the external
  datesheet site; see below), so "booked" can't be confirmed from the LMS and
  will need a different signal.
- The Home-grid red count badges give a coarse per-course "pending" count.

## 6. Institutional Windows inside (and adjacent to) the LMS

- **Course Selection** — inside the LMS:
  `/StudentServices/CourseSelection/CourseSelection.aspx` (title "Course
  Selection"). Shows "Course Enrollment (Fall 2026)" with a grid
  (*Sr.No | Course | Credits | Action*) and summary fields (Credits Earned,
  Current Semester No, Specialization, Credits Exempted, Maximum Credits Limit).
  When open, the Action column offers add/select controls. **Not actioned.**
- **Datesheet** — **external**, linked from Student Services as
  "Make your Date Sheet" → `http://datesheet.vu.edu.pk/`. Both *booking* and
  *availability* of the midterm/final datesheet are handled on that separate
  domain, not inside the LMS. A notifier covering datesheet windows must treat
  `datesheet.vu.edu.pk` as its own source (separate recon).
- **Announcements / Notices**:
  - Site-wide: `/NoticeBoard/NoticeBoard.aspx` (title "Notice Board") — dated
    notices (e.g. "16 October 2026"), with "Due"/"Open" wording; this is where
    institutional announcements (including, typically, datesheet-availability and
    course-selection-window notices) are posted.
  - Per-course: the Announcements ImageButton / `hylnkAnnouncements` on the Home
    grid, with its own red "new" badge.
- **Student Services hub** — `/StudentServices/StudentServices.aspx` groups
  Academics / Accounts / Examinations / Registrar links, including Lecture
  Schedule, Grade Book, Account Book, e-ID Card, and many Examinations/Registrar
  request forms (rechecking, rescheduling, transcripts, degree, freeze, etc.).
  These are mostly request actions, out of scope for read-only notifications.

## 7. XHR / JSON endpoints

The app is mostly full-page postbacks, not a JSON API. The only first-party
XHR observed is an ASP.NET page method:
`POST /LMS_LP.aspx/SetRandomQuote` (JSON, cosmetic — rotates a quote on the login
screen). The To-Do calendar lists populate client-side but no clean public JSON
feed for deadlines was exposed; the practical data source is the rendered
list-page HTML. Third-party XHR seen: Google Analytics (`google-analytics.com`,
`analytics.google.com`) and reCAPTCHA. There is also a referenced violation
endpoint `POST /api/csp-report` used by the security script.

## 8. Plain HTTP client vs. real browser

**A real browser is required.** Reasons, in order of importance:
1. **reCAPTCHA v3 at login** — a blank/invalid token yields "Incorrect username
   or password"; only a browser executing `api.js` produces an accepted token.
2. **ASP.NET ViewState postbacks** — navigation is stateful form posts, not
   GET URLs; a browser handles these natively.
Headless Chromium (Playwright) is sufficient — the v3 score threshold accepted a
headless, `HeadlessChrome`-UA session. (Masking `navigator.webdriver` and setting
a normal desktop UA is prudent but login succeeded regardless.)

## 9. Anti-bot / anti-scraping measures

- **reCAPTCHA v3** on login (section 1).
- **`js/security-protection.js`** (~78 KB) — a bespoke client-side hardening
  framework loaded on the login page. Config flags include DevTools protection,
  DOM monitoring, network monitoring, memory protection/wiping, Trusted Types,
  CSP enforcement, DOM sanitization/encryption, shadow-DOM & form & clipboard
  protection, input masking, data obfuscation with **decoy/honeypot** form fields
  (`fake_password_1`, `fake_email_1`, `fake_token_1` — accessing them flags
  "suspicious"), `enableAutoLogoutOnSuspiciousActivity` with
  `suspiciousActivityThreshold: 5`, and `sessionTimeout: 30 min`. Named modules
  include `DevToolsDetection`, `ProxyDetection`, `ExtensionDetector`,
  `SuspiciousActivityMonitor`, `NetworkProtection` (wraps `window.fetch` and
  `XMLHttpRequest.open/send`, injecting `X-Requested-With`), `TamperDetection`,
  `IntegrityCheck`, `MemoryWiper`.
- **Crucial nuance**: all of the above is *client-side JavaScript*. A non-browser
  HTTP client that never runs it is unaffected by the honeypots, DevTools and
  DOM checks — but can't get past reCAPTCHA v3. A real browser (which we need for
  reCAPTCHA) *does* run it, so automation must: never touch the decoy fields,
  select real controls by their stable ids/names, avoid opening DevTools, keep
  actions modest (the suspicious-activity counter auto-logs-out at 5), and expect
  a ~30-min idle cap. The framework did not block a careful headless Playwright
  session in this research.
- **Single-session enforcement** (`MultiLoginMessage.aspx`) doubles as an
  anti-abuse measure.
- A possible **server-side IP check** (`lblInvalidIp`).

## 10. Implications for the notifier (for the planning tickets, not decided here)

- **Browser required**: budget for Playwright/Chromium on the chosen free host;
  a pure `fetch`/cheerio scraper will not clear login. Confirm the host's egress
  IP is not blocked by the `lblInvalidIp` server check before committing.
- **Single-session limit is the dominant constraint**: the notifier logging in
  will kick the Member out of their own LMS session (and vice-versa). Options to
  weigh: run checks at low-traffic hours; keep sessions very short; coordinate/
  warn Members; or accept occasional mutual logouts. This feeds directly into the
  "polite checking" and check-frequency decisions in the map.
- **Per-run cost**: each check = one full browser login (reCAPTCHA + ViewState).
  Stagger Members and cache within the ~30-min session window; re-login is
  expensive and adds load.
- **Data sources to scrape** per Member: Home grid badges (cheap change signal),
  then per-course `StudentAssignmentListView.aspx`, `QuizList.aspx`,
  `GDB/Default.aspx` for dates + Submit/Status (Handled), `GradeBook.aspx` for
  results, `NoticeBoard.aspx` + per-course announcements, and
  `CourseSelection.aspx` for the course-selection window. **Datesheet needs a
  separate source** (`datesheet.vu.edu.pk`), out of the LMS.
- **Breakage detection**: pin to the stable control ids/page URLs listed here;
  alert if login stops returning `Home.aspx?id=...`, if `MultiLoginMessage.aspx`
  appears, or if expected columns (Due Date / Status / Submit Status) vanish.

## 11. Caveats / not verified

- At the time of this read the test account had no currently-open assignment or
  quiz with a live deadline, so exact *rendered values* of the Submit/Status
  cells (e.g. the literal "Submitted" vs "Not Submitted" strings) were inferred
  from the column headers and page structure rather than seen populated. The
  columns exist and are the right place; confirm exact status strings against a
  course that has an open item.
- "Pending Fee" appears on the To-Do calendar — fees may also be a dated
  obligation worth considering, though out of the current glossary.
- The external `datesheet.vu.edu.pk` and `qb.vu.edu.pk` (Overseas Exam) sites
  were not reconned (out of scope for this LMS ticket).
- Tested with one account from one IP on one date; VU can change markup, the
  reCAPTCHA config, or the IP policy at any time.
