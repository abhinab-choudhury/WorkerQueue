# WorkQueue
> Distributed Background Task Processing System

WorkQueue is a decoupled, asynchronous job-processing platform built with **Spring Boot** and **Redis**. Clients submit jobs to a REST API; a queue relays them to a pool of workers that dispatch each job to a specialized handler service.

The system is deliberately split into small, independently deployable microservices so that job ingestion, queue consumption, and task execution (email / image / video) can scale and fail on their own.

---

## Architecture at a Glance

```
                        ┌─────────────────────────────┐
                        │           Clients           │
                        │  POST /api/jobs             │
                        └──────────────┬──────────────┘
                                       │ HTTP
                                       ▼
                 ┌─────────────────────────────────────────┐
                 │              producer-service            │
                 │  REST API · request validation · enqueue │
                 └──────────────────────┬──────────────────┘
                                        │ LPUSH (Redis List)
                                        ▼
                        ┌─────────────────────────────┐
                        │           REDIS             │
                        │   work-queue (job messages) │
                        └─────────────────────────────┘
                                        │ BRPOP (blocking pop)
                                        ▼
                 ┌─────────────────────────────────────────┐
                 │              worker-service              │
                 │  queue consumer · type routing · threads │
                 │  WebClient (HTTP dispatch to handlers)   │
                 └───────┬───────────────┬──────────┬───────┘
                         │               │          │
                         ▼               ▼          ▼
                 ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
                 │ email-service│ │image-service│ │video-service│
                 │  send email  │ │  process    │ │  FFmpeg     │
                 │              │ │  image      │ │  encoding   │
                 └─────────────┘ └─────────────┘ └─────────────┘
```

## System Components

| Service            | Role                                                        | Stack highlights                      |
| ------------------ | ----------------------------------------------------------- | ------------------------------------- |
| **producer-service** | Accepts job requests via REST, validates them, pushes job messages onto a Redis queue. | Spring WebMVC, Validation, Redis      |
| **worker-service**   | Consumes jobs from the Redis queue with thread handling and routes each job to the correct handler over HTTP. | Spring WebMVC, Redis, WebClient       |
| **email-service**    | Handles `EMAIL` jobs — sends email (optional SMTP/mail support). | Spring WebMVC, mail (optional)        |
| **image-service**    | Handles `IMAGE` jobs — performs image processing via libraries. | Spring WebMVC, image libs             |
| **video-service**    | Handles `VIDEO` jobs — performs external processing via FFmpeg. | Spring WebMVC, FFmpeg (external)      |
| **Redis**            | Shared backbone — a durable in-memory queue decoupling producers from consumers. | Redis List / Stream                   |

## Job Lifecycle

1. **Submit** — a client calls the `producer-service` REST API with a job payload.
2. **Validate** — the payload is validated and normalized (unknown job types rejected).
3. **Enqueue** — a job message is pushed onto the Redis `work-queue` list.
4. **Consume** — the `worker-service` blocks on the queue (BRPOP), fetching one job at a time.
5. **Route** — the worker inspects the job `type` and dispatches it to the matching handler service via HTTP.
6. **Execute** — the handler service (email / image / video) performs the actual work and returns the outcome to the worker.

## Design Principles

- **Decoupling** — producers and consumers never talk directly; Redis is the only shared dependency.
- **Independent scaling** — each service scales horizontally on its own (more producers, more workers, more handlers).
- **Type-based routing** — adding a new job category = adding a handler service + a routing rule; no producer changes.
- **Isolation** — handler failures do not block ingestion; a job can be requeued or failed independently.
- **Standalone builds** — every service is an independent Gradle project with its own wrapper, build, and lifecycle.

## Project Structure

```
My-WorkQueue/
├── producer-service/        # REST API · job intake · Redis producer
├── worker-service/          # Redis consumer · routing · HTTP dispatch
├── email-service/           # EMAIL job handler
├── image-service/           # IMAGE job handler
├── video-service/           # VIDEO job handler (FFmpeg)
├── docs/
│   └── architecture.md      # Deep dive into design, data model, flows
└── infra/
    ├── docker-compose.yml   # Redis + all services in one command
    └── redis/
        └── redis.conf       # Redis server configuration
```

Each service directory is self-contained: `build.gradle`, `gradlew`, `settings.gradle`, and `src/` (no shared root build — services deploy independently).

## Getting Started

See [`docs/architecture.md`](docs/architecture.md) for the full design.

```bash
# 1. Start infrastructure (Redis)
docker compose -f infra/docker-compose.yml up redis

# 2. Start the handler services
./email-service/gradlew -p email-service bootRun
./image-service/gradlew -p image-service bootRun
./video-service/gradlew -p video-service bootRun

# 3. Start the worker (queue consumer)
./worker-service/gradlew -p worker-service bootRun

# 4. Start the producer (REST API)
./producer-service/gradlew -p producer-service bootRun
```

`docker compose -f infra/docker-compose.yml up` starts everything (Redis + all five services) at once.

## Roadmap

- [ ] Dead-letter queue (DLQ) for repeatedly failing jobs
- [ ] Retry policy with exponential backoff
- [ ] Idempotency keys to make handlers safe on replay
- [ ] Job status tracking (queued → running → done/failed)
- [ ] Queue monitoring (depth, latency, per-type throughput)

## License

[MIT](LICENSE.md)
