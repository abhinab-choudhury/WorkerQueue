# Architecture

## 1. Goals

WorkQueue processes long-running, fire-and-forget tasks (sending emails, resizing images, transcoding videos) without blocking API callers. The design separates three concerns:

1. **Ingestion** — collect and validate job requests (`producer-service`).
2. **Coordination** — move jobs from ingestion to execution asynchronously and reliably (Redis + `worker-service`).
3. **Execution** — perform type-specific work in isolated handlers (`email-service`, `image-service`, `video-service`).

## 2. Components

```
producer-service ──▶ Redis ──▶ worker-service ──▶ { email-service, image-service, video-service }
```

### 2.1 producer-service
- Exposes a REST API to submit jobs (e.g. `POST /api/jobs`).
- Validates payloads with Bean Validation (job type, payload shape) and rejects unknown types.
- Enqueues accepted jobs onto the Redis work queue (`LPUSH work-queue <job-json>`).
- Returns `202 Accepted` immediately — job processing happens asynchronously.
- Stack: Spring WebMVC, Validation, `spring-boot-starter-data-redis`.

### 2.2 Redis (work queue)
- A single Redis List (`work-queue`) acts as the message bus between producers and workers.
- Producers `LPUSH`; workers `BRPOP` (blocking right-pop) so idle workers sleep instead of polling.
- Using a List gives FIFO ordering and atomic, blocking pop semantics out of the box.
- *Upgrade path:* a Redis Stream (`XADD` / `XREADGROUP`) adds consumer groups, per-message acknowledgements, and replay, which are useful once retries/at-least-once delivery are introduced.

### 2.3 worker-service
- Blocks on the queue (`BRPOP`) to claim one job at a time.
- Maintains a thread pool so multiple jobs are processed concurrently.
- Parses the job message and inspects its `type` field.
- Routes via an HTTP call (Spring WebClient) to the handler that owns that type:

| job `type` | handler service  | example endpoint   |
| ---------- | ---------------- | ------------------ |
| `EMAIL`    | `email-service`  | `POST /api/email`  |
| `IMAGE`    | `image-service`  | `POST /api/image`  |
| `VIDEO`    | `video-service`  | `POST /api/video`  |

- On a successful handler response the job is considered done; on failure it can be retried or parked (see §5).

### 2.4 email-service
- Receives `EMAIL` jobs over REST.
- Composes and sends the email (SMTP support optional).
- Returns a success/failure response to the worker.

### 2.5 image-service
- Receives `IMAGE` jobs over REST.
- Runs image processing (resize, format conversion, watermarking) via a processing library.
- Returns a success/failure response to the worker.

### 2.6 video-service
- Receives `VIDEO` jobs over REST.
- Invokes FFmpeg (external binary) for encoding/transcoding.
- Returns a success/failure response to the worker.

## 3. Data Model

### 3.1 Job message (on the Redis queue)

```json
{
  "jobId":    "job_01H3XYZ",
  "type":     "IMAGE",
  "payload":  {
    "sourcePath": "s3://bucket/input.png",
    "width": 1280,
    "height": 720
  },
  "createdAt": "2026-08-03T12:00:00Z"
}
```

| Field       | Purpose                                            |
| ----------- | -------------------------------------------------- |
| `jobId`     | Unique identifier, used for tracing and idempotency |
| `type`      | Routing key → selects the handler service          |
| `payload`   | Type-specific work parameters (opaque to the worker) |
| `createdAt` | Submission timestamp for latency monitoring        |

### 3.2 Redis keys

| Key             | Type | Owner              | Purpose                    |
| --------------- | ---- | ------------------ | -------------------------- |
| `work-queue`    | List | producer / worker  | Pending jobs (FIFO)        |

## 4. Request Flow (sequence)

```
Client            producer            Redis            worker            email/image/video
  │ POST /api/jobs  │                  │                 │                        │
  ├────────────────▶│ validate payload │                 │                        │
  │                 │ LPUSH job       │                 │                        │
  │                 ├────────────────▶│                 │                        │
  │ 202 Accepted    │                  │                 │                        │
  │◀────────────────┤                  │                 │                        │
  │                 │                  │ BRPOP (block)   │                        │
  │                 │                  ├────────────────▶│                        │
  │                 │                  │   job message   │                        │
  │                 │                  │◀────────────────┤                        │
  │                 │                  │                 │  HTTP dispatch by type  │
  │                 │                  │                 ├───────────────────────▶│
  │                 │                  │                 │  execute job            │
  │                 │                  │                 │◀────────────────────────│
  │                 │                  │                 │   result                │
```

## 5. Failure Handling

| Failure                     | Behavior                                                   |
| --------------------------- | ---------------------------------------------------------- |
| Invalid job submitted       | Rejected at the producer (validation) — never enqueued     |
| Worker crashes mid-job      | Job is lost from the list pop (handled by re-queue on restart) |
| Handler service is down     | Worker gets a connection error → retry policy or DLQ       |
| Handler returns failure     | Worker logs and applies retry/backoff policy               |
| Unknown job `type`          | Worker rejects the message (no handler registered)         |

*Planned:* dead-letter queue, retries with exponential backoff, idempotency keys, and job-status tracking (see README Roadmap).

## 6. Scaling Model

| Service          | Scales on                                             |
| ---------------- | ----------------------------------------------------- |
| producer-service | HTTP traffic (horizontal API instances)               |
| worker-service   | Queue depth (more consumers → more parallelism)       |
| email/image/video| Handler throughput (CPU/IO-bound work)                |

Because handlers are only ever called by the worker, they can each be scaled without affecting ingestion. Redis is the single coordination point and is itself horizontally scalable (clustering/replication).

## 7. Configuration Reference

| Setting            | Example value        | Where                       |
| ------------------ | -------------------- | --------------------------- |
| Redis host         | `localhost:6379`     | every service (via env)     |
| Queue key          | `work-queue`         | producer + worker           |
| Handler base URLs  | `http://localhost:8081` | worker-service           |
| Mail server (SMTP) | `smtp.example.com`   | email-service (optional)    |
| FFmpeg binary path | `ffmpeg`             | video-service               |
