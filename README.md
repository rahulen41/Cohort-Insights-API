# Cohort Insights API
Cohort Insights API designed using FastAPI, MongoDB, Redis and Docker. The purpose of this project is to accept documents, process their content asynchronously through a two-stage pipeline, and return generated insights such as summaries and tags (FastAPI + MongoDB + Redis service for a two-stage simulated content pipeline).

## Run
```bash
docker compose up --build
```

## Design decisions
- **Staleness:** every content version increments `revision`; results carry `result_revision`. A response is current only when they match. PATCH clears exposed summary/tags immediately, so old output can never be returned as current or mixed with new output.
- **Atomic updates:** PATCH uses compare-and-swap on the previous revision. Clients can provide `expected_revision`; racing updates receive `409` rather than silently overwriting an acknowledged version.
- **Crosswalk:** `client_doc_ref` is a sparse unique index. A partner reference identifies one logical document and remains stable across PATCHes; duplicate references are rejected.
- **Pipeline:** Redis list is the durable handoff queue. Workers claim stage 1, then stage 2. Before each write, the worker rechecks the document revision, so obsolete jobs cannot publish results.
- **Ownership:** user listing always filters by `user_id`; single-document IDs are opaque UUIDs and return the same not-found shape for absent IDs. In a production authenticated deployment, user_id would come from identity claims instead of request input.
- **Caching/rate limiting:** content hashes are stored for deduplication and are safe across updates. IP-minute Redis counters provide a baseline limiter; production would add authenticated subject limits and endpoint-specific budgets.
- **Scale at 100x:** offset pagination becomes increasingly expensive for very large tenants because MongoDB scans/skips older rows; switch list endpoints to a `(user_id, created_at, _id)` cursor. A single Redis list becomes a queue hot key; partition by stage and hash bucket, while preserving per-document revision checks. `client_doc_ref` lookup remains O(log n) through its unique index; shard by a hashed reference if the index exceeds one shard's working set, not by user_id because partner lookups do not know the user.

## Endpoints
- `POST /documents`
- `PATCH /documents/{document_id}`
- `GET /documents/{document_id}`
- `GET /documents/by-ref/{client_doc_ref}`
- `GET /users/{user_id}/documents?page=1&page_size=20&status=complete`
- `GET /health`

## Environment
See `docker-compose.yml`; all settings are configurable via `MONGO_URL`, `MONGO_DB`, `REDIS_URL`, `RATE_LIMIT_PER_MINUTE`, `MAX_CONTENT_CHARS`, and worker concurrency variables.
