# Public sources for Institutional Windows (outside the LMS)

Research for issue #3: where Virtual University of Pakistan (VU) announces **Institutional Windows** (datesheet booking, datesheet availability, course selection, fee deadlines and similar) in places that need no login.

Checked on 2026-10-12, mostly with direct `curl` requests against the live sites (a few requests per site). Every date below comes from VU's own pages unless marked *third-party*.

## Answer in brief

- **No login is needed to find out about any Institutional Window.** Three public VU pages cover them all: the **news detail pages**, the **Academic Calendar** page, and the **datesheet portal landing page**. A login is only needed for a Member's *own status* inside a window, such as whether they have already booked their datesheet or selected courses.
- **Nothing is machine-readable in the API sense.** There is no RSS, Atom or JSON feed. Every source is server-rendered ASP.NET WebForms HTML, which is easy to parse with plain GETs.
- **Datesheet booking windows are short.** They usually run 7–10 days, but the Summer 2026 window lasted about 2 days (Oct 1 to Oct 2, 2026). Polling a few times a day is the minimum.
- **Some windows have tiers.** A datesheet window has a free period, then a one-day "late fee" day, then a one-day "double late fee" day. A single closing moment does not describe that well.
- **Social channels add nothing a scraper can use.** Facebook, X and the WhatsApp channel repeat the website on the same day and cannot be read without a login or JavaScript. No official Telegram channel exists.

## Sources

### 1. vu.edu.pk news (primary source for datesheet and other notices)

| | |
|---|---|
| List | `https://www.vu.edu.pk/news/NewsList.aspx` (about the 15 latest items, each with a date; no paging without a WebForms postback) |
| Homepage | `https://www.vu.edu.pk/` shows the 4 latest items in a `rptrNews` repeater |
| Detail | `https://www.vu.edu.pk/NewsDetails.aspx?type=&NewsId=<int>`. This is a plain GET (the list links are `WebForm_DoPostBackWithOptions` wrappers, but the target URL is in the markup). |
| Format | HTML. The detail page has a `Published On: <weekday, Month D, YYYY>` line, and sometimes `First Published On:` as well, followed by a title and a free-text body. Dates in the body are written in prose ("Thursday, January 22nd, 2026"), often split by `<sup>` tags. |
| Feeds | None. `/rss`, `/rss.xml`, `/feed` and `/news/rss.aspx` all return 404. `/sitemap.xml` is static, with no news entries and no lastmod. There is no robots.txt (404). |
| Login | Not needed. |
| Gotchas | **The TLS chain is incomplete.** `curl` fails with "unable to get local issuer certificate", so the scraper must bundle the intermediate certificate (or similar) rather than turn verification off silently. **NewsIds are not strictly chronological**: 9530 (Course Selection, "Published On" Oct 6) is lower than 9531 (Datesheet, Oct 1), so posts appear to get edited or re-dated. Detect new items as "id not seen before", not as "id greater than the last maximum". |
| Promptness | Datesheet notices go up on the day the portal opens. Course selection is less reliable: the Fall 2026 notice (9530) is dated Oct 6, 2026, but the link opened on Sep 18, 2026, per the notice itself and the calendar. |

Example window announcements (all public):

