# SyncBridge

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Python](https://img.shields.io/badge/Python-3.10%2B-blue)](#)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115.6-009688)](https://fastapi.tiangolo.com/)
[![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-2.0.36-red)](https://www.sqlalchemy.org/)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![Made with SQL](https://img.shields.io/badge/queue-database--backed-informational)](#)

SyncBridge is a backend-focused reliability project: a small reference implementation for durable integration jobs, retries, worker leases, dead-letter handling, replay, and operational visibility.

The demo syncs records from a mock CRM into a mock Billing system. The point is not the integration itself; the point is showing real failure handling in a codebase a reviewer can read end to end, instead of only demonstrating happy-path API CRUD.

## Quick Reviewer Path

Use this path if you have about five minutes and want to understand the project shape quickly.

Problem SyncBridge solves: reliable background sync between external systems when upstream failures, duplicate requests, retries, and operator debugging matter.

1. Start with the core flow:
   - API entry points: `app/routes/jobs.py`
   - Job creation, idempotency, retry, cancel, replay: `app/services/job_service.py`
   - Worker leasing and claiming: `app/jobs/worker.py`
   - Attempt execution, backoff, DLQ transitions: `app/jobs/executor.py`
   - Job and attempt persistence: `app/models/sync_job.py`, `app/models/sync_job_attempt.py`

2. Run the service:

```bash
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Open:

- API docs: http://127.0.0.1:8000/docs
- Admin UI: http://127.0.0.1:8000/ui/jobs
- Metrics: http://127.0.0.1:8000/metrics

3. Enqueue a normal customer sync:

```bash
curl -X POST http://127.0.0.1:8000/api/jobs/customer \
  -H "Content-Type: application/json" \
  -d '{"entity_id":"c_1001"}'
```

4. Trigger retryable failures:

```bash
curl -X POST http://127.0.0.1:8000/api/jobs/customer \
  -H "Content-Type: application/json" \
  -d '{"entity_id":"c_flaky"}'
```

`c_flaky` causes an intermittent CRM `503`, which is classified as retryable.

```bash
curl -X POST http://127.0.0.1:8000/api/jobs/customer \
  -H "Content-Type: application/json" \
  -d '{"entity_id":"c_1002"}'
```

`c_1002` causes Billing `429` rate limits and eventually demonstrates the retry budget and dead-letter behavior.

5. Inspect operations:
   - Job list: `GET /api/jobs` or http://127.0.0.1:8000/ui/jobs
   - Job detail: `GET /api/jobs/{id}` or `/ui/jobs/{id}`
   - Attempts: `GET /api/jobs/{id}/attempts`
   - Retry a non-retryable failure: `POST /api/jobs/{id}/retry`
   - Replay a failed attempt: `POST /api/jobs/{id}/replay`
   - DLQ/dead jobs: filter the UI with `/ui/jobs?status=dead`
   - Metrics: `GET /metrics`

6. Use the screenshots as evidence:
   - API docs prove the HTTP surface is discoverable.
   - Enqueue response proves jobs are durable records with status, attempts, and correlation IDs.
   - Job list proves operational state is visible.
   - Job detail proves attempt history and error classification are stored.
   - Retryable failure logs prove classification, retry scheduling, and correlation data are logged.
   - DLQ screenshot proves retry budget exhaustion leads to `dead`.
   - Replay screenshot proves replay creates a new linked job instead of rewriting history.
   - Metrics screenshot proves aggregate health signals are exposed.

## What This Project Proves

SyncBridge demonstrates backend reliability patterns that show up in real integration systems:

- Durable database-backed job processing, using SQL as the source of truth.
- Worker leases with `lease_owner` and `lease_expires_at` to avoid duplicate execution and recover expired claims.
- Retryable vs non-retryable error classification.
- Exponential backoff for retryable failures.
- Dead-letter behavior when retryable work exhausts `max_retries`.
- Replay that creates a new linked job and preserves the original failure history.
- Idempotency for job creation by rejecting duplicate active jobs for the same `(job_type, entity_id)`.
- Correlation IDs propagated to mock integration clients and structured logs.
- Operational visibility through job list/detail pages, attempt records, logs, and the `/metrics` endpoint.
- Clear tradeoffs against larger systems such as Celery, Temporal, Kafka, and RabbitMQ.

This is intentionally small. The value is that the reliability mechanics are explicit and reviewable.

## Job Lifecycle

The code uses `pending`, `running`, `success`, `failed`, `dead`, and `canceled` as stored status values. In reviewer terms:

- Queued means a `pending` job that is due to run.
- Leased/running means a worker has claimed the job, set `running`, and written lease metadata.
- Retry scheduled means the job is back to `pending` with `next_run_at` set in the future.
- Dead means a retryable failure exceeded the retry budget.

```text
enqueue
  |
  v
queued (pending, due now)
  |
  | worker claim writes lease_owner + lease_expires_at
  v
leased/running (running)
  |                     |
  | success             | retryable failure within budget
  v                     v
succeeded          retry_scheduled
                    (pending + future next_run_at)
                         |
                         | due again
                         v
                    leased/running

retryable failure beyond max_retries
  |
  v
dead
  |
  | replay
  v
new pending job linked by replay_of_job_id and replay_of_attempt_id
```

Non-retryable failures become `failed`. Those can be manually retried with `POST /api/jobs/{id}/retry`, which moves the job back to `pending`.

## Architecture

- `app/routes/` exposes HTTP APIs for enqueueing, inspecting, retrying, canceling, replaying, mock integrations, and metrics.
- `app/services/` owns job creation and control operations.
- `app/jobs/` contains the in-process worker loop, executor, handlers, and registry.
- `app/models/` defines SQLAlchemy models for `SyncJob` and `SyncJobAttempt`.
- `app/integrations/` wraps external calls and converts upstream failures into typed integration errors.
- `app/ui/` provides a small server-rendered admin UI for job inspection.
- `app/logging/` configures structured JSON logs.

## Key Design Decisions

### Database as the source of truth

Jobs and attempts are stored in SQL through SQLAlchemy. SQLite is the default local database, so the queue is easy to run and inspect during review.

### Leases for safe claiming

The worker claims one due job at a time and writes:

- `status=running`
- `lease_owner`
- `lease_acquired_at`
- `lease_expires_at`

If a worker dies mid-run, an expired `running` job can be claimed again.

### Typed error classification

External failures are classified in `app/jobs/executor.py`:

- `UpstreamTimeout`: retryable, used for missing status codes or `5xx`
- `UpstreamRateLimited`: retryable, used for `429`
- `NotFound`: non-retryable, used for `404`
- `ValidationError`: non-retryable fallback

Retryable failures are scheduled with exponential backoff:

```text
delay = SYNCBRIDGE_JOB_BACKOFF_SECONDS_BASE * 2^(attempt_count - 1)
```

### Dead-letter behavior

When a retryable failure exceeds `max_retries`, the job moves to `dead`. The final error summary and type are stored on the job as `dead_error` and `dead_error_type`, while every attempt remains in `sync_job_attempts`.

### Replay preserves history

Replay does not mutate the original job. It creates a new `pending` job with:

- `is_replay=true`
- `replay_of_job_id`
- `replay_of_attempt_id`

That keeps the failed job and its attempts available for audit and debugging.

### Idempotent active job creation

Creating a job checks for an existing active job with the same `(job_type, entity_id)`. If one exists in `pending` or `running`, the API returns `409` with the existing job ID instead of creating duplicate active work.

### Correlation IDs

Each job gets a `correlation_id`, which is propagated as `X-Correlation-ID` to the mock CRM and Billing clients and included in job execution logs.

## Quickstart

### Prerequisites

- Python 3.10+
- pip

### 1. Create a virtual environment

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Run the service

```bash
uvicorn app.main:app --reload
```

Open:

- Admin UI: http://127.0.0.1:8000/ui/jobs
- API docs: http://127.0.0.1:8000/docs
- Metrics: http://127.0.0.1:8000/metrics

## Demo Commands

### Enqueue customer sync

```bash
curl -X POST http://127.0.0.1:8000/api/jobs/customer \
  -H "Content-Type: application/json" \
  -d '{"entity_id":"c_1001"}'
```

### Enqueue invoice sync

```bash
curl -X POST http://127.0.0.1:8000/api/jobs/invoice \
  -H "Content-Type: application/json" \
  -d '{"entity_id":"i_2001"}'
```

### Trigger retryable failures

- `c_flaky`: intermittent CRM `503`; useful for seeing retry and eventual success.
- `i_flaky`: intermittent CRM `503`; invoice version of the same behavior.
- `c_1002`: Billing `429`; useful for exhausting retries and observing `dead`.
- `i_2002`: Billing `429`; invoice version of the same behavior.

```bash
curl -X POST http://127.0.0.1:8000/api/jobs/customer \
  -H "Content-Type: application/json" \
  -d '{"entity_id":"c_1002"}'
```

### Trigger non-retryable failure

Unknown CRM IDs return `404`, which is classified as `NotFound` and marked `failed`.

```bash
curl -X POST http://127.0.0.1:8000/api/jobs/customer \
  -H "Content-Type: application/json" \
  -d '{"entity_id":"c_missing"}'
```

### Inspect jobs and attempts

```bash
curl http://127.0.0.1:8000/api/jobs
curl http://127.0.0.1:8000/api/jobs/1
curl http://127.0.0.1:8000/api/jobs/1/attempts
curl http://127.0.0.1:8000/metrics
```

### Retry, cancel, and replay

```bash
curl -X POST http://127.0.0.1:8000/api/jobs/1/retry
curl -X POST http://127.0.0.1:8000/api/jobs/1/cancel
curl -X POST http://127.0.0.1:8000/api/jobs/1/replay \
  -H "Content-Type: application/json" \
  -d '{"attempt_id":null}'
```

## Metrics

`GET /metrics` returns:

- `total_jobs`
- `finished_jobs`
- `success_rate`
- `retry_count`
- `avg_execution_ms`

These are intentionally simple health signals for local review, not a replacement for production monitoring or alerting.

## Screenshots

All screenshots are from a live local run.

### API docs

Shows the FastAPI OpenAPI surface for job creation, job controls, inspection, mock integrations, and metrics.

![API docs](docs/screenshots/01-api-docs.png)

### Enqueue response

Shows that enqueueing creates a durable job record with status, retry settings, attempt count, and correlation ID.

![Enqueue response](docs/screenshots/02-enqueue-job-response.png)

### Job list

Shows the admin UI listing jobs across states, including quick access to dead jobs.

![Job list](docs/screenshots/03-ui-job-list.jpeg)

### Job detail with attempts

Shows one job with execution metadata, error fields, and the attempt history used for debugging.

![Job detail with attempts](docs/screenshots/04-ui-job-detail-attempts.png)

### Retryable failure logs

Shows structured worker logs for retryable failures, attempt numbers, job IDs, and correlation IDs.

![Retryable failure logs](docs/screenshots/05-retryable-failure-logs.jpeg)

### DLQ/dead job

Shows a job marked `dead` after the retry budget is exhausted.

![DLQ dead job](docs/screenshots/07-dlq-dead-job.png)

### Replay creates a new job

Shows replay producing a new job linked to the failed job and attempt, while preserving the original history.

![Replay creates a new job](docs/screenshots/10-replay-creates-new-job.png)

### Metrics endpoint

Shows aggregate job counts, retry count, success rate, and average execution duration.

![Metrics endpoint](docs/screenshots/11-metrics-endpoint.png)

## Tradeoffs Compared With Queue and Workflow Systems

SyncBridge is deliberately smaller than Celery, Temporal, Kafka, or RabbitMQ.

- Compared with Celery or Sidekiq, jobs live directly in the application database and the worker is in-process. That makes the mechanics easy to inspect, but it does not provide a mature distributed worker ecosystem.
- Compared with Temporal, this project demonstrates job durability and replay links, but it does not implement durable workflow histories, timers, signals, activities, or deterministic workflow execution.
- Compared with Kafka, it is not an event log, stream processor, or high-throughput message backbone.
- Compared with RabbitMQ, it does not provide broker-level routing, acknowledgements, exchanges, or queue topology.

The tradeoff is intentional: this repo favors clarity over infrastructure breadth.

## Current Scope / Honest Limitations

- This is not a full distributed queue.
- This is not a Temporal, Celery, Kafka, or RabbitMQ replacement.
- The worker is intentionally small and in-process.
- SQLite is used by default for local demonstration.
- The lease approach is educational and reviewable, not a complete multi-node concurrency strategy.
- There is no production migration framework in this repo.
- Production use would require stronger infrastructure, database locking/concurrency choices, monitoring, alerting, deployment hardening, operational runbooks, and failure-mode testing.

## Notes

- The database is the queue.
- Attempts are first-class records, not just log lines.
- Replay creates new work instead of rewriting old work.
- The project is optimized for backend reviewers who care about reliability behavior and operational clarity.
