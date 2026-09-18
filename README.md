# Time Manager & Study Planner (with Pomodoro Tracker)

A command-line Java application that acts as a **time manager** for students:
tasks carry real date-and-time deadlines, a **live clock dashboard** counts
down to each one, and a background watcher **reminds you shortly before a
deadline arrives**. Focused work is tracked with a built-in Pomodoro timer,
and everything is persisted locally in a SQLite database.

Built for **CSE2006 – Programming in Java** as a "Build Your Own Project"
submission.

---

## 1. Overview

Students juggle multiple subjects, assignments, and exams. Deadlines are
usually tracked at day-level granularity ("due Friday"), which is too coarse —
a 5 PM submission and a 11:59 PM submission are very different things, and
neither warns you while there is still time to act.

This application treats time as a first-class concept:

- Every task has a **deadline with a date and a time** (`2026-09-20 17:30`).
- Every task has its own **reminder lead time** — "warn me 30 minutes before",
  or 2 hours, or a day, per task.
- A **background watcher thread** continuously compares the live clock against
  those deadlines and raises an alert the moment a task enters its warning
  window, and again if a deadline passes.
- A **live dashboard** shows the running wall clock and a real-time countdown
  to each upcoming deadline, repainting every second.
- A **Pomodoro timer** runs focused work sessions against a chosen task, so
  you can see not just what is due, but how much time you've actually put in.

## 2. Features

1. **Subject & Task Management** — add subjects; add tasks (Assignment / Exam
   Prep / Reading) with priority, a date-and-time deadline, and a per-task
   reminder lead time; list, reschedule, complete, and delete tasks.
2. **Live Clock & Deadline Dashboard** — a running clock plus live countdowns
   to your next deadlines, repainted once per second in place, with tasks
   inside their reminder window flagged. Press ENTER to return to the menu.
3. **Deadline Reminder Engine** — a background daemon thread polls every 15
   seconds and alerts you *before* each deadline (at that task's own lead
   time) and again when one passes. Each alert fires exactly once, and
   rescheduling a task re-arms its reminder.
4. **Pomodoro Timer Engine** — live MM:SS countdown on a dedicated thread;
   press ENTER at any time to stop early; every session (completed or
   interrupted) is logged with actual minutes spent and attributed to the task.
5. **Reporting & Analytics** — a weekly `.txt` report (time spent per subject,
   task status breakdown, overdue list, upcoming deadlines with countdowns)
   and a `.csv` subject summary, both written to `reports/`.
6. **Robust validation & error handling** — custom checked exceptions for
   invalid input, duplicate subjects, missing tasks, and concurrent-session
   conflicts; no raw stack traces are ever shown to the user.

## 3. Technologies / Tools Used

| Concern       | Tool / Concept |
|---------------|----------------|
| Language      | Java 17+ (standard library only in application code) |
| Build tool    | Apache Maven |
| Persistence   | SQLite via JDBC (`org.xerial:sqlite-jdbc`) |
| Date & time   | `java.time` — `LocalDateTime`, `Duration`, `DateTimeFormatter` |
| Concurrency   | `Thread` / `Runnable`, `volatile` flags, `synchronized` blocks, `ConcurrentHashMap.newKeySet()` |
| I/O           | `java.io` character streams (`FileWriter` / `PrintWriter`) for logs and reports |
| Collections   | `ArrayList`, `TreeMap`, `PriorityQueue`, concurrent sets, Streams API |
| OOP concepts  | Abstract classes, inheritance, polymorphism, interfaces (`Comparable`), enums |

## 4. Threading Model

Four threads cooperate; this is the core of the design:

| Thread | Lifetime | Responsibility |
|---|---|---|
| `main` | whole session | Menu loop and all user input |
| `reminder-daemon` | whole session (daemon) | Polls the clock every 15s; raises pre-deadline and overdue alerts |
| `live-clock` | on demand | Repaints the countdown dashboard once per second |
| `pomodoro-timer` | on demand | Runs a focus session's live countdown |

Shared state is protected deliberately:

- **Console output** — all cross-thread printing goes through `ConsoleSync`,
  which holds a single monitor lock, so a reminder firing mid-countdown cannot
  garble the display.
- **Database writes** — DAO write methods synchronize on `DatabaseConnection.class`
  to serialize access to the shared SQLite connection.
- **Stop signals** — `volatile boolean` flags (`stopRequested`, `running`) for
  one-way signals between threads.
- **Alert de-duplication** — `ConcurrentHashMap.newKeySet()` tracks which tasks
  have already alerted, written by the daemon thread and cleared by the main
  thread on reschedule/complete/delete.

## 5. Project Structure

