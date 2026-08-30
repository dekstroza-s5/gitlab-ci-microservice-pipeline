# GitLab CI Microservice Pipeline

Small Python HTTP service with a delivery pipeline:

```text
lint -> test -> build -> publish -> deploy -> verify
```

## Local run

```bash
docker compose up --build
curl http://localhost:8080/health
```

The production deployment job requires manual approval. Kubernetes credentials must be stored as protected CI variables.
