# Architecture

## System overview

```
                         ┌─────────────────────┐
                         │   Frontend (React)   │
                         │  Restaurant / Shelter │
                         │     / Admin views     │
                         └──────────┬───────────┘
                          REST +  WebSocket
                                    │
                         ┌──────────▼───────────┐
                         │   FastAPI backend      │
                         │  (routes, auth, ws)    │
                         └──────────┬───────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 │                  │                  │
        ┌────────▼───────┐ ┌───────▼────────┐ ┌───────▼────────┐
        │ Restaurant agent│ │Coordinator agent│ │ Logistics agent│
        └────────┬───────┘ └───────┬────────┘ └───────┬────────┘
                 │                  │                  │
                 └──────── Redis pub/sub bus ──────────┘
                                    │
                         ┌──────────▼───────────┐
                         │   Shelter agents (N)   │
                         │  one per shelter org   │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │   PostgreSQL (state)   │
                         │ listings, bids,        │
                         │ contracts, users        │
                         └────────────────────────┘
```

## Components

- **Frontend** — React SPA. Restaurants post listings; shelters see broadcasts and their
  own agent's bid status; admins see everything. REST for CRUD, WebSocket for live
  listing/bid/contract updates.
- **Backend API (FastAPI)** — auth, CRUD for listings/shelters/restaurants/contracts, and
  the WebSocket gateway that pushes agent activity to connected clients.
- **Agents** — run as async tasks inside the backend process for a single-instance
  deployment, or as separate worker processes/containers at scale. All communicate only
  through the Redis pub/sub bus, never by calling each other directly — this keeps agents
  independently deployable and testable.
  - *Restaurant agent*: publishes `listing.created` events.
  - *Coordinator agent*: subscribes to `listing.created`, publishes `listing.broadcast`,
    collects `bid.submitted` events within a fixed window (e.g. 30s), publishes
    `contract.awarded`.
  - *Shelter agent*: subscribes to `listing.broadcast`, evaluates fit, publishes
    `bid.submitted` (or nothing, if declining).
  - *Logistics agent*: subscribes to `contract.awarded`, creates a delivery record,
    publishes `delivery.confirmed`.
- **Redis** — pub/sub bus for agent-to-agent messaging; can be swapped for a proper
  message broker (e.g. RabbitMQ, Kafka) if throughput grows.
- **PostgreSQL** — source of truth for listings, bids, contracts, and users. Agents write
  their decisions here for audit and to survive restarts.

## Data flow (single listing)

1. Restaurant submits a listing via REST → row inserted → `listing.created` published.
2. Coordinator agent broadcasts `listing.broadcast` to all shelter agents.
3. Each shelter agent independently computes a bid score and publishes `bid.submitted`
   (or stays silent).
4. Coordinator agent waits out the bidding window, picks the highest score, inserts a
   `contract` row, publishes `contract.awarded`.
5. Logistics agent creates a delivery record, publishes `delivery.confirmed`.
6. Backend's WebSocket gateway relays each of these events to connected frontend clients
   in real time.

## Deployment topology

- `docker-compose.yml` runs backend, frontend, Postgres, and Redis together for local dev
  and small deployments.
- For production scale, backend API and agent workers are split into separate services
  (same codebase, different entrypoints) so agents can be scaled independently of the API.

---

## CI/CD pipeline

Implemented with GitHub Actions under `.github/workflows/`.

### `backend-ci.yml` — on PR / push to `backend/**`
1. Checkout code.
2. Set up Python, install `requirements.txt`.
3. Lint (`ruff` / `flake8`) and type-check (`mypy`).
4. Run `pytest` against a Postgres + Redis service container.
5. Build the backend Docker image (no push) to confirm it builds cleanly.

### `frontend-ci.yml` — on PR / push to `frontend/**`
1. Checkout code.
2. Set up Node, `npm install`.
3. Lint (`eslint`) and run `npm test`.
4. `npm run build` to confirm the production build succeeds.

### `deploy.yml` — on push/merge to `main`
1. Re-run backend and frontend CI as a gate.
2. Build and tag backend and frontend Docker images with the commit SHA.
3. Push images to the container registry (e.g. GitHub Container Registry / ECR).
4. Run `alembic upgrade head` against the production database.
5. Deploy new images (e.g. `docker-compose pull && up -d`, or a Kubernetes rollout).
6. Run a post-deploy smoke test (health check on `/health` and the WebSocket endpoint).

### Environments

| Environment | Trigger | Notes |
|---|---|---|
| CI (ephemeral) | Every PR | Spins up Postgres + Redis as service containers, torn down after. |
| Staging | Merge to `develop` | Mirrors production, used for manual QA. |
| Production | Merge to `main` | Requires CI to pass; deploy is gated behind smoke tests. |

### Secrets management

`DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, and registry credentials are stored as GitHub
Actions secrets and injected as environment variables at deploy time — never committed to
the repo.