```
study-planner/
├── pom.xml                         # Maven build config (dependencies + run/package plugins)
├── README.md
├── statement.md                    # Problem statement / scope / target users
├── data/                           # SQLite database file lives here (auto-created)
├── logs/                           # reminders.log written here at runtime
├── reports/                        # generated weekly/CSV reports written here
└── src/main/java/com/studyplanner/
    ├── Main.java                   # CLI entry point / menu loop
    ├── model/                      # Task hierarchy, Subject, Priority, TaskStatus, PomodoroSession
    ├── service/                    # TaskService, PomodoroTimer, ReminderDaemon, ClockService
    ├── db/                         # DatabaseConnection + DAOs (JDBC CRUD, schema, migrations)
    ├── report/                     # ReportGenerator (file I/O based reports)
    ├── exception/                  # Custom checked exceptions
    └── util/                       # ConsoleSync, InputValidator, TimeFormatter
```

## 6. Prerequisites

1. **Java Development Kit (JDK) 17 or newer**
   - Check with: `java -version` and `javac -version`
   - Download: https://adoptium.net/ if not already installed.
2. **Apache Maven 3.8+**
   - Check with: `mvn -version`
   - Download: https://maven.apache.org/download.cgi
   - Maven needs internet access **the first time you build**, to download the
     SQLite JDBC driver into your local `~/.m2` repository. After that first
     successful build it is cached and no further internet access is required.
3. **Git** (only needed to clone the repository)

No separate database server is required — SQLite is file-based and its driver
is a pure-Java library pulled in automatically by Maven.

## 7. Setup & Installation

```bash
# 1. Clone the repository
git clone <your-repository-url>
cd study-planner

# 2. Build the project (downloads dependencies on first run)
mvn clean compile
```

If you see `BUILD SUCCESS`, you're ready to run.

## 8. Running the Application

### Option A — Run with the Maven exec plugin (fastest for development)

```bash
mvn compile exec:java
```

### Option B — Build a runnable JAR and run it

```bash
mvn clean package
java -jar target/study-planner.jar
```

On launch the app shows the current time and anything due in the next 24
hours, then the menu:

```
==================================================
     Time Manager & Study Planner
     with Pomodoro focus sessions
==================================================

Current time: Wed 16 Sep 2026  14:03:21
Due in the next 24 hours:
   Lab 5 report                  2026-09-16 17:30  (in 3h 26m)

  [ Wed 16 Sep 2026  14:03:21 ]
--------------- MENU ---------------
1.  Add subject
2.  List subjects
3.  Add task (with deadline + reminder)
4.  List all tasks
5.  List pending tasks by urgency
6.  Live clock & deadline dashboard
7.  Start a Pomodoro session
8.  Reschedule a task deadline
9.  Mark task as completed
10. Delete task
11. Generate weekly report (file)
12. Export subject summary as CSV
0.  Exit
-------------------------------------
Choose an option:
```

The database file is created automatically at `data/study_planner.db` on first
run — no manual setup required.

### The live dashboard (option 6)

```
+--------------------------------------------------------------+
|  CURRENT TIME : Wed 16 Sep 2026  14:03:21                    |
+--------------------------------------------------------------+
|  UPCOMING DEADLINES                                          |
| >> Lab 5 report           2026-09-16 14:20 17m 59s           |
|    CAT-2 revision         2026-09-16 20:00 5h 56m            |
|    Chapter 7 - JDBC       2026-09-18 18:00 2d 03h 56m        |
+--------------------------------------------------------------+
|  OVERDUE: 1                                                  |
|    Forgotten worksheet      OVERDUE by 3h 20m                |
+--------------------------------------------------------------+
   >> marks a task inside its reminder window.
   Press ENTER to return to the menu.
```

## 9. Deadline & Reminder Formats

**Deadlines** accept either form:

| Input | Interpreted as |
|---|---|
| `2026-09-20 17:30` | 20 Sep 2026 at 5:30 PM |
| `2026-09-20T17:30` | same (ISO `T` separator also accepted) |
| `2026-09-20` | 20 Sep 2026 at **23:59** (end of day) |

**Reminder lead time** is entered in minutes, per task. Press Enter to accept
the default of 30 minutes. Examples: `30` (half an hour before), `120` (two
hours before), `1440` (a day before).

## 10. Suggested First-Run Walkthrough

1. Choose **1**, add a subject: name `Programming in Java`, code `CSE2006`.
2. Choose **3** to add a task. To see the reminder fire quickly, set the
   deadline a few minutes into the future and the lead time to something that
   covers it — e.g. deadline `<today> 14:20` with a lead time of `30` minutes
   while it is currently 14:03. The task is immediately inside its window.