| NewsId | Published | Title | Window |
|---|---|---|---|
| 9203 | Fri Jan 17, 2025 | Make Your Date sheet NOW! For FALL 2024 Final Term | Free until Jan 22; late fee Jan 23; double late fee Jan 24, 2025 |
| 9382 | Wed Jan 14, 2026 | Make Your Date sheet NOW! For Fall 2025 Final Term | Free until Jan 20; late fee Jan 21; double late fee Jan 22, 2026 |
| 9444 | Thu May 14, 2026 | Make Your Date sheet NOW! For Spring 2026 Mid-Term | Free until May 21; late fee May 22; double late fee May 23, 2026 |
| 9470 | Fri Jul 17, 2026 | Spring 2026 End Semester Examinations – Make Your Date sheet NOW! | Free until Jul 21; late fee Jul 22; double late fee Jul 23, 2026 |
| 9531 | Thu Oct 1, 2026 | Make Your Date Sheet NOW! For Summer 2026 End-Term | Closes Oct 2, 2026 11:59 PM PST; no late tiers |
| 9530 | Tue Oct 6, 2026 | Announcement for Course Selection Fall 2026 Semester | Opened Fri Sep 18; add/drop closes Wed Oct 21, 2026 |
| 9534 | Mon Oct 5, 2026 | Summer 2026 … Rescheduling of Missed / Unplanned Paper(s) | Apply by Tue Oct 6, 11:59 PM; exam held Wed Oct 7, 2026 |
| 9539 | Oct 7, 2026 | Extension of Deadline for Applying for Need-Based Scholarship – Fall 2026 | Extends a calendar deadline (Oct 16) |

The notices also contain other windows we could track: rescheduling of missed papers, scholarship deadlines, convocation registration (9516) and admission extensions (9532).

### 2. Academic Calendar (primary source for course selection, term dates and exam periods)

| | |
|---|---|
| URL | `https://www.vu.edu.pk/StudentServices/AcademicCalendar.aspx` |
| Format | An HTML table of rows (Description, Day, Date) for each semester. The page currently holds **two calendars**: Summer 2026 (in a modal) and Fall 2026. Date formats differ between tables (`03-Aug-26` and `September 18, 2026`), and some rows have typos (an orientation listed as "October 12 - 12, 2026"). |
| Login | Not needed. |
| Promptness | Published ahead of the semester. Fall 2026 dates were already on the page well before Sep 18. VU also posts extensions as news (e.g. 9532 and 9539), so the calendar alone can go out of date. |

Fall 2026 rows, as published:

- Option to change study program for existing students: Fri Sep 4, 2026
- **Course Selection link open for enrolment: Fri Sep 18, 2026**
- Last date for course exemption / credit transfer: Sat Oct 10, 2026
- Classes commence: Mon Oct 12, 2026 (orientation week Oct 12–16)
- Last date to apply for need-based scholarship: Fri Oct 16, 2026 (later extended by news 9539)
- Last date for semester unfreeze: Wed Oct 21, 2026
- **Last date for course selection add/drop/replace: Wed Oct 21, 2026**
- Mid-term exams: from Mon Dec 7, 2026
- Final-term exams: from Mon Jan 25, 2027

Summer 2026 rows: course selection opened Mon Aug 3; add/drop closed Thu Aug 13; classes started Aug 17; mid-terms Sep 14–15; finals Oct 3–4; results Wed Oct 7, 2026.

Spring 2026 (*third-party*, ilmkidunya): course selection opened around Mar 3, 2026, with orientation Apr 1–6.

**Datesheet booking dates are not on the calendar**, only the exam periods. The booking window is announced only through the news page and the portal.

### 3. Datesheet portal landing page (cheapest live signal for datesheet booking)

| | |
|---|---|
| URL | `https://datesheet.vu.edu.pk/` |
| Format | A small HTML page. Without a login it shows the current term and its deadline, e.g. "Make your Date Sheet – NOW / Summer 2026 / Finalterm Examinations / Last Date of Signup is Oct 02, 2026". |
| Login | Not needed for the banner. Booking itself, and a Member's own status, require a VULMS login. |
| Caveat | The banner **stays up after the window closes** (it still showed the Oct 2 deadline on Oct 12). It is good for spotting changes, but "open" has to be inferred from today's date compared with the last signup date. |

### 4. Social channels (do not use as sources)

