# Surplus Match

Real-time matching of restaurant food surplus with nearby shelters, using a multi-agent
negotiation system (Contract Net Protocol) instead of a single centralized matching script.

## Why multi-agent

Each shelter has its own capacity, distance, and urgency — that's private local state.
Rather than one algorithm guessing all of it, shelters run their own agent that bids on
listings independently. A coordinator agent just runs the auction. This makes the system
resilient to no-shows (re-award to the next bid) and easy to extend (add new agent types
without touching the others).

## How it works

1. **Restaurant agent** detects surplus (manual entry or POS webhook) and posts a listing.
2. **Coordinator agent** broadcasts the listing to nearby shelter agents.
3. **Shelter agents** independently evaluate and bid, based on capacity, distance, and need.
4. **Coordinator agent** scores bids and awards the contract to the best fit.
5. **Logistics agent** arranges pickup/delivery and confirms to both sides.

See `ARCHITECTURE.md` for the full system diagram and `FILE_STRUCTURE.md` for the repo layout.

## Tech stack

| Layer | Choice |
|---|---|
| Backend | Python, FastAPI, asyncio |
| Agent messaging | Redis pub/sub |
| Database | PostgreSQL + SQLAlchemy + Alembic |
| Frontend | React + Vite |
| Realtime updates | WebSockets |
| Containerization | Docker, docker-compose |
| CI/CD | GitHub Actions |

## Getting started

### Prerequisites
- Docker and docker-compose
- Node 18+ (for local frontend dev without Docker)
- Python 3.11+ (for local backend dev without Docker)

### Run everything with Docker

```bash
git clone <repo-url>
cd surplus-match
cp .env.example .env
docker-compose up --build
```

- Backend API: `http://localhost:8000`
- Frontend: `http://localhost:5173`
- API docs (Swagger): `http://localhost:8000/docs`

### Run backend locally

```bash
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
alembic upgrade head
uvicorn app.main:app --reload
```

### Run frontend locally

```bash
cd frontend
npm install
npm run dev
```

### Run tests

```bash
cd backend && pytest
cd frontend && npm test
```

## Project structure

See `FILE_STRUCTURE.md` for the full annotated file tree (backend and frontend covered
separately).

## Environment variables

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string for agent pub/sub |
| `JWT_SECRET` | Secret for auth token signing |
| `VITE_API_BASE_URL` | Backend URL the frontend calls |
| `VITE_WS_BASE_URL` | WebSocket URL the frontend connects to |

## License

MIT (adjust as needed).