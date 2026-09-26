# File structure

```
surplus-match/
├── backend/
├── frontend/
├── docs/
├── .github/workflows/
├── docker-compose.yml
├── .env.example
├── README.md
├── ARCHITECTURE.md
└── FILE_STRUCTURE.md
```

---

## Backend (`/backend`)

```
backend/
├── app/
│   ├── main.py
│   ├── agents/
│   │   ├── base_agent.py
│   │   ├── restaurant_agent.py
│   │   ├── shelter_agent.py
│   │   ├── coordinator_agent.py
│   │   └── logistics_agent.py
│   ├── api/
│   │   ├── routes/
│   │   │   ├── listings.py
│   │   │   ├── shelters.py
│   │   │   ├── restaurants.py
│   │   │   ├── contracts.py
│   │   │   └── auth.py
│   │   └── deps.py
│   ├── core/
│   │   ├── config.py
│   │   ├── database.py
│   │   ├── messaging.py
│   │   └── security.py
│   ├── models/
│   │   ├── listing.py
│   │   ├── shelter.py
│   │   ├── restaurant.py
│   │   ├── bid.py
│   │   └── contract.py
│   ├── schemas/
│   │   ├── listing.py
│   │   ├── shelter.py
│   │   ├── bid.py
│   │   └── contract.py
│   ├── services/
│   │   ├── matching_service.py
│   │   ├── geo_utils.py
│   │   └── notification_service.py
│   └── websocket/
│       └── connection_manager.py
├── alembic/
│   ├── env.py
│   └── versions/
├── tests/
│   ├── test_agents.py
│   ├── test_matching_service.py
│   └── test_api_listings.py
├── requirements.txt
├── Dockerfile
└── alembic.ini
```

| File / folder | Description |
|---|---|
| `app/main.py` | FastAPI app entrypoint — mounts routers, starts the coordinator agent's background listener on startup. |
| `app/agents/base_agent.py` | Shared agent interface: subscribe to a channel, publish a message, lifecycle hooks. |
| `app/agents/restaurant_agent.py` | Builds and publishes a `FoodListing` event when surplus is reported. |
| `app/agents/shelter_agent.py` | Subscribes to listing broadcasts, scores fit (distance/capacity/need), publishes a `Bid`. |
| `app/agents/coordinator_agent.py` | Runs the Contract Net Protocol: broadcasts listings, collects bids within a time window, picks a winner. |
| `app/agents/logistics_agent.py` | Reacts to an awarded contract, creates a pickup/delivery record, notifies both parties. |
| `api/routes/listings.py` | REST endpoints for restaurants to create/view surplus listings. |
| `api/routes/shelters.py` | Endpoints for shelter profile, capacity, and live status updates. |
| `api/routes/restaurants.py` | Endpoints for restaurant profile and listing history. |
| `api/routes/contracts.py` | Endpoints to view awarded contracts and delivery status. |
| `api/routes/auth.py` | Login/signup and JWT issuance for restaurants and shelters. |
| `core/config.py` | Loads environment variables (DB URL, Redis URL, secrets) via pydantic settings. |
| `core/database.py` | SQLAlchemy engine/session setup. |
| `core/messaging.py` | Redis pub/sub client wrapper used by all agents to broadcast/receive events. |
| `core/security.py` | Password hashing, JWT encode/decode. |
| `models/*.py` | SQLAlchemy ORM models for listings, shelters, restaurants, bids, contracts. |
| `schemas/*.py` | Pydantic request/response schemas matching each model. |
| `services/matching_service.py` | Scoring logic (distance, capacity, urgency weighting) shared by agents and API. |
| `services/geo_utils.py` | Haversine distance and ETA estimation helpers. |
| `services/notification_service.py` | Sends confirmation notifications (email/push) once a contract is awarded. |
| `websocket/connection_manager.py` | Tracks connected frontend clients and pushes live listing/bid/contract updates. |
| `alembic/` | Database migration scripts. |
| `tests/` | Unit tests for agents, matching logic, and API routes. |
| `requirements.txt` | Python dependencies. |
| `Dockerfile` | Backend container build. |

---

## Frontend (`/frontend`)

```
frontend/
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── pages/
│   │   ├── RestaurantDashboard.jsx
│   │   ├── ShelterDashboard.jsx
│   │   ├── AdminPanel.jsx
│   │   └── Login.jsx
│   ├── components/
│   │   ├── ListingCard.jsx
│   │   ├── BidStatus.jsx
│   │   ├── MapView.jsx
│   │   └── ContractTimeline.jsx
│   ├── services/
│   │   ├── api.js
│   │   └── websocket.js
│   ├── hooks/
│   │   ├── useLiveListings.js
│   │   └── useAuth.js
│   ├── context/
│   │   └── AuthContext.jsx
│   └── styles/
│       └── index.css
├── public/
├── index.html
├── package.json
├── vite.config.js
└── Dockerfile
```

| File / folder | Description |
|---|---|
| `src/main.jsx` | React app bootstrap. |
| `src/App.jsx` | Top-level routing (restaurant view, shelter view, admin view). |
| `pages/RestaurantDashboard.jsx` | Post surplus listings, see live bid status per listing. |
| `pages/ShelterDashboard.jsx` | View incoming listing broadcasts, see own agent's bid decisions. |
| `pages/AdminPanel.jsx` | Overview of all active listings, contracts, and agent activity. |
| `pages/Login.jsx` | Auth screen for restaurants and shelters. |
| `components/ListingCard.jsx` | Renders a single surplus listing with expiry countdown. |
| `components/BidStatus.jsx` | Shows bid scores and which shelter won a listing. |
| `components/MapView.jsx` | Map showing restaurant and shelter locations for a listing. |
| `components/ContractTimeline.jsx` | Status timeline: posted → bidding → awarded → picked up → delivered. |
| `services/api.js` | REST client (fetch/axios wrapper) for backend endpoints. |
| `services/websocket.js` | WebSocket client for live listing/bid/contract updates. |
| `hooks/useLiveListings.js` | Subscribes to the websocket feed and exposes live listing state. |
| `hooks/useAuth.js` | Auth state and token handling. |
| `context/AuthContext.jsx` | App-wide auth context provider. |
| `package.json` | Frontend dependencies and scripts. |
| `vite.config.js` | Vite build/dev server config. |
| `Dockerfile` | Frontend container build (served via nginx in production). |

---

## Shared / root

| File / folder | Description |
|---|---|
| `docs/` | Additional design notes, ADRs, diagrams. |
| `.github/workflows/` | CI/CD pipeline definitions (see `ARCHITECTURE.md`). |
| `docker-compose.yml` | Runs backend, frontend, Postgres, and Redis together for local dev. |
| `.env.example` | Template for required environment variables. |