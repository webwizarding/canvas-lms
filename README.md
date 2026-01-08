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
submission pipeline. This is the exact capture outline so you can implement the
same pattern in another web app.

1. Boot the quiz log client on the quiz-taking page. Canvas wires
   `ui/shared/quiz-log-auditing/jquery/log_auditing.js` and registers trackers,
   configures localStorage buffering, and sets the delivery URL from
   `ENV.QUIZ_SUBMISSION_EVENTS_URL`.
2. Start capture. `EventManager` creates an `EventBuffer`, installs each tracker
   (page focus/blur, question viewed, question flagged, session started), and
   begins a delivery loop that runs every 15000 ms (`autoDeliveryFrequency`).
3. Trackers emit events. Each tracker calls a delivery callback with its event
   payload. A `QuizEvent` is created with `event_type`, `event_data`, and
   `client_timestamp` (recorded locally at capture time).
4. Buffer and persist. Each event is appended to the buffer and serialized into
   localStorage under the key `qla_events`. This preserves events across reloads
   until they are delivered.
5. Batch delivery. Every 15000 ms, the client POSTs JSON to the submission
   events endpoint (`/courses/:course_id/quizzes/:quiz_id/submissions/:id/events`)
   with a `quiz_submission_events` array. If delivery succeeds, the buffer
   discards the batch. If delivery fails, the events are marked pending and will
   be retried on the next batch. Consecutive `page_focused` events are ignored
   unless a `page_blurred` occurred in between.
6. Server persistence. `Quizzes::QuizSubmissionEventsApiController#create`
   iterates the array and inserts a `Quizzes::QuizSubmissionEvent` record for
   each item with `quiz_submission_id`, `event_type`, `event_data`,
   `client_timestamp`, and `attempt`. `created_at` is set server-side to support
   partitioning.
7. Server-side answer capture. When the quiz-taking UI backs up answers via
   `Quizzes::QuizSubmissionsController#backup`, `QuizSubmission#record_answer`
   extracts `question_answered` events from the submission payload. It parses
   `question_<id>` fields, serializes full answers, removes duplicates against
   previous events in the same attempt, and stores the event if it contains new
   answers.
8. Retrieval. `Quizzes::QuizSubmissionEventsApiController#index` returns events
   for a specific attempt ordered by server `created_at` and filtered to events
   created after the attempt started.

Event types and payloads captured by the client:
- `page_focused`: window focus, throttled to 5000 ms, `event_data` null
- `page_blurred`: window blur, throttled to 5000 ms, `event_data` null
- `question_viewed`: scroll-based visibility detection, throttled to 2500 ms,
  `event_data` is an array of quiz question ids
- `question_flagged`: click on `.flag_question`, `event_data` includes
  `questionId` and `flagged`
- `session_started`: page load on quiz take URL, `event_data` includes
  `user_agent`

Event types captured server-side:
- `question_answered`: derived from backup payloads, `event_data` is an array of
  `{ "quiz_question_id": "<id>", "answer": <serialized answer> }`

Example client payload:

```json
{
  "quiz_submission_events": [
    {
      "client_timestamp": "2024-01-01T12:00:00Z",
      "event_type": "page_focused",
      "event_data": null
    },
    {
      "client_timestamp": "2024-01-01T12:00:05Z",
      "event_type": "question_viewed",
      "event_data": ["12", "14"]
    },
    {
      "client_timestamp": "2024-01-01T12:00:10Z",
      "event_type": "question_flagged",
      "event_data": { "questionId": "12", "flagged": true }
    },
    {
      "client_timestamp": "2024-01-01T12:00:12Z",
      "event_type": "session_started",
      "event_data": { "user_agent": "Mozilla/5.0 ..." }
    }
  ]
}
```

Example server-extracted `question_answered` event payload:

```json
{
  "event_type": "question_answered",
  "event_data": [
    { "quiz_question_id": "12", "answer": "A" },
    { "quiz_question_id": "14", "answer": ["B", "C"] }
  ]
}
```

Storage fields for each event:
`quiz_submission_id`, `attempt`, `event_type`, `event_data`, `client_timestamp`,
and server `created_at`.
