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

Implementation at a glance (copy/paste friendly):

```js
// 1) Define event types and constants.
const EVENT_TYPES = {
  PAGE_FOCUSED: 'page_focused',
  PAGE_BLURRED: 'page_blurred',
  QUESTION_VIEWED: 'question_viewed',
  QUESTION_FLAGGED: 'question_flagged',
  SESSION_STARTED: 'session_started',
}
const STORAGE_KEY = 'qla_events'
const DELIVERY_INTERVAL_MS = 15000

// 2) Basic event object format.
function makeEvent(eventType, eventData) {
  return {
    event_type: eventType,
    event_data: eventData ?? null,
    client_timestamp: new Date().toISOString(),
  }
}

// 3) Buffer (localStorage-backed).
function loadBuffer() {
  try {
    return JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]')
  } catch {
    return []
  }
}
function saveBuffer(buffer) {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(buffer))
  } catch {
    // Ignore quota errors; you can add telemetry here.
  }
}

// 4) Enqueue + de-dupe example for page focus.
let lastEventType = null
function enqueueEvent(buffer, event) {
  if (event.event_type === EVENT_TYPES.PAGE_FOCUSED && lastEventType !== EVENT_TYPES.PAGE_BLURRED) {
    return
  }
  buffer.push(event)
  lastEventType = event.event_type
  saveBuffer(buffer)
}

// 5) Delivery loop.
async function deliverEvents(buffer, endpointUrl) {
  if (!buffer.length) return
  const payload = {quiz_submission_events: buffer}
  const res = await fetch(endpointUrl, {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify(payload),
    credentials: 'include',
  })
  if (res.ok) {
    buffer.length = 0
    saveBuffer(buffer)
  }
}
```

End-to-end walkthrough (client + server):

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

Client capture details (Canvas implementation):

- Buffering strategy: `EventBuffer` loads/saves events in localStorage under
  `qla_events` (`ui/shared/quiz-log-auditing/jquery/event_buffer.js`).
- Delivery strategy: `EventManager` batches pending events and POSTs to
  `ENV.QUIZ_SUBMISSION_EVENTS_URL` every 15000 ms
  (`ui/shared/quiz-log-auditing/jquery/event_manager.js`).
- Trackers are installed by `ui/shared/quiz-log-auditing/jquery/log_auditing.js`
  and run on the quiz-taking page only.

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

Server storage model (Canvas implementation):

```rb
# app/models/quizzes/quiz_submission_event.rb
class Quizzes::QuizSubmissionEvent < ActiveRecord::Base
  # event_type: string
  # event_data: JSON
  # client_timestamp: datetime
  # created_at: datetime (server time)
  # attempt: integer
  # quiz_submission_id: integer
end
```

Server ingest (Canvas implementation):

```rb
# app/controllers/quizzes/quiz_submission_events_api_controller.rb
def create
  if authorized_action(@quiz_submission, @current_user, :record_events)
    params["quiz_submission_events"]&.each do |datum|
      Quizzes::QuizSubmissionEvent.create do |event|
        event.quiz_submission_id = @quiz_submission.id
        event.event_type = datum["event_type"]
        event.event_data = datum["event_data"]
        event.client_timestamp = datum["client_timestamp"]
        event.attempt = @quiz_submission.attempt
      end
    end
    head :no_content
  end
end
```

Server-side answer extraction (Canvas implementation):

```rb
# app/models/quizzes/log_auditing/question_answered_event_extractor.rb
# Extracts question_<id> fields from the submission payload, serializes answers,
# and stores one event per backup if new answers were found.
def create_event!(submission_data, quiz_submission)
  event = build_event(submission_data, quiz_submission)
  predecessors = Quizzes::QuizSubmissionEvent.where(SQL_FIND_PREDECESSORS, {
    quiz_submission_id: quiz_submission.id,
    attempt: event.attempt,
    started_at: quiz_submission.started_at,
    created_at: event.created_at
  }).order(created_at: :desc)

  if predecessors.any?
    optimizer = Quizzes::LogAuditing::QuestionAnsweredEventOptimizer.new
    optimizer.run!(event.answers, predecessors)
  end

  event.tap(&:save!) if event.answers.any?
end
```

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

If you are implementing this in another website:

1. Build a client event pipeline with:
   - A small set of event types
   - A local buffer (memory + localStorage fallback)
   - A batch delivery loop (15s is a reasonable starting point)
2. Define a stable event schema (`event_type`, `event_data`, `client_timestamp`)
   and version it if you expect to evolve it.
3. Send events to a server endpoint that:
   - Authenticates the user/session
   - Stamps server time (`created_at`)
   - Stores the attempt or session context
4. (Optional but recommended) Parse answer data server-side on backup to record
   authoritative `question_answered` events, which are harder to spoof than
   client-reported answer changes.
