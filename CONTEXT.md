# VU Smarter Notifications

Timely email reminders for Virtual University of Pakistan (VU) students about things they must act on, sourced from VU LMS and VU's other channels.

## Language

**Member**:
One of the small group of VU students the notifier serves; each Member's own LMS login is used only for their own data.
_Avoid_: user, customer, subscriber

**Personal Deadline**:
A dated obligation tied to one Member's enrolled courses (assignment, quiz, GDB, MDB, result release).
_Avoid_: task, due item

**Institutional Window**:
A period that applies to all students, with an opening and a closing moment, during which an action is possible (datesheet booking, datesheet availability, course selection).
_Avoid_: announcement, deadline

**Reminder**:
An email sent to a Member ahead of a Personal Deadline's due moment or an Institutional Window's closing.

**Window Alert**:
An email sent when an Institutional Window opens or changes.

**Digest**:
A single daily email summarising a Member's open Personal Deadlines and Institutional Windows.

**Handled**:
A Personal Deadline the Member has already completed (e.g. submitted), which suppresses further Reminders.

## Relationships

- A **Member** has many **Personal Deadlines**; an **Institutional Window** is shared by all **Members**
- **Reminders** and **Window Alerts** are both delivered by email; a **Digest** collects both kinds of item
- A **Handled** Personal Deadline produces no further **Reminders**
