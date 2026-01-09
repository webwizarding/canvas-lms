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

Quiz Log Auditing Overview 
=====================================

Canvas captures fine-grained quiz activity using a client-side event pipeline
plus server-side ingestion and aggregation. The pipeline is designed to preserve
events across page reloads, batch them efficiently, and provide instructors with
an audit trail that includes both raw client events and server-derived answer
changes.

Data model and persistence
--------------------------

Events from the browser are stored as `Quizzes::QuizSubmissionEvent` records. Each
row includes the quiz submission id, attempt number, event type, event data as
JSON, the client-side timestamp captured at the moment of the event, and a
server-side `created_at` timestamp. The server timestamp is also used for
partitioning, which keeps monthly chunks manageable for large installations.
Partitioning is handled by `Quizzes::QuizSubmissionEventPartitioner` using the
Canvas Partman framework, which creates and prunes monthly partitions so quiz log
queries remain efficient.【F:app/models/quizzes/quiz_submission_event.rb†L1-L44】【F:app/models/quizzes/quiz_submission_event_partitioner.rb†L1-L48】

Server endpoints and access control
-----------------------------------

Client events are written through `Quizzes::QuizSubmissionEventsApiController#create`.
The controller validates access, iterates the `quiz_submission_events` array from
the client payload, and inserts one row per event. Reads are served by the same
controller’s `index` action, which enforces the `quiz_log_auditing` feature flag,
filters by quiz attempt, and returns events ordered by server `created_at` so the
instructor view can reconstruct the timeline.【F:app/controllers/quizzes/quiz_submission_events_api_controller.rb†L1-L116】

Server-side enrichment and aggregation
--------------------------------------

Canvas does not rely solely on client-reported answer changes. The
`QuestionAnsweredEventExtractor` rebuilds `question_answered` events from quiz
submission snapshots posted via the backup pipeline. It parses `question_<id>`
fields, serializes full answers, and only emits events when new answer data is
found. The `QuestionAnsweredEventOptimizer` then deduplicates and prunes answer
events so repeated submissions do not bloat the log, and the `EventAggregator`
keeps only the most recent answer per question when materializing the
`submission_data` structure used by instructor-facing audit views and reports.
Together these classes convert raw events into a readable audit timeline.
【F:app/models/quizzes/log_auditing/question_answered_event_extractor.rb†L1-L103】【F:app/models/quizzes/log_auditing/question_answered_event_optimizer.rb†L1-L52】【F:app/models/quizzes/log_auditing/event_aggregator.rb†L1-L101】

Client boot sequence
--------------------

When the quiz-taking page renders, the Rails view exposes
`QUIZ_SUBMISSION_EVENTS_URL`. The quiz bundle reads this value, boots the
`QuizLogAuditing` singleton, and forces a buffer flush so the browser can start
streaming events to the API as soon as tracking begins.【F:ui/features/take_quiz/jquery/index.js†L1060-L1063】
`QuizLogAuditing` wires up a single `EventManager`, registers tracker factories,
forces `EventBuffer` to use `localStorage`, and configures the delivery URL from
the environment. Unauthorized responses trigger a reload to recover a valid
session.【F:ui/shared/quiz-log-auditing/jquery/log_auditing.js†L19-L53】

Event manager and delivery loop
-------------------------------

`EventManager` wraps tracker emissions in `QuizEvent` objects, keeps recent events
in memory, and persists them via the buffer. It uses two delivery modes: some
events are sent immediately, while the rest are batched on a 15-second interval
(`autoDeliveryFrequency`). Both success and failure paths update the buffer so
pending events are retried automatically. `QuizEvent` always includes
`event_type`, a cloned copy of the tracker data, and a client-side timestamp that
represents when the activity occurred (distinct from the server timestamp).
【F:ui/shared/quiz-log-auditing/jquery/event_manager.js†L31-L211】【F:ui/shared/quiz-log-auditing/jquery/event.js†L23-L81】

Persistent buffering
--------------------

The `EventBuffer` is responsible for persistence and durability. It loads any
saved events from `localStorage` on startup, appends new events to an in-memory
queue, and writes the updated queue back to storage after each enqueue. If the
storage quota is exhausted, it degrades gracefully and continues in memory. The
buffer uses `EventSet` instances to track pending, in-flight, and delivered
batches, removing them once the POST succeeds and retaining them on failure.
【F:ui/shared/quiz-log-auditing/jquery/event_buffer.js†L24-L113】【F:ui/shared/quiz-log-auditing/jquery/event_set.js†L1-L44】

Trackers and event types
------------------------

Trackers are bound to the quiz-taking DOM only. They emit a small set of event
types defined in shared constants:
- `page_focused` and `page_blurred`: window focus changes, throttled to every
  five seconds so rapid tabbing does not spam the log. Blur events caused by
  iframe focus changes are ignored in some browsers.
- `question_viewed`: emitted when scroll-driven visibility detection finds a
  quiz question on screen; each question is only reported once.
- `question_flagged`: toggles on `.flag_question` and related UI elements,
  recording the question id and flag state.
