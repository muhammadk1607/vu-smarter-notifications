# Research: What does `datesheet.vu.edu.pk` expose to a logged-in student?

Resolves issue #12 (sub-issue of the map #1). Read-only reconnaissance of
`http://datesheet.vu.edu.pk/` performed on **2026-10-12** with one student
account the owner authorised for research.

Method: a plain HTTP client (`curl`) for the public landing page and the login
flow, then a real browser (Playwright/Chromium, headless) to log in and observe
the logged-in view. **Nothing was ever booked, selected, confirmed or
submitted**; only the login form was POSTed. The session was logged out at the
end (`btnLogout`) and cookies cleared. Credentials were passed via environment
variables only and appear nowhere in this file, in git, or in the issue. The
student's ID and name are redacted. Raw pages were kept out of the repo.

Primary sources are the live pages and HTTP responses themselves.

---

## Bottom line

- **Login reuses the VULMS ID + password.** The datesheet site has no separate
  account. Its login page is literally titled **"Login : VULMS"** and carries a
  hidden `action=datesheet` field — it is VU's shared Single-Sign-In page, the
  same style as the LMS login.
- **There is a CAPTCHA: Google reCAPTCHA v3 (invisible).** Same mechanism as the
  LMS, different site key (`6Ldw1zUdAAAAALYsOkd56ujcBk9QP_9VsQQhgSDT`, action
  `datesheet`). A plain HTTP client **cannot** log in; a real browser is
  required. Headless Chromium cleared it on the first try (v3 score threshold is
  lenient, as on the LMS).
- **The public landing page needs no login** and is plain ASP.NET WebForms HTML
  (`curl`-friendly, ~10 KB). It carries the current term, exam type, and the
  window's closing date(s) / late-fee stages. A plain HTTP client works for this.
- **Outside a booking window (now), a logged-in student lands straight on their
  "Exam Entrance Slip"** (`ExamEnteranceSlip.aspx`) — the finalised slip for the
  most recent term. The booking UI (city/centre/slot pickers) is **not reachable**
  when no window is open.
- **Booking status is readable without changing anything.** The entrance slip is
  a read-only page (its only actions are *Logout* and *Send email to myself*). A
  populated slip (exam password + per-course dates/centre) means **booked**; an
  empty slip (blank password/name/dates) means **not booked** for that term. This
  is the **Handled** signal for datesheet windows.
- **One-session limit: not confirmed to be shared with the LMS.** The datesheet
  app keeps its **own** `ASP.NET_SessionId` on its own domain; logging in to
  datesheet alone produced no "Multiple Login" error. The shared SSO login page
  means the *credential check* is centralised, but the LMS's single-active-session
  enforcement was not observed to extend to datesheet. Treat as possibly shared
  and keep sessions short (see Implications).
- **TLS:** `https://datesheet.vu.edu.pk/` has an **incomplete certificate chain**
  (`unable to get local issuer certificate`, `ssl_verify_result=20`), same as the
  rest of `vu.edu.pk`. Plain **`http://` works** and is what the site's own links
  use; for the browser step, Playwright's `ignoreHTTPSErrors` (or supplying the
  intermediate CA) handles it. We did not disable verification on the HTTP client;
  we simply used `http://`.

---

## 1. Login mechanism

- **Landing page** `http://datesheet.vu.edu.pk/` (served as `login.aspx`) is a
  WebForms page (`<form id="frmlogin" action="./">`) whose only control is a
  **`btnLogin` "Login"** submit button — there are *no* credential fields here.
  Server headers: `X-Powered-By: ASP.NET`, `Set-Cookie: ASP.NET_SessionId=...;
  HttpOnly; SameSite=Lax`.
