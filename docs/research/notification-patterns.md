# Notification patterns in learning platforms

Research for [#2](https://github.com/StijnEntras/cohort-005-project-stijn-pocock/issues/2), part of the map [#1](https://github.com/StijnEntras/cohort-005-project-stijn-pocock/issues/1) (In-app Notifications and Nudges spec). Researched 2026-11-16.

**Question.** What do established learning platforms and in-app inbox patterns do for Notifications and motivational Nudges? Which events do they notify on, how do they aggregate and keep noise down, what read/unread and click-through conventions are standard, and what evidence is there on what drives continuation versus fatigue?

Vocabulary follows `GLOSSARY.md`: **Student**, **Instructor**, **Notification** (something happened that involves you), **Nudge** (motivation prompted by time passing or your own progress).

**How much to trust each source.** Each claim is tagged:
- **[P]** primary: the platform's own docs or source code, or the paper's own text.
- **[S]** read through a secondary page (a university help mirror, a search snippet, or a press summary) because the primary page could not be fetched (403 or 404). Treat these as directional.

---

## 1. Which events platforms notify on

### Moodle [P]
Moodle's notification menu sends these, among others ([Moodle Docs: Notifications](https://docs.moodle.org/en/Notifications)):
- **Assignments:** submissions (to teachers), overdue submissions, grades and feedback, and reminders 48 hours and 7 days before the due date.
- **Forums:** posts in forums you subscribe to, plus forum digests.
- **Quizzes:** submissions (to teachers), submission confirmations, overdue warnings, grading feedback, and a notice 48 hours before a quiz opens.
- **Other:** badges awarded, course completion, enrolment expiry, private messages.

Preferences are set per notification type and per channel (web, email, mobile, SMS). Admins set the defaults and can lock types.

### Open edX [P]
The discussion Notifications are listed on [Staying Updated with Notifications](https://docs.openedx.org/en/ulmo/learners/sfd_discussions/discussions_notifications.html). They are scoped by your relationship to the content:
- **Your post:** new responses, new comments, and an endorsed response.
- **Your response:** new comments, and endorsement.
- **Posts you follow:** new responses and new comments.
- **New posts:** discussion posts, question posts, and instructor posts.

Defaults:
- "Activity Notifications" (everything you authored or follow): tray **on**, email daily.
- "New posts from instructors": tray **on**.
- New discussion posts and new question posts from anyone: **off** in both tray and email.

Platform-wide, the tray has tabs such as **Discussions, Updates, Grading**. Preferences cover nine activity types, each with a tray toggle and an email toggle. Email cadence is Immediately, Daily or Weekly, and digests show at most 5 items per area with a "View more" link back to the tray ([Notifications & Preferences](https://docs.openedx.org/en/latest/learners/sfd_notifications/index.html)).

### Canvas [S]
Canvas groups Notifications into course activities, discussions, conversations, groups, scheduling and conferences. Each type can be set to *right away*, *daily summary*, *weekly summary*, or *off*, and course-level settings override account-level ones.

Typical institutional defaults:
- Announcement: right away.
- Submission comment (an instructor commenting on your work): daily or right away.
- **New Discussion topic: off.**
- Discussion post in a thread you take part in: daily.

Sources: [Indiana University KB](https://www.kb.indiana.edu/d/bfdl), [SJSU defaults](https://sjsu.edu/cfeti/teaching-resources/software/canvas/student-resources/canvas-notifications-default-settings.php), [Tufts recommendations](https://tuftsedtech.screenstepslive.com/s/18992/m/73355/l/1527735-what-are-the-recommended-notification-preferences-for-students). The Instructure community pages redirected or returned 404.

### Udemy [S]
- Instructors get Notifications for **Q&A, direct messages and course reviews**, with push options for each.
- **Q&A reply Notifications cannot be disabled.** The "Daily Q&A Digest" email is a separate, optional setting.
- Students can schedule their own daily, weekly or one-time **learning reminders** under *My learning > Learning tools* ([Udemy Business help](https://business-support.udemy.com/hc/en-us/articles/4501800912919-How-to-schedule-learning-reminders-on-the-Udemy-Business-mobile-app)).
- The instructor notifications help page returned 403, so those details come from search snippets of [Instructor Account Notifications](https://support.udemy.com/hc/en-us/articles/115008393928-Instructor-Account-Notifications).

### Teachable [P]
- School owners and authors can choose to be notified of **new comments in their courses** ([Course comments](https://support.teachable.com/en/articles/11682464-course-comments)).
- In the admin comment inbox, new comments are **bold with a blue dot**. Clicking one marks it read, and there is **Mark as Read / Mark as New**, individually and in bulk.
- The comment Notification carries the Student, course, lesson and timestamp, and deep-links to where you reply.

### Patterns across platforms
1. **"Involves you" beats "happened somewhere."** Every platform defaults on for replies to your own content and for activity in threads you follow. They default **off** for "someone posted something new" in a course, both Canvas (new discussion topic) and Open edX (new discussion and question posts).
2. **Instructor-authored content is the exception.** Announcements in Canvas and instructor posts in Open edX default on for Students.
3. **Instructors are notified about Student questions and comments.** Udemy Q&A, Teachable comments and Moodle submissions all do this. Udemy makes Q&A replies non-optional.
4. **Time-based reminders are their own category.** Moodle due-date reminders, Udemy learning reminders and Open edX goal reminders (section 4) all behave like Nudges, not Notifications.

---

## 2. Inbox conventions: badge, read/unread, click-through

| Convention | Evidence |
| --- | --- |
| A bell or badge in the masthead opens a drawer or popover. The badge shows the unseen count. | Moodle shows "a number in the notifications menu at the top of the screen" [P]. Open edX shows "a count next to the bell icon" of unseen items [P]. PatternFly says the badge "must be placed in the masthead" and toggles the drawer ([PatternFly notification drawer](https://www.patternfly.org/components/notification-drawer/design-guidelines)) [P]. |
| Newest first. | PatternFly: "reverse chronological order" [P]. |
| Unread items look different. | PatternFly: bold [P]. Teachable: bold with a blue dot [P]. Open edX: a red dot until clicked [P]. |
| **Clicking an item marks it read and deep-links to the subject.** | Moodle: clicking the body "takes you to where the event happened" [P]. PatternFly: one click marks it read [P]. Teachable: the same [P]. |
| **Mark all as read.** | Moodle's popover offers mark all as read, preferences, and view all [P]. PatternFly has a global "Mark all read" and a per-category version [P]. |
| A "view all" page in addition to the popover. | Moodle [P]. |
| Few categories or tabs. | Open edX uses 3 tabs (Discussions, Updates, Grading) [P]. PatternFly warns against "more than 3-4 categories" [P]. |
| A reason for each item. | GitHub labels each inbox item with its reason, such as mention, subscribed or review requested, and you can filter by it ([GitHub: About notifications](https://docs.github.com/en/subscriptions-and-notifications/concepts/about-notifications)) [P]. |
| Done/archive and unsubscribe per thread. | GitHub: *Done* removes an item from the inbox. *Unsubscribe* stops updates from that conversation unless you are @mentioned again [P]. |
| Retention of about 3 months. | GitHub keeps unsaved items 3 months [P]. Open edX lets the site admin set a limit; Confluence notes say 3 months [P]/[S]. |
| The drawer is not for urgent action. | PatternFly: don't use it as the only way to alert users to events that need immediate action [P]. |

**Seen versus read.** Open edX counts **unseen** items on the badge (opening the tray clears it) but keeps a per-item **unclicked** dot. Two states work well together: the badge answers "anything new since I last looked?" and the dot answers "have I dealt with this one?" [P].

---

## 3. Aggregation and noise control

- **Auto-subscribe on participation, then thread-level updates.** GitHub subscribes you automatically when you open, comment on, or are mentioned in a thread, and keeps your Notifications at the level of that conversation [P]. Open edX's "posts you follow" works the same way [P].
- **Group while unseen.** When a new Notification arrives, Open edX checks for an existing **unseen** Notification with the same type and context. If one exists, it updates that one instead of adding a new one. Once the user has **seen** it, the next event starts a new group ([Open edX Confluence: Current state of notifications](https://openedx.atlassian.net/wiki/spaces/OEPM/pages/4750475268), via search snippet) [S]. Grouping currently ships only for a few types (new discussion posts) behind a flag [S].
- **Merge preferences, not just items.** Open edX folds several activity types into one "Activity Notifications" preference "to avoid clutter in preferences" [P].
- **Digests for the noisy kinds.** Canvas uses daily or weekly summaries, Open edX caps email digests at 5 items per area, and Udemy has a Daily Q&A Digest [P]/[S]. Cadence is in-app only, so the in-app equivalent is the grouped row: "5 new comments on Lesson X".
- **Volume is the fatigue driver.** In a one-week in-situ study, information workers handled **63.5 Notifications per day** on average, mostly from messengers and email, and usually looked at them within minutes (Pielot, Church & de Oliveira, *An In-Situ Study of Mobile Phone Notifications*, MobileHCI 2014, [PDF](https://ic.unicamp.br/~oliveira/doc/MHCI2014_An-in-situ-study-of-mobile-phone-notifications.pdf)) [S, abstract-level]. Mehrotra et al. (CHI 2016, [PDF](https://discovery.ucl.ac.uk/1502364/1/CHI16.pdf)) found that receptivity depends on the **sender–recipient relationship** and the content, not just the timing [S]. That supports ranking "a person replied to you" above system-generated items.

---

## 4. Nudges: what the evidence says

### Open edX weekly goal reminders (source code) [P]
This is the most concrete published Nudge rule set found. In [`goal_reminder_email.py`](https://github.com/openedx/edx-platform/blob/master/lms/djangoapps/course_goals/management/commands/goal_reminder_email.py), a Student who sets a goal of *N days per week* gets a reminder only if all of the following hold:
- they are subscribed to reminders, and **no reminder has been sent this week** (the status resets on Monday);
- they are actively enrolled, and **did not enroll this week**;
- the course does not end, and their access does not expire, before Sunday, so the goal is still possible;
- they have **not already completed** the course (no certificate);
- **days of goal still needed == days left in the week.** The reminder fires on the last day the goal can still be met, never earlier and never once it is already missed;
- it is between 08:00 and 18:00 in the Student's timezone.

Learners can change or cancel the goal and reminders "at any time from the widget or via links in the email" ([config docs](https://edx.readthedocs.io/projects/edx-installing-configuring-and-running/en/open-release-olive.master/configuration/enable_weekly_learning_goals.html)) [P].

**Lesson:** a well-designed Nudge is **conditional, at most weekly, and silent when it would be pointless**: goal already met, goal impossible, new Student, or finished Student.

### Duolingo practice reminders [P/S]
- Duolingo chooses each daily reminder from a pool of templates using a bandit algorithm. The reward is a lesson completed within **two hours** of the reminder (Yancey & Settles, *A Sleeping, Recovering Bandit Algorithm for Optimizing Recurring Notifications*, KDD 2020, [PDF](https://research.duolingo.com/papers/yancey.kdd20.pdf)) [S: the PDF text could not be extracted, so this comes from summaries of it].
- Duolingo's own blog: "learners don't like seeing the same notification too often." Recently seen templates are demoted along a forgetting curve, and new templates are tested on a small group first ([Duolingo blog](https://blog.duolingo.com/hi-its-duo-the-ai-behind-the-meme/)) [P].
- **Lesson:** repeated identical Nudge copy loses effect. Rotate the wording, and measure a Nudge by whether the Student resumes soon after seeing it.

### Behavioural interventions at scale [S]
- Kizilcec et al., *Scaling up behavioral science interventions in online education*, PNAS 2020 ([DOI 10.1073/pnas.1921417117](https://doi.org/10.1073/pnas.1921417117)): about 250,000 students across 247 MOOCs over 2.5 years.
- Scaling up "can reduce their average effectiveness by an order-of-magnitude."
- Self-regulation interventions (planning) raised **early engagement but not completion**.
- A model that targeted interventions to individuals did no better than giving everyone the same one. The authors: "not a silver bullet."
- The PNAS page returned 403. These figures come from the [MIT TSL summary](https://tsl.mit.edu/research/the-limits-of-scalable-interventions/) and [Cornell press release](https://news.cornell.edu/node/319503).
- **Lesson:** expect small Nudge effects. Keep Nudges cheap and separate from Notifications (as the map already decided), and measure them against a holdout rather than assuming a lift.

### Goal reminders at Coursera [S]
A Coursera Engineering post reports that Students who set a goal and were reminded of it were **7% more likely** to complete it than a holdout who set goals without reminders ([The Power of Goals](https://medium.com/coursera-engineering/the-power-of-goals-a5134b8d0e1f); the page returned 403, so the figure comes from a search snippet). This points the same way as edX: reminders tied to a goal the Student chose.

---

## 5. Recommendations for Cadence

These are grounded in what Cadence has today (`app/db/schema.ts`):
- `comments`: per lesson, one level of replies, soft delete, edits.
- `lesson_progress` and `video_watch_events`: Watched time.
- `enrollments`, `course_ratings`, `refunds`.
- the `instructor.questions` route.

### Student Notifications (candidates, in priority order)
1. **Someone replied to your comment.** The core case. Default on and **not mutable**, like Udemy's Q&A replies. If the reply is by the course's Instructor, show that prominently ("Instructor replied").
2. **New reply in a thread you replied in.** This is the participating auto-subscribe model from GitHub and Open edX. Make it unsubscribable per thread.
3. **Refund Request decided** (approved or declined) for the buyer. It involves the Student directly and is rare, so it is low-noise.
4. *(Optional, lower priority)* **New lesson or module published** in a course you're enrolled in. Group per course ("3 new lessons in Course X"). Platforms treat Instructor-authored content updates as default on.

Do **not** notify Students about other people's new top-level comments. Canvas and Open edX both default that off.

### Instructor Notifications (candidates)
1. **New top-level comment or question on one of your lessons.** Group per lesson while unseen: "5 new comments on *Lesson X*".
2. **Reply in a thread on your lesson that you took part in**, for example a Student following up on your answer.
3. *(Optional)* **New course rating**: Udemy notifies on reviews. **New Enrollments**: group per course per day at most.

### Nudges (separate thread, separate visual treatment)
- **Student, "pick up where you left off".** The Student has an Enrollment, some Watched progress, no activity for N days (for example 7), and the course is not complete. At most one per course per week. Stop after 2–3 Nudges are ignored. Never send to Lapsed Students or Students who enrolled in the last N days. This copies the edX suppress-when-pointless rules.
- **Student, progress milestones** such as finishing a module or the course, linking to the next lesson. These fire on progress, so they come at a moment when the Student is likely receptive.
- **Instructor, "Students are waiting".** Comments or questions on your lessons with no Instructor reply after X hours (for example 48h), grouped into one Nudge ("3 Student questions waiting more than 2 days"). This directly serves the goal that Instructors react to Students. At most once a day.
- Rotate the copy for each Nudge kind (Duolingo novelty decay). Measure each Nudge by whether the Student resumes a lesson soon after seeing it, or the Instructor replies, against a holdout (Kizilcec: expect small effects).

### Aggregation and read/unread conventions
- **One inbox row per (recipient, kind, subject)** while it is unseen. A new event on the same subject updates the row's count, actor list and timestamp, and moves it to the top. Once the row is seen, the next event starts a new row (the Open edX model). Suggested subjects: the parent comment thread for replies, the lesson for new comments, the course for new lessons.
- **Badge = unseen count** (cleared by opening the bell). **Per-row unread dot or bold** is cleared by clicking the row or by **Mark all as read**.
- **Clicking deep-links** to the subject, for example the lesson page anchored at the comment, and marks the row read.
- **Newest first.** A popover shows the latest ~10, plus a "View all" page.
- **At most 2–3 categories.** Notifications and Nudges at minimum, or Students' and Instructors' views. Render Nudges distinctly so they never bury Notifications.
- **Show a reason on each row** ("because you commented", "on your lesson"), and offer **unsubscribe from this thread** (GitHub).
- **Retention of about 3 months** (GitHub, Open edX). Rows whose subject is deleted should fall back gracefully, since comments are soft-deleted. This feeds the "Retention & expiry" question on the map.
- **Model the Notification so it can later be delivered by email or as a digest** (out of scope now). Every platform surveyed uses one event with multiple channels and per-type preferences.