- `session_started`: emitted on page load, typically with `user_agent` metadata.
These trackers are implemented in the quiz log auditing bundle and hooked up by
`log_auditing.js`.【F:ui/shared/quiz-log-auditing/jquery/event_trackers/page_focused.js†L22-L30】【F:ui/shared/quiz-log-auditing/jquery/event_trackers/page_blurred.js†L22-L42】【F:ui/shared/quiz-log-auditing/jquery/event_trackers/question_viewed.js†L26-L63】【F:ui/shared/quiz-log-auditing/jquery/event_trackers/question_flagged.js†L25-L48】【F:ui/shared/quiz-log-auditing/jquery/event_trackers/session_started.js†L23-L38】【F:ui/shared/quiz-log-auditing/jquery/constants.js†L18-L36】

Server-side answer capture
--------------------------

In addition to the client pipeline, the quiz-taking UI periodically posts
submission backups. The server extracts `question_answered` events from these
payloads by parsing `question_<id>` fields, serializing the answer content, and
deduplicating against earlier events in the same attempt. This provides a more
authoritative record of answer changes than client-side events alone.

Implementation pattern (simplified)
-----------------------------------

The following condensed outline mirrors the Canvas client pipeline and is useful
for implementing a similar system elsewhere:

```js
const EVENT_TYPES = {
  PAGE_FOCUSED: 'page_focused',
  PAGE_BLURRED: 'page_blurred',
  QUESTION_VIEWED: 'question_viewed',
  QUESTION_FLAGGED: 'question_flagged',
  SESSION_STARTED: 'session_started',
};
const STORAGE_KEY = 'qla_events';
const DELIVERY_INTERVAL_MS = 15000;

function makeEvent(eventType, eventData) {
  return {
    event_type: eventType,
    event_data: eventData ?? null,
    client_timestamp: new Date().toISOString(),
  };
}

function loadBuffer() {
  try {
    return JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
  } catch {
    return [];
  }
}

function saveBuffer(buffer) {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(buffer));
  } catch {
    // Ignore quota errors; consider telemetry here.
  }
}

let lastEventType = null;
function enqueueEvent(buffer, event) {
  if (event.event_type === EVENT_TYPES.PAGE_FOCUSED &&
      lastEventType !== EVENT_TYPES.PAGE_BLURRED) {
    return;
  }
  buffer.push(event);
  lastEventType = event.event_type;
  saveBuffer(buffer);
}

async function deliverEvents(buffer, endpointUrl) {
  if (!buffer.length) return;
  const payload = { quiz_submission_events: buffer };
  const res = await fetch(endpointUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
    credentials: 'include',
  });
  if (res.ok) {
    buffer.length = 0;
    saveBuffer(buffer);
  }
}
```

End-to-end flow (client + server)
---------------------------------

1. Boot the quiz log client on the quiz-taking page. The bundle wires
   `ui/shared/quiz-log-auditing/jquery/log_auditing.js`, registers trackers,
   configures localStorage buffering, and sets the delivery URL from
   `ENV.QUIZ_SUBMISSION_EVENTS_URL`.
2. Start capture. `EventManager` creates an `EventBuffer`, installs each tracker
   (page focus/blur, question viewed, question flagged, session started), and
   begins a delivery loop that runs every 15000 ms.
3. Trackers emit events. Each tracker calls a delivery callback with its event
   payload. A `QuizEvent` is created with `event_type`, `event_data`, and
   `client_timestamp`.
4. Buffer and persist. Each event is appended to the buffer and serialized into
   `localStorage` under `qla_events`. This preserves events across reloads until
   they are delivered.
5. Batch delivery. Every 15000 ms, the client POSTs JSON to the submission events
   endpoint with a `quiz_submission_events` array. If delivery succeeds, the
   buffer discards the batch; if delivery fails, events remain pending for retry.
6. Server persistence. The API creates a `Quizzes::QuizSubmissionEvent` record
   for each item with `quiz_submission_id`, `event_type`, `event_data`,
   `client_timestamp`, and `attempt`. Server `created_at` supports partitioning.
7. Server-side answer capture. When the quiz UI backs up answers via
   `Quizzes::QuizSubmissionsController#backup`, the server extracts
   `question_answered` events from the payload, deduplicates them, and stores
   only new answer data.
8. Retrieval. `Quizzes::QuizSubmissionEventsApiController#index` returns events
   for a specific attempt ordered by server `created_at` and filtered to events
   created after the attempt started.

Event payload examples
----------------------

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
-----------------------------------------------

1. Build a client event pipeline with a small set of event types, a local buffer,
   and a batch delivery loop (15s is a reasonable starting point).
2. Define a stable event schema (`event_type`, `event_data`, `client_timestamp`)
   and version it if you expect it to evolve.
3. Send events to a server endpoint that authenticates the user/session, stamps
   server time (`created_at`), and stores the attempt or session context.
4. Parse answer data server-side on backup to record authoritative
   `question_answered` events that are harder to spoof than client-reported
   changes.
