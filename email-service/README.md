# email-service
> `EMAIL` job handler — REST endpoint | Optional mail (SMTP) support

One of the **handler** services in WorkQueue. Executes `EMAIL` jobs dispatched by the [worker-service](../worker-service) over HTTP.

## Role in the architecture

```
worker-service ──HTTP (POST /api/email)──▶ email-service
```

- Exposes a REST endpoint that receives `EMAIL` job payloads.
- Composes and sends the email (SMTP support optional).
- Returns a success/failure response so the worker can mark the job done or retry.

## Run

```bash
./gradlew bootRun
# or containerized:
docker compose -f ../infra/docker-compose.yml up email
```

See [docs/architecture.md](../docs/architecture.md) for the full design.
