# image-service
> `IMAGE` job handler — REST API | Image processing (via libraries)

One of the **handler** services in WorkQueue. Executes `IMAGE` jobs dispatched by the [worker-service](../worker-service) over HTTP.

## Role in the architecture

```
worker-service ──HTTP (POST /api/image)──▶ image-service
```

- Exposes a REST endpoint that receives `IMAGE` job payloads.
- Performs image processing (resize, format conversion, etc.) using a processing library.
- Returns a success/failure response so the worker can mark the job done or retry.

## Run

```bash
./gradlew bootRun
# or containerized:
docker compose -f ../infra/docker-compose.yml up image
```

See [docs/architecture.md](../docs/architecture.md) for the full design.