| Channel | URL | Machine-readable / no login? | Notes |
|---|---|---|---|
| X | `https://x.com/VUPakistan` (linked from the vu.edu.pk footer) | No. JS app; fetching a status returns HTTP 402 with no body. | Posts datesheet openings on the same day as the website. Tweet 2054909248669155742, "Date Sheet Portal is now open!", decodes to 2026-05-14 12:58 UTC, which matches news 9444 published May 14. |
| Facebook | `https://www.facebook.com/VirtualUniversityOfPakistan` (footer) | No. A logged-out HTML fetch returns about 500 KB with no post text. | Same content as the website. |
| WhatsApp Channel | `https://whatsapp.com/channel/0029VbAyX5M3LdQSZz3n6Z1o` ("Virtual University", about 31K followers, describes itself as official and links to vu.edu.pk) | No. Only the og:title and description are public; posts need the app. | Not linked from vu.edu.pk, so its official status is unconfirmed. Other "VU" channels are run by students. |
| Telegram | none found | n/a | No official channel turned up in search, and none is linked from vu.edu.pk. |
| Instagram, YouTube, LinkedIn | footer links | Need login or JS | Mostly promotional. |

### 5. Email

- VU emails individual students at their **VU email address** from `no-reply@vu.edu.pk` (for example, rescheduling confirmations in 9534). That inbox belongs to each student and is not a public source.
- No public mailing list or newsletter signup for window announcements was found. `newsletter.vu.edu.pk` exists but was not examined.
- Contact addresses that appear on the pages: `datesheet@vu.edu.pk` and `password@vu.edu.pk`.

## Historical patterns

**Datesheet booking (mid-term and final) opens about 7–12 days before the exam and closes 1–3 days before it.**

| Term / exam | Opened (news date) | Free until | Hard close | Length |
|---|---|---|---|---|
| Fall 2024 final | Jan 17, 2025 | Jan 22 | Jan 24, 2025 | 8 days |
| Fall 2025 final | Jan 14, 2026 | Jan 20 | Jan 22, 2026 | 9 days |
| Spring 2026 mid | May 14, 2026 | May 21 | May 23, 2026 | 10 days |
| Spring 2026 final | Jul 17, 2026 | Jul 21 | Jul 23, 2026 | 7 days |
| Summer 2026 final | Oct 1, 2026 | — | Oct 2, 2026 23:59 PST | 2 days (exams Oct 3–4) |

- Fee tiers are always the same: free, then a one-day late fee (Rs 500 / $20), then a one-day double late fee (Rs 1000 / $25), with the double-late day as the hard close. Summer 2026 had no tiers.
- Seats are first come, first served, so an early **Window Alert** is worth more than a late **Reminder**.
- **Datesheet availability**, meaning the exam period itself, comes from the calendar (e.g. Fall 2026 mid-terms from Dec 7, finals from Jan 25, 2027). The likely Fall 2026 booking windows are therefore around late November for mid-terms and around Jan 13–22, 2027 for finals. This is an inference from the pattern above.

**Course selection opens 2–4 weeks before classes and add/drop closes about 9 days after classes start.** Fall 2026: Sep 18 to Oct 21 (classes Oct 12). Summer 2026: Aug 3 to Aug 13 (classes Aug 17).

**Rescheduling of missed papers** has a window of about 1 day, right after the end-term exams (Oct 5–6, 2026, with the exam on Oct 7).

## What this means for the plan

1. Institutional Windows can be found **without any Member login**, from three plain-HTML pages. A login is only needed to mark a window as handled for a particular Member.
2. Scrape **news + calendar + datesheet banner**. Ignore social channels.
3. The scraper must cope with **VU's incomplete TLS chain** and with **NewsIds that are not in order**.
4. Poll **at least every few hours**: one datesheet window lasted about 2 days.
5. The domain model may need **tiered closing moments** (free, late fee, double late fee) for an Institutional Window. That would change the definition of "closing" in CONTEXT.md.
6. Window dates are written in prose with inconsistent formats, so extracting them is a parsing problem (or a job for an LLM), not structured data.