3. Choose **6** to watch the live dashboard: the clock ticks each second and
   the countdown decreases. Your task should be flagged with `>>`. Press
   ENTER to return.
4. Sit at the menu for ~15 seconds — the reminder daemon polls and prints a
   `*** DEADLINE APPROACHING ***` alert once. It will not repeat-spam.
5. Let the deadline pass to see the `!!! DEADLINE PASSED !!!` alert fire once.
6. Choose **8** to reschedule that task further out — its reminders re-arm
   against the new deadline.
7. Choose **7** to run a Pomodoro session against the task; watch the live
   MM:SS countdown, press ENTER to stop early, and note the minutes get logged.
8. Choose **11** to generate a weekly report; check `reports/` for the file.
9. Check `logs/reminders.log` — every alert raised is appended there.
10. Choose **0** to exit cleanly.

## 11. Testing / Validation

The time-handling logic was validated against a purpose-written test harness
covering 21 assertions, all passing:

- **Countdown formatting** — multi-day, hours, minutes, seconds-only, and
  negative (overdue) durations.
- **Lead-time formatting** — minute, exact-hour, and hour-plus-minute cases.
- **Deadline parsing** — full date-time, ISO `T` separator, date-only
  defaulting to 23:59, and rejection of malformed input.
- **Reminder window edges** — inside window, outside window, long lead time
  spanning hours, overdue tasks excluded from the "approaching" alert,
  completed tasks never alerting.
- **Ordering** — same-day tasks correctly ordered by *time*, not just date.

Functional testing was done manually against every menu option, including
deliberately triggering each custom exception (blank title, past deadline,
duplicate subject code, unknown task id, starting a second Pomodoro while one
runs) to confirm clean user-facing messages rather than stack traces, and
verifying persistence by restarting the app.

If you want to add automated tests, JUnit 5 can be added as a `test`-scoped
Maven dependency; the layered structure (service classes depend only on
`db`/`model`, never on `Main`) makes the logic straightforward to unit test in
isolation from the CLI.

## 12. Troubleshooting

| Problem | Likely cause / fix |
|---|---|
| `mvn: command not found` | Maven isn't installed or isn't on your `PATH`. Install it and re-open your terminal. |
| Build fails downloading `sqlite-jdbc` | No internet on first build. Connect once so Maven can populate `~/.m2`, then rebuild offline afterwards. |
| `Deadline must be in the future` | Deadlines must be later than the current date *and time*. |
| Dashboard looks garbled or doesn't repaint in place | The dashboard uses ANSI escape codes. Use a standard terminal; some IDE consoles (and older Windows `cmd`) don't process them. Windows Terminal, PowerShell 7+, macOS Terminal, and Linux terminals all work. |
| Reminder never fires | Check the task's lead time actually covers the gap to its deadline, and that the task isn't already completed. The daemon polls every 15 seconds, so allow a moment. |
| Database has stale data from an older version | The app auto-migrates old date-only deadlines to end-of-day. To fully reset, delete `data/study_planner.db`; it is recreated on next run. |

## 13. Notes on Design Choices

- **`LocalDateTime` over `LocalDate`** — the whole point of a time manager is
  sub-day precision. Deadlines are persisted as ISO-8601 strings, which sort
  lexicographically in the same order they sort chronologically, so SQL
  `ORDER BY deadline` gives correct chronological ordering for free.
- **Per-task reminder lead times** — a 10-minute warning is right for a quiz
  and useless for a 3-day project. Because each task carries its own lead
  time, the "is it time to warn?" test can't be a single fixed SQL cut-off;
  the DAO fetches a generous candidate window and each `Task` decides for
  itself via `isInReminderWindow()`, keeping the rule with the data it
  concerns.
- **Alerts fire exactly once** — without de-duplication a task would re-alert
  on every 15-second poll for its entire lead window. Alerted task ids are
  tracked in concurrent sets, and cleared when a task is rescheduled,
  completed, or deleted so it can legitimately alert again.
- **The dashboard is a modal view** — a CLI has one shared screen, so a
  permanently-ticking clock would fight with menu input. Instead the clock is
  an explicit mode the user enters and leaves with ENTER, while the menu
  header still shows a timestamp on every redraw.
- **SQLite over MySQL/PostgreSQL** — no external service needs to be
  installed or running, so the project is genuinely runnable from a bare
  terminal with only a JDK and Maven present.
- **Polymorphism for tasks** — `Assignment`, `ExamPrep`, and `ReadingTask`
  each estimate required study effort differently, but all calling code
  (service, reports, CLI, dashboard) depends only on the abstract `Task` type.
