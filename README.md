Canvas LMS
======

Canvas is a modern, open-source [LMS](https://en.wikipedia.org/wiki/Learning_management_system)
developed and maintained by [Instructure Inc.](https://www.instructure.com/) It is released under the
AGPLv3 license for use by anyone interested in learning more about or using
learning management systems.

[Please see our main wiki page for more information](http://github.com/instructure/canvas-lms/wiki)

Installation
=======

Detailed instructions for installation and configuration of Canvas are provided
on our wiki.

 * [Quick Start](http://github.com/instructure/canvas-lms/wiki/Quick-Start)
 * [Production Start](http://github.com/instructure/canvas-lms/wiki/Production-Start)

Quiz Logs
=======

Canvas records quiz log data from both the quiz-taking front end and the quiz
submission pipeline.

- Client-side event capture: the quiz-taking page registers event trackers
  (page focus/blur, question viewed, question flagged, session started). Events
  are buffered in localStorage, then batched and posted every ~15 seconds to the
  quiz submission events endpoint as JSON (`quiz_submission_events` with
  `event_type`, `event_data`, and `client_timestamp`).
- Server-side answer capture: when quiz answers are backed up during a session,
  Canvas extracts `question_answered` events from the submission data and stores
  them with the current attempt.
- Storage: all events are persisted as `Quizzes::QuizSubmissionEvent` records
  containing server `created_at`, `client_timestamp`, `event_type`, `event_data`,
  and the submission attempt.
