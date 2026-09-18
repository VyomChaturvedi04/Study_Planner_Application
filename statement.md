# Problem Statement

## Problem Statement

University students take multiple courses simultaneously, each with its own
assignments, exams, and reading load. Deadlines and study time are usually
tracked informally (memory, sticky notes, or scattered notes apps), which
makes it hard to know how urgent a task really is, how much time has actually
been spent studying each subject, and which deadlines are quietly slipping by.

Two specific gaps motivate this project:

1. **Day-level granularity is not enough.** Most student to-do tools track a
   deadline as a date. In practice a 5:00 PM submission and a 11:59 PM
   submission on the same day are very different constraints, and a tool that
   cannot tell them apart cannot tell you how much time you actually have left.
2. **Passive lists do not warn you.** A list shows what is due only when you
   remember to open it. What a student actually needs is to be told *before* a
   deadline arrives, while there is still time to act.

The **Time Manager & Study Planner** addresses both by treating time as a
first-class concept: deadlines carry a date *and* a time, a live clock
dashboard counts down to them in real time, and a background watcher thread
raises an alert at a per-task lead time before each deadline. Focused work is
then tracked against those tasks using a built-in Pomodoro timer, so the tool
shows not only what is due and when, but how much time has genuinely been
invested — all in a single lightweight, locally-run command-line application
with no external services to configure.

## Scope of the Project

In scope:

- Managing subjects and study tasks (add, list, reschedule, complete, delete).
- Deadlines with full date-and-time precision, and a configurable per-task
  reminder lead time ("warn me N minutes before this is due").
- Three task types with different effort-estimation logic: Assignments, Exam
  Preparation, and Reading tasks.
- A live clock dashboard showing the current time alongside real-time
  countdowns to upcoming deadlines, repainting once per second.
- A background watcher thread that alerts the user before each deadline
  arrives (at that task's own lead time) and again once one has passed, with
  each alert raised exactly once.
- Running timed Pomodoro focus sessions against a specific task, with a live
  countdown and the ability to stop early.
- Persisting all data locally using SQLite via JDBC, so history survives
  between runs.
- Generating file-based reports: a weekly summary (time per subject, task
  status breakdown, overdue list, upcoming deadlines with countdowns) and a
  CSV subject summary.

Out of scope (possible future work, see README's design notes / report's
"Future Enhancements" section):

- A graphical user interface (this is intentionally a CLI tool per the
  project's executability requirements).
- Multi-user accounts, authentication, or cloud sync across devices.
- Calendar/timetable integration (e.g. importing a class timetable).
- Mobile or desktop OS-level notifications (the reminder daemon surfaces
  alerts to the console and to a log file only).
- Recurring/repeating tasks and time-zone handling (all times are local).

## Target Users

- University/college students who juggle several courses and want a single,
  lightweight tool to track study tasks and actually spend focused time on
  them, without needing to install or configure a heavier project-management
  app.
- Students specifically comfortable working from a terminal (this is also a
  demonstration project for a Java programming course, so a CLI interface is
  an explicit requirement rather than a limitation).

## High-Level Features

1. **Subject & Task Management** — organise study work by subject, with
   typed tasks (Assignment / Exam Prep / Reading) that each carry a priority,
   a date-and-time deadline, a reminder lead time, and an effort estimate.
2. **Live Clock & Deadline Dashboard** — a running wall clock shown alongside
   live countdowns to the next deadlines, updating every second, flagging any
   task that has entered its reminder window.
3. **Deadline Reminder Engine** — a background daemon thread that continuously
   compares the live clock against every task's deadline and warns the user
   ahead of time, at a lead time chosen per task, without being asked.
4. **Pomodoro Timer Engine** — a live, threaded countdown timer that logs
   actual focus time against a task, whether the session ran to completion or
   was stopped early.
5. **Reporting & Analytics** — exportable weekly text reports and CSV subject
   summaries showing where study time actually went and what is coming up.
6. **Persistent local storage** — all subjects, tasks, and Pomodoro session
   history are stored in a local SQLite database via JDBC, so nothing is lost
   between sessions.
