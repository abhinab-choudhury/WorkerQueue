# worker-service
> Queue consumer — Redis BRPOP | Type routing | HTTP dispatch (WebClient) | Thread handling

The **consumer** side of WorkQueue. Blocks on the Redis work queue, claims jobs one at a time, and routes each job to the handler service that owns its `type`.

## Role in the architecture

```
producer-service ──▶ Redis ──BRPOP──▶ worker-service ──HTTP──▶ { email, image, video }-service
```

- Consumes jobs from Redis (`BRPOP`) with a thread pool for concurrency.
- Inspects the job `type` and dispatches via Spring WebClient:

| job `type` | handler service                              |
| ---------- | -------------------------------------------- |
| `EMAIL`    | [email-service](../email-service)            |
| `IMAGE`    | [image-service](../image-service)            |
| `VIDEO`    | [video-service](../video-service)            |

- Stack: Spring WebMVC, Redis, WebClient.

## Run

```bash
./gradlew bootRun          # needs Redis at localhost:6379
# or containerized:
docker compose -f ../infra/docker-compose.yml up worker
```

Handler base URLs are configurable (e.g. `WORKER_EMAIL_URL`, `WORKER_IMAGE_URL`, `WORKER_VIDEO_URL`). See [docs/architecture.md](../docs/architecture.md) for the full design.
