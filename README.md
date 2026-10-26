# Real Intelligence RAG Platform

A retrieval-augmented generation (RAG) API for retrieving knowledge-base
content, generating evidence-grounded answers, validating claims, and
returning citations or a safe refusal.

> **Note:** This repository contains the platform implementation.
> Knowledge-base content is maintained separately. Review deployment
> configuration and security controls before using this project with
> production data.

# Contents

-   [Overview](#overview)
-   [Capabilities](#capabilities)
-   [Architecture](#architecture)
-   [Repository layout](#repository-layout)
-   [Quick start](#quick-start)
-   [Configuration](#configuration)
-   [HTTP API](#http-api)
-   [Retrieval cache](#retrieval-cache)
-   [Safety and grounding](#safety-and-grounding)
-   [Knowledge-base synchronization](#knowledge-base-synchronization)
-   [Testing](#testing)
-   [Troubleshooting](#troubleshooting)
-   [Security](#security)
-   [Limitations](#limitations)
-   [Contributing](#contributing)
-   [License](#license)

# Overview

The Real Intelligence RAG Platform provides an API for answering
questions from a curated knowledge base. It combines semantic retrieval
with answer generation and validation stages intended to reduce
unsupported responses.

At a high level, the service validates a question, applies input-safety
checks, retrieves candidate knowledge chunks from Qdrant, checks the
evidence, generates a candidate answer using OpenAI-backed components,
validates claims, applies output-safety checks, and returns a validated
response with citations---or a safe refusal when evidence is
insufficient or a restriction applies.

A refusal is an expected outcome when the system cannot support a
reliable answer from the available evidence. Safeguards reduce risk but
do not guarantee correctness.

# Capabilities

-   FastAPI HTTP API for health checks and question answering.
-   Semantic retrieval using OpenAI embeddings and Qdrant.
-   Platform-aware retrieval for `ifi`, `btcx`, and `shared`.
-   Support for pinned or always-included knowledge chunks.
-   Evidence relevance and status checks.
-   Claim validation and semantic assessment of claim support.
-   Citation construction and validation against source metadata.
-   Financial-input and financial-output safety checks.
-   Optional Redis-backed retrieval-result cache with configurable
    enablement, TTL, version, and scope.
-   Automated tests and GitHub Actions CI.
-   Automatically triggered knowledge-base synchronization workflow.
-   Operational logging and sync artifacts for troubleshooting.

# Architecture

Client
  |
  v
FastAPI API
  |
  v
GroundedAnswerPipeline
  |
  +--> Request validation and input safety
  |
  +--> Optional RetrievalEvidenceCache (Redis)
  |       |
  |       +--> Cache hit: reuse raw retrieval results
  |       +--> Cache miss: query the retriever
  |
  +--> QdrantRetriever
  |       +--> OpenAI query embeddings
  |       +--> Qdrant vector search
  |       +--> Platform filters and pinned chunks
  |
  +--> Evidence relevance/status checks
  +--> Answer generation
  +--> Claim validation and semantic judgment
  +--> Answer-grounding and output-safety checks
  +--> Citation construction and validation
  |
  v
Validated response or safe refusal


The cache stores raw retrieval results, not final answers. Evidence
filtering, safety checks, answer generation, claim validation, and
citation validation still run for each request. Cache read/write errors
are designed to fail open so the pipeline can attempt normal retrieval.

# Technology stack

  Component        Purpose
  ---------------- ----------------------------------------
  Python           Application runtime
  FastAPI          HTTP API
  Uvicorn          ASGI server
  OpenAI API       Embeddings and model-backed components
  Qdrant           Vector storage and semantic retrieval
  Redis            Optional retrieval-result cache
  Docker Compose   Local multi-service development
  pytest           Automated tests
  GitHub Actions   CI and Automated synchronization workflow

Use the dependency files and CI configuration as the source of truth for
supported versions.

# Repository layout

| Variable | Purpose | Notes |
|---|---|---|
| `OPENAI_API_KEY` | OpenAI API authentication | Required by current Compose configuration |
| `RI_API_TOKEN` | Platform/API token | Required by current Compose configuration; keep private |
| `OPENAI_EMBEDDING_MODEL` | Embedding model override | Retriever default: `text-embedding-3-small` |
| `QDRANT_URL` | Qdrant endpoint | Use the Compose service name from containers |
| `QDRANT_API_KEY` | Optional Qdrant authentication | Needed only if Qdrant requires it |
| `QDRANT_COLLECTION` | Qdrant collection | Retriever default: `ri_knowledge` |
| `REDIS_URL` | Redis connection URL | Compose default: `redis://redis:6379/0` |
| `RAG_RETRIEVAL_CACHE_ENABLED` | Enable retrieval cache | Compose default is `0` unless overridden |
| `RAG_CACHE_VERSION` | Cache namespace/version | Default: `v1` |
| `RAG_CACHE_TTL_SECONDS` | Cache entry lifetime in seconds | Default: `300` |
| `RAG_CACHE_SCOPE` | Cache scope | Default: `shared`; review tenant-isolation needs |
| `CORS_ALLOW_ORIGINS` | Allowed browser origins | Allow only required origins |


The current checkout may differ; use the repository tree as the source
of truth.

# Quick start

These commands assume PowerShell from the repository root and a working
Docker Compose installation.

# 1. Prepare configuration

Copy-Item .env.example .env
code .env

Populate the required values locally. The current Compose configuration
requires `OPENAI_API_KEY` and `RI_API_TOKEN`. Review `.env.example` and
`docker-compose.yml` for the complete list. Never commit `.env` or share
its contents.

# 2. Validate and start services


docker compose config --quiet
docker compose up -d --build
docker compose ps


Wait for configured health checks to pass.

# 3. Check the API

Invoke-RestMethod http://127.0.0.1:8000/health

When enabled by the application, interactive documentation is available
at `http://127.0.0.1:8000/docs` and the OpenAPI schema at
`http://127.0.0.1:8000/openapi.json`.

A successful health response does not prove that OpenAI authentication,
Qdrant retrieval, or answer generation works end to end.

# 4. Stop services

docker compose down

Review the Compose file before removing volumes. Do not remove volumes
unless you understand which data they contain.

# Configuration

The authoritative configuration inventory is defined in `.env.example`,
`docker-compose.yml`, and application code. Common settings include:

  ----------------------------------------------------------------------------------
  Variable                        Purpose                 Notes
  ------------------------------- ----------------------- --------------------------
  `OPENAI_API_KEY`                OpenAI API              Required by current
                                  authentication          Compose configuration

  `RI_API_TOKEN`                  Platform/API token      Keep private
                                  required by current
                                  Compose configuration

  `OPENAI_EMBEDDING_MODEL`        Embedding model         Retriever default:
                                  override                `text-embedding-3-small`

  `QDRANT_URL`                    Qdrant endpoint         Use the Compose service
                                                          name from containers

  `QDRANT_API_KEY`                Optional Qdrant         Needed only if Qdrant
                                  authentication          requires it

  `QDRANT_COLLECTION`             Qdrant collection       Retriever default:
                                                          `ri_knowledge`

  `REDIS_URL`                     Redis connection URL    Compose default:
                                                          `redis://redis:6379/0`

  `RAG_RETRIEVAL_CACHE_ENABLED`   Enable retrieval cache  Compose default is `0`
                                                          unless overridden

  `RAG_CACHE_VERSION`             Cache namespace/version Default: `v1`

  `RAG_CACHE_TTL_SECONDS`         Cache entry lifetime in Default: `300`
                                  seconds

  `RAG_CACHE_SCOPE`               Cache scope             Default: `shared`; review
                                  configuration           tenant-isolation needs

  `CORS_ALLOW_ORIGINS`            Comma-separated browser Allow only required
                                  origins                 origins
  ----------------------------------------------------------------------------------

Do not assume this table replaces `.env.example` as the complete
configuration reference. Never commit API keys, bearer tokens,
passwords, or production secrets. Rotate credentials if exposed and use
a managed secret store for production deployments.

# HTTP API

The FastAPI application is defined in `scripts/api.py`.

### Health check

GET /health

Example response:

{
  "status": "ok"
}

This is a basic health endpoint, not necessarily an end-to-end
dependency check.

# Ask a question

POST /v1/answer
Content-Type: application/json

Example request:

{
  "question": "What information is available about this topic?",
  "platform": "ifi",
  "history": []
}


Supported platform values are `ifi`, `btcx`, and `shared`. Check the
request model in source for exact field validation limits.

PowerShell example:

$body = @{
    question = "What information is available about this topic?"
    platform = "ifi"
    history = @()
} | ConvertTo-Json -Depth 8

Invoke-RestMethod `
    -Method Post `
    -Uri "http://127.0.0.1:8000/v1/answer" `
    -ContentType "application/json" `
    -Body $body

The response model includes answer text, claims, evidence chunk IDs,
refusal status, refusal reason, and citations. Citation metadata can
include document ID, source path, section, and chunk ID.

# Error behavior

Depending on the failure, the API may return request-validation details,
a normal response indicating a refusal, or a generic service-unavailable
response for an unexpected internal error. Internal exception details
should be diagnosed from authorized local logs, not exposed to API
clients or public issue reports.

# Retrieval cache

The optional Redis cache reduces repeated retrieval work for eligible
repeated queries.

Example settings:

RAG_RETRIEVAL_CACHE_ENABLED=1
RAG_CACHE_VERSION=v1
RAG_CACHE_TTL_SECONDS=300
RAG_CACHE_SCOPE=shared
REDIS_URL=redis://redis:6379/0

Set `RAG_RETRIEVAL_CACHE_ENABLED=0` to disable it. Compose defaults to
disabled unless explicitly enabled.

Operational notes:

-   Cache entries expire after the configured TTL.
-   The cache version creates a namespace/version boundary.
-   Cache read/write failures are designed to fail open.
-   Redis memory is bounded by the Compose configuration and uses an
    eviction policy.
-   A cache hit does not establish that evidence is relevant or that the
    final answer is correct.
-   Before using a shared cache across tenants, review cache-key
    composition, isolation, data sensitivity, and invalidation
    requirements.

Check Redis connectivity:

docker compose exec redis redis-cli ping

Expected output:

PONG

Inspect cache population and memory during authorized local testing:

docker compose exec redis redis-cli DBSIZE
docker compose exec redis redis-cli INFO memory


## Safety and grounding

The pipeline applies input-safety rules, evidence relevance/status
checks, claim validation, answer-grounding decisions, output-safety
checks, and citation validation. Some personalized financial or trading
requests may be restricted rather than answered with individualized
recommendations.

The platform can refuse when evidence is missing, irrelevant,
insufficient, invalid, or when a safety rule requires refusal. This is
intentional behavior, not necessarily an API failure.

These mechanisms are fallible. Citations improve traceability but do not
eliminate hallucinations, stale content, or incorrect source material.

# Knowledge-base synchronization

A Automaticallty triggered GitHub Actions workflow synchronizes content from a
separate knowledge-base repository into Qdrant. It use repository
credentials, Qdrant credentials, synchronization state, and
observability artifacts. Configure GitHub Actions secrets using the
exact names in `.github/workflows/sync.yml`.

Before a production sync:

1.  Confirm the target Qdrant endpoint and collection.
2.  Confirm the source repository revision.
3.  Review dry-run or planning output where supported.
4.  Verify backups and the recovery path.
5.  Confirm expected pre-sync state and validation requirements.
6.  Run only with explicit authorization.
7.  Review the final verification and operational artifacts.

Inspect CLI options with:

py -m scripts.sync.sync_cli --help

Do not trigger a production-impacting workflow merely to test local
setup or documentation.

# Testing

Install development dependencies:

py -m venv .venv
.\.venv\Scripts\Activate.ps1
py -m pip install -r requirements-dev.txt

Run the offline suite using the repository's configured command:

py -m pytest -q scripts tests -m "not integration"

Run focused tests when changing a component. Retrieval-cache changes
should be covered by cache tests and relevant pipeline/safety tests. Use
actual test paths from the current checkout.

Integration tests may require credentials, network access, and
controlled external services. Do not run them against production
resources without approval.

# Troubleshooting

Check service status:

docker compose ps

Read API logs locally:

docker compose logs --tail 200 api

Check Redis:

docker compose exec redis redis-cli ping

  -----------------------------------------------------------------------
  Symptom                             What to check
  ----------------------------------- -----------------------------------
  Compose reports a missing required  Check `.env`, variable names, and
  variable                            `.env.example`; do not print
                                      secrets

  OpenAI returns                      Confirm the key is active, saved
  `401 invalid_api_key`               correctly, and loaded into the
                                      recreated container

  API returns generic `503`           Inspect authorized local logs for
                                      the underlying exception

  Redis connection fails              Check service health, `REDIS_URL`,
                                      Compose network, and Redis logs

  API is healthy but answers fail     Check OpenAI credentials, Qdrant
                                      connectivity, collection settings,
                                      and retrieval logs

  Answer is refused                   Inspect evidence retrieval,
                                      relevance/status checks, safety
                                      decisions, and knowledge-base
                                      coverage

  Cache stays empty                   Ensure caching is enabled and the
                                      request reaches a successful,
                                      cache-eligible retrieval
  -----------------------------------------------------------------------

`docker compose config --quiet` checks Compose syntax and variable
interpolation; it does not validate credentials. The health endpoint
alone does not guarantee successful answer generation.

# Security

-   Keep `.env` and all credentials out of source control.
-   Bind services to private interfaces by default. Use an authenticated
    gateway or equivalent controls before exposing the API publicly.
-   CORS is not authentication; allow only required origins.
-   Do not log secrets, authorization headers, or unnecessary
    personal/sensitive content.
-   Review retrieval filters and cache scope before serving multiple
    tenants.
-   Protect synchronization credentials and review target, state, and
    recovery procedures before production changes.
-   Keep dependencies updated through the normal review and testing
    process.
-   Do not describe the platform as a licensed financial adviser.
-   Review the applicable data-protection and retention requirements
    before processing sensitive information.

# Limitations

-   Answer quality depends on knowledge-base coverage, freshness,
    structure, and correctness.
-   End-to-end answering depends on external model and vector-store
    availability.
-   Refusals may occur when evidence is missing or fails relevance,
    status, or safety checks.
-   Retrieval caching does not replace evidence validation or
    access-control design.

# Custom AI, RAG & Cloud Deployment Services

Need a custom RAG API, AI chatbot, or end-to-end AI solution?

I help businesses and individuals design, develop, integrate, deploy, and maintain custom AI applications based on their requirements.

What I Can Help You Build

Custom RAG Systems: Document-based question answering, knowledge-base assistants, semantic search, and enterprise RAG pipelines.

AI Chatbots & Agents: Website chatbots, business assistants, internal knowledge bots, and AI-powered workflow automation.

RAG APIs & Backend Development: Custom REST APIs, LLM integrations, retrieval pipelines, and production-ready AI backends.

Multi-Provider AI Integration: Integration with OpenAI, Anthropic Claude, Google Gemini, Azure OpenAI, AWS Bedrock, Mistral, Cohere, DeepSeek, Ollama, Hugging Face, and other compatible AI providers, depending on project requirements.

Cloud AI Deployment: Custom AI application deployment on AWS, Microsoft Azure, Google Cloud Platform (GCP), DigitalOcean, and other suitable cloud or hosting platforms.

Infrastructure & DevOps: Docker, Kubernetes, CI/CD pipelines, cloud infrastructure, reverse proxies, monitoring, logging, and production configuration.

Vector Databases & Retrieval: Qdrant and other suitable vector databases, embedding pipelines, reranking, and retrieval optimization.

Existing Project Support: Troubleshooting, API integration, deployment failures, performance optimization, maintenance, and architecture improvements.

Need Something Custom?

Whether you need a complete RAG platform built from scratch, an AI chatbot integrated into your business, an existing AI application deployed to the cloud, or help with a specific technical challenge, I can help you plan and implement a solution tailored to your requirements.

Let's discuss your project.

Portfolio: https://keshav-kumar.online

GitHub: https://github.com/KeshavkumarKK

Email : keshavkumar7376515@gmail.com

Available for freelance development, custom AI solutions, cloud deployment, DevOps, integrations, and technical consulting.

Provider availability and specific model compatibility depend on project requirements, API access, and the selected infrastructure.

## License

See [`LICENSE`](LICENSE) for the applicable license terms.