- POSTing `btnLogin` (with the page's `__VIEWSTATE` / `__EVENTVALIDATION`)
  returns **302 → `/authenticateStudent.aspx`**.
- **`authenticateStudent.aspx`** (title **"Login : VULMS"**,
  `<form id="form1" action="./authenticateStudent.aspx" autocomplete="off">`) is
  the real login form:
  - `txtSId` — text, `maxlength=11`, placeholder "Student Id" (the VULMS roll no.)
  - `txtP` — password, `maxlength=15`, placeholder "Password"
  - `btnLogin` — "Sign In" (`onclick="return onbtnClick(event);
    __doPostBack('btnLogin','')"`)
  - hidden `g-recaptcha-response`, and hidden `action=datesheet`
  - loads `https://www.google.com/recaptcha/api.js?render=6Ldw1zUdAAAAALYsOkd56ujcBk9QP_9VsQQhgSDT`
    and runs `grecaptcha.execute('6Ldw1zUdAAAAALYsOkd56ujcBk9QP_9VsQQhgSDT',
    {action:'datesheet'})` to fill the hidden token (invisible reCAPTCHA **v3**).
    A commented-out block uses action `'Single_SignIn'` — confirming this is VU's
    generic SSO login page, parameterised per app.
- **No OTP / 2FA** observed. Student ID + password + reCAPTCHA token only. The
  instructions block on the landing page states plainly: *"Logon using your VULMS
  ID and Password (… password problems → password@vu.edu.pk)"* — so credentials
  **are** the LMS credentials.
- **Success** (observed): POST of the filled form redirected to
  **`ExamEnteranceSlip.aspx`** (title "Exam Enterance Slip"). No error label.

**Does a plain HTTP client work?** For the **public landing page, yes** (window
info is readable with `curl`). For **login / anything behind it, no** — reCAPTCHA
v3 needs a browser to produce an accepted token, exactly as on the LMS. Headless
Playwright/Chromium is sufficient.

## 2. What is visible outside a booking window

Today is well past the Summer 2026 final signup close (Oct 02, 2026), so this is
an out-of-window read.

### 2a. Public landing page (no login)
Shows, in plain spans:
- `lblDisplayMsg` = "Make your Date Sheet – NOW"
- `lblSemester` = "Summer 2026", `lblExamType` = "Finalterm"
- A closing-date panel `pnlOnlyClosingDate` →
  **"Last Date of Signup is `lblSignupLastDate` = Oct 02, 2026"**
- Static instructions (choose city → choose centre (once chosen, cannot change) →
  select date/start-time per course → Confirm → get Exam Password + print slip),
  and contacts `datesheet@vu.edu.pk`, `password@vu.edu.pk`.
- A link to the VUTES exam-software demo (`http://qb.vu.edu.pk/Vutes_demo/...`).

**Format of the window dates / late-fee stages (important):** the page contains a
late-fee panel (CSS classes `late-fee`, `late-fee__title`, `late-fee__list`,
`late-fee__val`) that renders either:
- a **single** closing date (`pnlOnlyClosingDate`, shown now: "Last Date of
  Signup is Oct 02, 2026"), **or**
- a **tiered late-fee list** when the window has stages. The decoded `__VIEWSTATE`
  of the current page still carries the tier data even though that panel is hidden:
  a free-until date (**"Oct 02, 2026"**) plus amounts **`500` / `20`** and
  **`1000` / `25`** — i.e. Rs 500 / $20 (late fee) and Rs 1000 / $25 (double late
  fee), matching the free → late → double-late pattern documented for #3. Dates
  are English prose like **"Oct 02, 2026"**. So the landing page **is** a usable
  source for the opening/closing and late-fee stages, but the banner **persists
  after the window closes** (it still showed the Oct 02 date on Oct 12), so "open"
  must be inferred by comparing today against the signup date — consistent with
  the #3 finding.

### 2b. Logged-in view (no window open)
Login lands directly on **`ExamEnteranceSlip.aspx`** — "Exam Entrance Slip
(Summer 2026 - Finalterm)". There is **no booking workflow reachable** while no
window is open; the student only sees their slip for the last term. Fields on the
slip: `lblSemester`, `lblExamType`, `lblStudyProgram`, `lblStudentID`
(= the student's VULMS ID), `lblName`, `lblPassword` (the Exam Password), an
`lblMsg`, a *Send email to myself* checkbox (`chkSendMail` + `btnSendmail`), and
a `btnLogout`. The slip body sits in AJAX `UpdatePanel`s.

For the test account the slip rendered with **blank** name / study-program /
password and no exam-date rows — i.e. the **"not booked for this term"** state
(this student had no Summer 2026 final booking). A booked student's slip would
instead carry the exam password and the per-course exam dates/centre (that is the
printable slip the instructions describe).

## 3. Reading booking status without changing anything (the Handled signal)

**Yes — feasible and read-only.** After login the entrance slip tells you whether
the student has booked:
- **Booked** → slip is populated (Exam Password present; per-course exam
  date/time + chosen centre listed).
- **Not booked** → slip is empty (blank password/name, no date rows), as observed.

Reading the slip changes nothing: the page's only actions are **Logout** and
**Send email to myself**; neither books or alters a selection. The chosen
centre/slot therefore **can be read** from a booked slip without touching the
booking. Note the trade-off vs. the LMS: inside the LMS "datesheet booked" is
**not** visible (per #2), so this external slip is the only place the Handled
signal for datesheet exists.

(Not verifiable with this account: the exact rendered layout of a *booked* slip,
since the account had no booking and the window is closed. The field labels and
the landing-page instructions make the content unambiguous, but confirm the exact
markup against a booked account during a live window.)

## 4. One active session per student — shared with the LMS?

- The datesheet app issues its **own** `ASP.NET_SessionId` cookie on
  `datesheet.vu.edu.pk` (separate from the LMS's cookie on `vulms.vu.edu.pk`).
- Logging in to datesheet **alone** produced no "Multiple Login" page; login was
  clean and landed on the slip. (Contrast the LMS, where a second concurrent login
  bumps the other to `MultiLoginMessage.aspx` — see #2.)
- The login page is the **shared** VU SSO ("Login : VULMS", `action=datesheet`),
  so the **credential check** is centralised, but the single-active-session
  *enforcement* observed in #2 is an LMS-application behaviour and was **not**
  observed to extend to datesheet in this read.
- **Not fully confirmed** whether a datesheet login invalidates a concurrent LMS
  session (would require being logged into both at once and watching one get
  bumped — not done, to avoid disrupting the account). **Safe assumption:** treat
  the limit as *possibly* shared, keep the datesheet session short, and log out —
  which we did.

## 5. Navigation / scraping model

- WebForms + `__doPostBack`, `__VIEWSTATE`/`__EVENTVALIDATION`, AJAX
  `UpdatePanel`s on the slip. Same shape as the LMS, so a browser (Playwright) is
  the pragmatic driver; stable ids above (`btnLogin`, `txtSId`, `txtP`,
  `lblPassword`, `lblStudentID`, `btnLogout`) are good anchors.
- Logout: `btnLogout` → `login.aspx` (verified; session ended, revisiting the
  landing starts fresh).

## 6. Implications for the notifier (not decided here)

- **Window detection for datesheet stays login-free**: scrape the public landing
  page (`http://datesheet.vu.edu.pk/`) for term/exam-type/closing-date and the
  late-fee tiers, as #3 already proposed. A plain `fetch`/cheerio GET suffices for
  this; no browser needed. Infer "open" by comparing today to the signup date
  (the banner lingers after close).
- **Per-Member "have they booked?" (Handled)** requires a **browser login**
  (reCAPTCHA v3) to `datesheet.vu.edu.pk`, then read `ExamEnteranceSlip.aspx`.
  This is the same cost profile as an LMS check and the only way to confirm a
  datesheet booking (the LMS does not show it).
- **Session caution**: because the single-session limit *may* be shared with the
  LMS, a datesheet login could bump the Member's LMS session (and vice-versa).
  Prefer to piggy-back datesheet checks onto the same login slot/schedule as the
  LMS check, keep them short, and always log out. Confirm cross-app bumping before
  relying on concurrent LMS + datesheet sessions.
- **Breakage anchors**: alert if login stops landing on `ExamEnteranceSlip.aspx`
  (or the booking page during a window), if the "Login : VULMS" page or the
  reCAPTCHA site key changes, or if the landing-page spans (`lblSignupLastDate`,
  late-fee panel) disappear.

## 7. Caveats / not verified

- Tested with one account, one IP, on 2026-10-12, with **no datesheet window
  open**. The live booking page (city/centre/slot selectors, the "Confirm" flow)
  and a **populated** entrance slip were therefore not observed first-hand — only
  the out-of-window slip and the public landing/late-fee markup.
- Cross-application session sharing with the LMS is **unconfirmed** (section 4).
- VU can change markup, the reCAPTCHA config, TLS, or session policy at any time.
