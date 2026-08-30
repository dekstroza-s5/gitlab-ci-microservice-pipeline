# GitLab CI Microservice Pipeline

A runnable Python HTTP service and an end-to-end GitLab delivery pipeline. The repository demonstrates the path from a source change to a verified Kubernetes rollout.

## Delivery flow

```text
commit
  -> lint and unit test
  -> container build
  -> registry publication with immutable commit tag
  -> automatic stage deployment
  -> rollout and HTTP verification
  -> manually approved tagged production deployment
```

## Application endpoints

- `GET /`: service identity and state
- `GET /health`: readiness endpoint
- `GET /metrics`: Prometheus metrics

## Local development

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
ruff check .
pytest -q
python app.py
```

Expected test output:

```text
1 passed
```

Run the production container command locally:

```bash
docker compose up --build -d
curl --fail http://localhost:8080/health
curl --fail http://localhost:8080/metrics | head
docker compose logs api
```

## Container design

The Dockerfile uses a wheel-building stage and a smaller runtime stage. The runtime process runs as UID 10001, dependencies are copied from built wheels and Gunicorn handles the HTTP process.

Useful checks:

```bash
docker build -t demo-api:local .
docker image inspect demo-api:local --format '{{.Size}}'
docker run --rm demo-api:local id
docker history demo-api:local
```

## Required GitLab variables

Configure these as protected and masked variables where applicable:

| Variable | Purpose |
|---|---|
| `KUBE_CONFIG` or agent context | cluster authentication |
| `STAGE_HEALTH_URL` | external health endpoint |
| `CI_REGISTRY_USER` | provided by GitLab |
| `CI_REGISTRY_PASSWORD` | provided by GitLab |
| `CI_REGISTRY_IMAGE` | provided by GitLab |

No kubeconfig or registry password belongs in the repository.

## Pipeline behavior

The image tag is `$CI_COMMIT_SHORT_SHA`, giving every deployment a traceable artifact. Stage deploys from `main`. Production deploys only from a Git tag and requires manual approval.

The deployment job:

1. substitutes the immutable image into the manifest;
2. applies resources to the selected namespace;
3. waits for `kubectl rollout status`;
4. fails the pipeline when readiness is not reached.

## Release example

```bash
git tag -a v1.2.0 -m "release v1.2.0"
git push origin v1.2.0
```

Review the pipeline, approve `deploy_production`, then verify:

```bash
kubectl -n production get deploy,pods,svc
kubectl -n production rollout history deployment/demo-api
curl --fail https://api.example.com/health
```

## Failure investigation

- lint/test failure: reproduce using the exact commands from the job.
- build failure: inspect Docker context, dependency resolution and registry availability.
- rollout timeout: use `kubectl describe`, pod logs and namespace events.
- verification failure after successful rollout: inspect Service endpoints, Ingress and external DNS.
- wrong version running: compare deployment image, registry digest and commit SHA.

Rollback:

```bash
kubectl -n production rollout undo deployment/demo-api
kubectl -n production rollout status deployment/demo-api
```

## Security notes

The pipeline does not print secrets, production deployment is gated, containers run without root and release artifacts use immutable tags. In a real platform, add image scanning, dependency scanning, signed images and policy enforcement.
