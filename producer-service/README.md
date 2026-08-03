# producer-service
> Job intake — REST API | Validation | Redis enqueue

The **producer** side of WorkQueue. Accepts job submissions over HTTP, validates them, and pushes a job message onto the Redis work queue for the [worker-service](../worker-service) to consume.

## Role in the architecture

```
Client ──▶ producer-service ──LPUSH──▶ Redis (work-queue) ──BRPOP──▶ worker-service
```

- Exposes a REST endpoint (e.g. `POST /api/jobs`) to submit jobs.
- Validates the payload (job `type`, required fields) and rejects unknown types.
- Returns `202 Accepted` after enqueuing; processing happens asynchronously.
- Stack: Spring WebMVC, Validation, `spring-boot-starter-data-redis`.

## Run

```bash
./gradlew bootRun          # needs Redis at localhost:6379
# or containerized:
docker compose -f ../infra/docker-compose.yml up producer
```

See [docs/architecture.md](../docs/architecture.md) for the full design.
