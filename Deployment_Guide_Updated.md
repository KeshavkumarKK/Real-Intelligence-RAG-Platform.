# Real Intelligence RAG Platform --- Deployment Guide

This guide documents the platform architecture, deployment workflow,
verification, and operational controls for the Real Intelligence RAG
Platform.

> **Deployment profile:** AWS is the current hosting environment. The
> application has previously been reported live and tested end to end.
> Cloudflare and HTTPS are reported working. The exact reverse-proxy
> implementation has not been confirmed. Application deployment is
> reported to use GitHub Actions CI/CD; verify the workflow files before
> documenting individual triggers or rollout mechanics. Redis retrieval
> caching is currently disabled by default. Current end-to-end
> validation may require the client's replacement OpenAI API key.

## Contents

-   [Overview](#overview)
-   [Architecture](#architecture)
-   [Deployment inventory](#deployment-inventory)
-   [Prerequisites](#prerequisites)
-   [Configuration and secrets](#configuration-and-secrets)
-   [Deploying with Docker Compose](#deploying-with-docker-compose)
-   [Domain, Cloudflare, and HTTPS](#domain-cloudflare-and-https)
-   [GitHub Actions and releases](#github-actions-and-releases)
-   [Knowledge-base synchronization](#knowledge-base-synchronization)
-   [Verification and acceptance
    tests](#verification-and-acceptance-tests)
-   [Logging and troubleshooting](#logging-and-troubleshooting)
-   [Security checklist](#security-checklist)
-   [Backup, recovery, and rollback](#backup-recovery-and-rollback)
-   [Multi-cloud considerations](#multi-cloud-considerations)
-   [Deployment record](#deployment-record)

# Overview

The Real Intelligence RAG Platform is a FastAPI service that retrieves
knowledge-base evidence, evaluates evidence relevance, generates
grounded answers, validates claims and citations, applies safety checks,
and returns an answer or a safe refusal.

The platform uses Docker Compose to manage services. OpenAI provides
embeddings and model-backed processing, Qdrant stores and retrieves
vectors, and Redis is available for optional caching of raw retrieval
results.

The application repository and the knowledge-base content are maintained
separately. The repository's `docker-compose.yml`, `.env.example`,
Dockerfile, application code, tests, and GitHub Actions workflows are
the source of truth for exact settings and supported commands.

A successful health check only confirms that the health endpoint
responds; it does not prove that OpenAI authentication, retrieval,
relevance judging, answer generation, or citation validation succeeds
end to end.

# Architecture

Client / Frontend
       |
       | HTTPS
       v
Domain / Cloudflare
       |
       v
Server gateway or reverse proxy
(implementation to be confirmed)
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
       +---- Redis (optional; cache disabled by default)
               +-- Raw retrieval-result cache

This is a verified as-built network diagram.
Confirm the actual request path, gateway/reverse-proxy product, TLS
termination, firewall rules, and internal service boundaries against the
deployed server before publishing implementation-specific details.

The retrieval cache stores raw retrieval results, not final answers.
Evidence filtering, safety checks, answer generation, claim validation,
and citation validation should still run for each request. Because a
cache hit can bypass a fresh Qdrant search, evidence freshness and
invalidation must be addressed before enabling caching in production.

# Deployment inventory

Only publish details that are safe to disclose.

  -----------------------------------------------------------------------
  Item                                Current understanding
  ----------------------------------- -----------------------------------
  Cloud provider                      AWS

  Server operating system             Ubuntu Linux

  Container management                Docker Compose

  API framework                       FastAPI / Uvicorn

  Vector database                     Qdrant

  Cache                               Redis, optional; currently disabled
                                      by default

  AI provider                         OpenAI API

  DNS / edge service                  Cloudflare

  HTTPS                               Reported configured and working

  Application deployment              Reported automated through GitHub
                                      Actions CI/CD; verify the exact
                                      workflow

  Knowledge-base synchronization      GitHub Actions workflow is present;
                                      inspect
                                      `.github/workflows/sync.yml`

  Application status                  Previously reported live and tested
                                      end to end; repeat end-to-end
                                      validation when valid API
                                      credentials are available

  Reverse proxy                       Exact product/configuration not
                                      confirmed

  Public domain                       Add only if approved for public
                                      disclosure
  -----------------------------------------------------------------------

Notot publish account IDs, private IP addresses, private hostnames, SSH
details, credentials, or other sensitive infrastructure information.

# Prerequisites

-   An AWS-hosted Ubuntu server with suitable CPU, memory, and disk
    capacity.
-   Docker Engine and the Docker Compose plugin.
-   Access to the application repository and the approved release
    revision.
-   Valid OpenAI credentials and the required platform/API token.
-   Correct Qdrant endpoint, authentication (if enabled), and collection
    configuration.
-   Redis configured if retrieval caching is intentionally enabled.
-   DNS, Cloudflare, HTTPS, firewall, and gateway configuration
    appropriate to the deployment.
-   GitHub Actions secrets and permissions configured for the workflows
    actually used.
-   A documented backup, restoration, and rollback process.

Use the dependency files and repository workflows to determine supported
versions and required variables.

# Configuration and secrets

Use `.env.example`, `docker-compose.yml`, and application code to
determine the required environment. The current Compose setup requires
`OPENAI_API_KEY` and `RI_API_TOKEN`.

Relevant settings include:

  Variable                        Purpose
  ------------------------------- -------------------------------------
  `OPENAI_API_KEY`                OpenAI API authentication
  `RI_API_TOKEN`                  Platform/API token
  `OPENAI_EMBEDDING_MODEL`        Optional embedding-model override
  `QDRANT_URL`                    Qdrant endpoint
  `QDRANT_API_KEY`                Optional Qdrant authentication
  `QDRANT_COLLECTION`             Vector collection name
  `REDIS_URL`                     Redis connection URL
  `RAG_RETRIEVAL_CACHE_ENABLED`   Enable or disable retrieval caching
  `RAG_CACHE_VERSION`             Cache namespace/version
  `RAG_CACHE_TTL_SECONDS`         Cache entry lifetime
  `RAG_CACHE_SCOPE`               Cache scope
  `CORS_ALLOW_ORIGINS`            Allowed browser origins

This table exhaustive. Check the current configuration files
before deploying.

Security requirements:

-   Keep `.env` out of source control.
-   Never put keys, tokens, passwords, or private SSH keys in
    documentation, logs, or public issues.
-   Restrict access to server-side secret files and CI/CD secrets.
-   Prefer a managed secret store where appropriate.
-   Rotate credentials promptly if exposed or revoked.
-   Do not bake secrets into Docker images.
-   Verify which environment the running container loaded without
    printing secret values.
-   When a key expires, update the approved secret source and
    recreate/restart the relevant service as required; do not commit the
    replacement key.

## Deploying with Docker Compose

Run commands from the application directory on the target server after
configuring its environment. Review the change set and Compose
configuration before applying a release.

``` bash
# Validate Compose syntax and variable interpolation
docker compose config --quiet

# Build and start configured services
docker compose up -d --build

# Inspect service status and health
docker compose ps

# Read recent API logs
docker compose logs --tail 200 api
```

Wait for configured health checks to pass. Investigate unhealthy or
repeatedly restarting containers before continuing.

For later releases, verify the approved source revision and tests before
updating the application. Avoid destructive volume commands unless the
data implications and recovery plan are understood. Use the actual
service names from the current Compose file.

## Domain, Cloudflare, and HTTPS

Cloudflare and HTTPS are reported to be part of the current deployment
and working. The exact server-side gateway or reverse proxy has not been
confirmed, so do not claim a specific product or configuration until
verified.

Verify and document:

-   DNS records and the intended public hostname.
-   Cloudflare proxy and TLS settings.
-   The reverse proxy or gateway actually used on the server, if any.
-   The upstream API address and port.
-   TLS certificate issuance and renewal responsibility.
-   HTTP-to-HTTPS redirection, if configured.
-   Firewall rules and exposed ports.
-   API authentication and rate limiting, where applicable.
-   Request-size and timeout limits, where appropriate.

Do not expose Qdrant or Redis directly to the public internet. Keep
internal services on a private interface or private container network
unless a separately reviewed architecture requires otherwise. CORS is
not authentication.

## GitHub Actions and releases

Application deployment is reported to use GitHub Actions CI/CD. Inspect
the actual workflow files before describing the exact trigger, build
mechanism, target host, approvals, or rollout strategy. The repository
also contains a knowledge-base synchronization workflow; application
deployment and knowledge-base synchronization should remain separate
unless the configured automation explicitly combines them.

Document the following from the actual workflow:

1.  Source branch, release trigger, and approved revision.
2.  Tests and checks that run before deployment.
3.  How the image or application artifact is built and delivered.
4.  How the deployment host receives the approved release.
5.  Where environment-specific secrets are stored.
6.  Workflow permissions and any required approvals.
7.  Post-deployment health and end-to-end checks.
8.  Rollback procedure if a release fails.

Do not claim zero-downtime deployment, automatic rollback, or a specific
deployment strategy unless it is implemented and verified.

## Knowledge-base synchronization

Knowledge-base content is maintained separately from the application
repository. Before synchronizing content:

1.  Confirm the source repository and revision.
2.  Confirm the target Qdrant endpoint and collection.
3.  Review the workflow and required repository credentials.
4.  Review dry-run or planning output where supported.
5.  Confirm backup and recovery arrangements.
6.  Run synchronization only with appropriate authorization.
7.  Review synchronization artifacts and verify the resulting content.

The repository documentation references this CLI help command:

``` bash
py -m scripts.sync.sync_cli --help
```

Use the Python environment and command available in the deployed
runtime. Do not trigger a production synchronization solely to test
local setup.

If retrieval caching is introduced, ensure a successful knowledge-base
update invalidates or versions out cached retrieval results. Do not
enable production evidence caching until freshness and invalidation
behavior have been verified.

## Verification and acceptance tests

A successful health endpoint does not prove that the complete RAG
workflow works. Verify each dependency and then run an authorized
end-to-end request.

### 1. Check container status

``` bash
docker compose ps
```

### 2. Check API health

For an API bound locally to port 8000:

``` bash
curl -fsS http://127.0.0.1:8000/health
```

If the API is exposed through a domain, verify the configured HTTPS
health endpoint as well.

### 3. Check Redis when needed

``` bash
docker compose exec redis redis-cli ping
```

Expected output:

``` text
PONG
```

The Redis container may be healthy while application-level caching is
disabled. Verify the effective cache setting without printing secrets.
The current safe default is `RAG_RETRIEVAL_CACHE_ENABLED=0`.

### 4. Test an authenticated end-to-end request

Use the actual API address and a question supported by the deployed
knowledge base. This local example may need adjustment for the
production gateway and API authentication configuration.

``` bash
curl -X POST "http://127.0.0.1:8000/v1/answer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${RI_API_TOKEN}" \
  -d '{
    "question": "What information is available about this topic?",
    "platform": "ifi",
    "history": []
  }'
```

Use a valid, authorized API credential and a knowledge-base-supported
question. Never paste credentials into shared logs or documentation.

Confirm that the response is a grounded answer with valid citations or
an appropriate safe refusal when evidence is insufficient or a safety
restriction applies. Check the server logs for dependency or
authentication errors if the request fails.

### Acceptance checklist

-   [ ] Compose configuration validates.
-   [ ] Required containers are healthy.
-   [ ] API health endpoint responds.
-   [ ] OpenAI authentication and embedding generation succeed.
-   [ ] Qdrant retrieval works against the intended collection.
-   [ ] End-to-end requests produce grounded answers or expected safe
    refusals.
-   [ ] Citation validation behaves as expected.
-   [ ] Redis connectivity is checked when relevant.
-   [ ] Cache miss/hit, TTL, failure fallback, and invalidation are
    tested before enabling caching.
-   [ ] Domain and HTTPS are verified.
-   [ ] GitHub Actions workflows match the intended release process.
-   [ ] Backup and rollback procedures are documented and tested.

Record the date, source revision, test outcomes, and known limitations
after running checks. If a required API key is expired, mark live
end-to-end validation as blocked pending credential renewal rather than
reporting it as currently passing.

## Logging and troubleshooting

Read recent API logs:

``` bash
docker compose logs --tail 200 api
```

Inspect Qdrant and Redis logs when needed:

``` bash
docker compose logs --tail 100 qdrant
docker compose logs --tail 100 redis
```

Use the actual service names in the Compose configuration. Never publish
secrets, authorization headers, or unnecessary sensitive content when
sharing logs.

  -----------------------------------------------------------------------
  Symptom                             Investigation
  ----------------------------------- -----------------------------------
  API returns a generic 503           Inspect authorized API logs for the
                                      underlying exception

  OpenAI returns                      Verify that the key is active and
  `401 invalid_api_key`               the API container has loaded the
                                      intended value

  Retrieval returns no useful         Check Qdrant connectivity,
  evidence                            collection settings, filters,
                                      relevance decisions, and
                                      knowledge-base coverage

  Redis connection fails              Check service health, `REDIS_URL`,
                                      and the Compose network

  Cache stays empty                   Confirm caching is enabled and the
                                      request reaches an eligible
                                      retrieval operation

  HTTPS fails                         Check DNS, Cloudflare TLS settings,
                                      gateway/proxy configuration,
                                      certificates, and firewall rules

  Deployment fails                    Review workflow logs, image build
                                      output, environment configuration,
                                      and service health
  -----------------------------------------------------------------------

## Security checklist

-   [ ] Secrets are excluded from Git.
-   [ ] Public API access is appropriately authenticated.
-   [ ] HTTPS is enabled and verified where required.
-   [ ] Qdrant and Redis are not publicly exposed.
-   [ ] Firewall rules restrict unnecessary access.
-   [ ] CORS allows only required origins.
-   [ ] Cache-key composition and tenant isolation have been reviewed.
-   [ ] Cache invalidation and evidence freshness are verified before
    caching is enabled.
-   [ ] Logs avoid credentials and unnecessary sensitive content.
-   [ ] CI/CD credentials follow least privilege.
-   [ ] Backup access is restricted.
-   [ ] Recovery and rollback procedures have been tested.
-   [ ] Dependencies are updated through the normal review process.

## Backup, recovery, and rollback

Document the actual backup and recovery procedures for:

-   Qdrant collections and knowledge-base data.
-   Persistent application state and synchronization metadata.
-   Relevant configuration, excluding secrets from ordinary backups
    where possible.
-   Release versions and deployment artifacts.

Define backup frequency, retention, protected storage location,
recovery-point objective, and recovery-time objective according to
service requirements.

Redis cache contents may be disposable if they can safely be rebuilt;
verify the persistence configuration before assuming this. Do not delete
volumes during routine troubleshooting without understanding their
contents.

A backup is not proven usable until a restoration test succeeds. Record
the most recent successful restore test and outcome.

For rollback, document how to return to a previously approved
application image or source revision, how to verify service health
afterward, and whether data or schema changes require a separate
recovery step.

## Multi-cloud considerations

AWS is the current hosting environment. The containerized architecture
may simplify deployment to other cloud providers, but portability is not
the same as a verified deployment.

Before claiming another provider as validated, test its deployment path,
networking, secrets management, persistent storage, observability,
backup/recovery, and end-to-end RAG behavior. Record the provider, date,
tested configuration, and result.

Do not imply that the production workload runs simultaneously across
multiple clouds unless that is the actual architecture.

## Deployment record

Update this section after a release or validation run. Fill in only
verified, non-sensitive values.

  -----------------------------------------------------------------------
  Field                               Value
  ----------------------------------- -----------------------------------
  Provider                            AWS

  Server OS                           Ubuntu Linux

  Application revision                Record commit SHA or release tag

  Deployment date                     Record actual date

  Deployment workflow                 Record actual workflow name

  Health checks                       Pass / Fail, with date

  End-to-end test                     Pass / Fail / Blocked pending valid
                                      API credentials, with date

  Cache verification                  Disabled / Pass / Fail / Not tested

  HTTPS verification                  Reported working; record last
                                      verified date

  Backup/restore verification         Record result and date

  Known issues                        Record confirmed issues only
  -----------------------------------------------------------------------

Update this document when the deployment architecture, security
controls, service configuration, or release process changes.
