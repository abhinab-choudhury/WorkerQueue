# video-service
> `VIDEO` job handler — REST API | External processing (FFmpeg)

One of the **handler** services in WorkQueue. Executes `VIDEO` jobs dispatched by the [worker-service](../worker-service) over HTTP.

## Role in the architecture

```
worker-service ──HTTP (POST /api/video)──▶ video-service
```

- Exposes a REST endpoint that receives `VIDEO` job payloads.
- Performs video processing (transcoding/encoding) by invoking FFmpeg as an external binary.
- Returns a success/failure response so the worker can mark the job done or retry.

## Run

```bash
./gradlew bootRun          # requires ffmpeg on PATH
# or containerized:
docker compose -f ../infra/docker-compose.yml up video
```

See [docs/architecture.md](../docs/architecture.md) for the full design.
