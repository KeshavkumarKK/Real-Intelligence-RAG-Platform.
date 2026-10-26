# Real Intelligence RAG Platform — Deployment Guide

This document describes the deployment architecture, release workflow, verification steps, and operational practices for the Real Intelligence RAG Platform.

> **Deployment profile:** AWS is the current hosting environment. The platform is reported to be live and tested end to end. Other cloud providers should be described as validated deployment targets only to the extent that they have been independently tested. Replace environment-specific placeholders below with verified values before publishing.

## Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Deployment inventory](#deployment-inventory)
- [Prerequisites](#prerequisites)
- [Configuration and secrets](#configuration-and-secrets)
- [Deploying with Docker Compose](#deploying-with-docker-compose)
- [Domain, Cloudflare, and HTTPS](#domain-cloudflare-and-https)
- [GitHub Actions and releases](#github-actions-and-releases)
- [Knowledge-base synchronization](#knowledge-base-synchronization)
- [Verification and acceptance tests](#verification-and-acceptance-tests)
- [Logging and troubleshooting](#logging-and-troubleshooting)
- [Security checklist](#security-checklist)
- [Backup, recovery, and rollback](#backup-recovery-and-rollback)
- [Multi-cloud considerations](#multi-cloud-considerations)
- [Deployment record](#deployment-record)

## Overview

The Real Intelligence RAG Platform is a FastAPI service that retrieves knowledge-base evidence, evaluates evidence relevance, generates grounded answers, validates claims and citations, and returns an answer or a safe refusal.

The platform uses Docker Compose to manage application services. OpenAI provides embeddings and model-backed processing, Qdrant stores and retrieves vectors, and Redis can optionally cache eligible raw retrieval results.

The repository's `docker-compose.yml`, `.env.example`, Dockerfile, application code, and GitHub Actions workflows remain the source of truth for exact settings and supported commands.

## Architecture

Client / Frontend
       |
       | HTTPS
       v
Domain and Cloudflare
       |
       v
Reverse proxy / gateway
       |
       v
FastAPI application (Uvicorn)
       |
       +---- OpenAI API
       |       +-- Query embeddings
       |       +-- Model-backed processing
       |
       +---- Qdrant
       |       +-- Vector search
       |       +-- Knowledge-base retrieval
       |
       +---- Redis (optional)
               +-- Raw retrieval-result cache

This is a logical architecture. Confirm the exact request path, reverse-proxy product, network boundaries, and TLS termination points against the live server before publishing it as an as-built diagram.

The retrieval cache stores raw retrieval results, not final answers. Evidence filtering, safety checks, answer generation, claim validation, and citation validation should continue to run for each request.

## Deployment inventory

Fill in only values that are safe to disclose publicly.

| Item | Current deployment information |
|---|---|
| Cloud provider | AWS |
| Server operating system | Ubuntu Linux |
| Container management | Docker Compose |
| API framework | FastAPI |
| Vector database | Qdrant |
| Cache | Redis, when enabled |
| AI provider | OpenAI API |
| DNS / edge service | Cloudflare |
| HTTPS | Reported configured and working |
| Release automation | GitHub Actions CI/CD (confirm application deployment workflow) |
| Application status | Reported live and tested end to end |
| Public domain | Add only if intended to be public |
| Reverse proxy | Confirm the actual software and configuration |

Do not publish account IDs, private IP addresses, private hostnames, SSH details, or credentials.

## Prerequisites

- An AWS-hosted Ubuntu server sized for the application and its data.
- Docker Engine and the Docker Compose plugin.
- Access to the application repository.
- Valid OpenAI credentials and the required platform/API token.
- Correct Qdrant connection and collection configuration.
- Redis configured if retrieval caching is enabled.
- DNS, Cloudflare, HTTPS, and firewall configuration appropriate to the deployment.
- GitHub Actions secrets and permissions configured for the workflows actually used.
- A documented backup and recovery process for persistent data.

Use the dependency files and repository workflows to determine supported versions and required variables.

## Configuration and secrets

Use `.env.example` and the Compose configuration to determine required environment variables. The current Compose setup requires `OPENAI_API_KEY` and `RI_API_TOKEN`.

Relevant settings may include:

| Variable | Purpose |
|---|---|
| `OPENAI_API_KEY` | OpenAI API authentication |
| `RI_API_TOKEN` | Platform/API token |
| `OPENAI_EMBEDDING_MODEL` | Optional embedding model override |
| `QDRANT_URL` | Qdrant endpoint |
| `QDRANT_API_KEY` | Optional Qdrant authentication |
| `QDRANT_COLLECTION` | Vector collection name |
| `REDIS_URL` | Redis connection URL |
| `RAG_RETRIEVAL_CACHE_ENABLED` | Enable or disable retrieval caching |
| `RAG_CACHE_VERSION` | Cache namespace/version |
| `RAG_CACHE_TTL_SECONDS` | Cache entry lifetime |
| `RAG_CACHE_SCOPE` | Cache scope |
| `CORS_ALLOW_ORIGINS` | Allowed browser origins |

This list may not be exhaustive. Check `.env.example`, `docker-compose.yml`, and application code for the current configuration.

Security requirements:

- Keep `.env` out of source control.
- Never paste keys, tokens, passwords, or private SSH keys into public issues or documentation.
- Restrict access to secret files and CI/CD secrets.
- Prefer a managed secret store where appropriate.
- Rotate credentials promptly if exposed.
- Do not bake secrets into Docker images or print them in logs.
- Verify which environment the running container has loaded without printing secret values.

## Deploying with Docker Compose

Run these commands from the application directory on the target server after configuring its environment. Review the Compose file before applying changes.

```bash
# Validate Compose syntax and variable interpolation
docker compose config --quiet

# Build and start the configured services
docker compose up -d --build

# Inspect service status and health
docker compose ps

# Read recent API logs
docker compose logs --tail 200 api

Wait for configured health checks to pass. Investigate containers that are unhealthy or repeatedly restarting before continuing.

For later releases, review the source revision and change set before updating the application. Avoid destructive volume commands unless the data implications and recovery plan are understood.

## Domain, Cloudflare, and HTTPS

Cloudflare is reported to be part of the current deployment and HTTPS is reported working. Confirm the actual configuration before documenting specific implementation details.

Verify and document:

- DNS records and the intended public hostname.
- Cloudflare proxy and TLS settings.
- The reverse proxy or gateway used on the server, if any.
- The upstream API address and port.
- TLS certificate issuance and renewal responsibility.
- HTTP-to-HTTPS redirection, if configured.
- Firewall rules and exposed ports.
- API authentication and rate limiting, where applicable.
- Request-size and timeout limits where appropriate.

Do not expose Qdrant or Redis directly to the public internet. CORS is not authentication. Keep internal services on a private interface or private container network unless an explicitly reviewed architecture requires otherwise.

## GitHub Actions and releases

The repository uses GitHub Actions for CI and has a knowledge-base synchronization workflow. Confirm whether application deployment itself is automated by a separate workflow before describing it as fully automated CI/CD.

For the application release process, document:

1. The source branch and release trigger.
2. Tests and checks that run before deployment.
3. How the image is built and delivered to the server.
4. How the deployment host receives the approved release.
5. Where environment-specific secrets are stored.
6. Required approvals and permissions.
7. Post-deployment health and end-to-end checks.
8. The rollback procedure if a release fails.

Keep application deployment and knowledge-base synchronization as separate workflows unless the configured automation explicitly combines them.

## Knowledge-base synchronization

Knowledge-base content is maintained separately from the application repository. Before synchronizing content:

1. Confirm the source repository and revision.
2. Confirm the target Qdrant endpoint and collection.
3. Review the workflow and required repository credentials.
4. Review dry-run or planning output where supported.
5. Confirm backup and recovery arrangements.
6. Run synchronization only with appropriate authorization.
7. Review synchronization artifacts and verify the resulting content.

Inspect synchronization CLI options using the command supported by the current checkout and runtime. The repository documentation references:

py -m scripts.sync.sync_cli --help

On a Linux server, use the Python environment and command available in that deployment. Do not trigger a production synchronization solely to test a local setup.

## Verification and acceptance tests

A successful health endpoint does not prove that the complete RAG workflow works. Verify each dependency and then run an authorized end-to-end request.

### 1. Check container status


docker compose ps

### 2. Check API health

For an API bound locally to port 8000:

curl -fsS http://127.0.0.1:8000/health

If the API is exposed through a domain, verify the configured HTTPS health endpoint as well.

### 3. Check Redis when enabled

docker compose exec redis redis-cli ping


Expected output:

PONG

### 4. Test an end-to-end request

Use the actual API address and a question supported by the deployed knowledge base. This example is for a local API and may need adjustment for the production gateway and authentication configuration.

curl -X POST "http://127.0.0.1:8000/v1/answer" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "What information is available about this topic?",
    "platform": "ifi",
    "history": []
  }'

Confirm that the response is a grounded answer with valid citations or an appropriate safe refusal when evidence is insufficient or a safety restriction applies.

### Acceptance checklist

- [ ] Compose configuration validates.
- [ ] Required containers are healthy.
- [ ] API health endpoint responds.
- [ ] OpenAI authentication and embedding generation succeed.
- [ ] Qdrant retrieval works against the intended collection.
- [ ] End-to-end requests produce grounded answers or expected safe refusals.
- [ ] Citation validation behaves as expected.
- [ ] Redis connectivity and cache hit/miss behavior are tested when caching is enabled.
- [ ] Domain and HTTPS are verified.
- [ ] GitHub Actions workflows match the intended release process.
- [ ] Backup and rollback procedures are documented and tested.

Record the date, source revision, test outcomes, and known limitations after running the checks.

## Logging and troubleshooting

Read recent API logs:

docker compose logs --tail 200 api

Inspect Qdrant and Redis logs when needed:

docker compose logs --tail 100 qdrant
docker compose logs --tail 100 redis


Use the actual service names in the Compose configuration.

| Symptom | Investigation |
|---|---|
| API returns a generic 503 | Inspect authorized API logs for the underlying exception |
| OpenAI returns `401 invalid_api_key` | Verify the key is active and the API container has loaded the intended value |
| Retrieval returns no useful evidence | Check Qdrant connectivity, collection settings, filters, and knowledge-base coverage |
| Redis connection fails | Check service health, `REDIS_URL`, and the Compose network |
| Cache stays empty | Confirm caching is enabled and the request reaches an eligible retrieval operation |
| HTTPS fails | Check DNS, Cloudflare TLS settings, proxy configuration, certificates, and firewall rules |
| Deployment fails | Review workflow logs, image build output, environment configuration, and service health |

Never publish secrets, authorization headers, or unnecessary sensitive content when sharing logs.

## Security checklist

- [ ] Secrets are excluded from Git.
- [ ] Public API access is appropriately authenticated.
- [ ] HTTPS is enabled and verified where required.
- [ ] Qdrant and Redis are not publicly exposed.
- [ ] Firewall rules restrict unnecessary access.
- [ ] CORS allows only required origins.
- [ ] Cache-key composition and tenant isolation have been reviewed.
- [ ] Logs avoid credentials and unnecessary sensitive content.
- [ ] CI/CD credentials follow least privilege.
- [ ] Backup access is restricted.
- [ ] Recovery and rollback procedures have been tested.
- [ ] Dependencies are updated through the normal review process.

## Backup, recovery, and rollback

Document the actual backup and recovery procedures for:

- Qdrant collections and knowledge-base data.
- Persistent application state and synchronization metadata.
- Relevant configuration, excluding secrets from ordinary backups where possible.
- Release versions and deployment artifacts.

Define backup frequency, retention, protected storage location, recovery-point objective, and recovery-time objective according to service requirements.

Redis cache contents may be disposable if they can safely be rebuilt; verify the persistence configuration before assuming this. Do not delete volumes during routine troubleshooting without understanding their contents.

A backup is not proven usable until a restoration test succeeds. Record the most recent successful restore test and outcome.

For rollback, document how to return to a previously approved application image or source revision, how to verify service health afterward, and whether data or schema changes require a separate recovery step.

## Multi-cloud considerations

AWS is the current hosting environment. The containerized architecture may simplify deployment to other cloud providers, but portability is not the same as a verified deployment.

Before claiming a provider as validated, test its deployment path, networking, secrets management, persistent storage, observability, backup/recovery, and end-to-end RAG behavior. Record the provider, date, tested configuration, and result.

Do not imply that the production workload runs simultaneously across multiple clouds unless that is the actual architecture.

## Deployment record

Update this section after a release or validation run.

| Field | Value |
|---|---|
| Provider | AWS |
| Server OS | Ubuntu Linux |
| Application revision | [Record commit SHA or release tag] |
| Deployment date | [YYYY-MM-DD] |
| Deployment workflow | [Record actual workflow name] |
| Health checks | [Pass / Fail, with date] |
| End-to-end test | [Pass / Fail, with date] |
| Cache verification | [Pass / Fail / Disabled / Not tested] |
| Backup/restore verification | [Record result and date] |
| Known issues | [List known issues or `None confirmed`] |

 This document is updated when the deployment architecture, security controls, service configuration, or release process changes.
