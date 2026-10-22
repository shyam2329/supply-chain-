# LogiSphere AI — Smart Supply Chain Visibility & Digital Twin Platform

> "See Every Movement. Control Every Journey."

A full-stack logistics platform with a real-time 3D Digital Twin: shipment
status changes in the database physically move trucks in a 3D garage/road
scene on the frontend, powered by Redis pub/sub.

## Stack

| Layer      | Tech                                              |
|------------|----------------------------------------------------|
| Frontend   | React, TypeScript, Vite, Tailwind CSS, React Three Fiber, Three.js, Framer Motion |
| Backend    | Node.js, Express, TypeScript, REST API, WebSocket (Socket.IO) |
| Database   | PostgreSQL                                        |
| Cache/Events | Redis (caching + pub/sub for live shipment events) |
| DevOps     | Docker, Docker Compose                            |

## API Reference

All routes are mounted under `/api` and (except `/api/auth/login`) require
`Authorization: Bearer <token>`.

| Method | Route                          | Description                                      |
|--------|---------------------------------|---------------------------------------------------|
| POST   | `/api/auth/login`               | Returns `{ token, user }`                          |
| GET    | `/api/auth/me`                  | Current user from the token                        |
| GET    | `/api/dashboard/summary`        | Aggregated dashboard stats (real PostgreSQL data)  |
| GET    | `/api/shipments`                | List all shipments                                 |
| GET    | `/api/shipments/:id/timeline`   | Event history for one shipment                      |
| PATCH  | `/api/shipments/:id/status`     | Change status (validated transition + zod body)     |
| GET    | `/api/warehouses`               | List warehouses with capacity/bay stats             |
| GET    | `/api/warehouses/:id`           | Single warehouse                                    |
| GET    | `/api/inventory`                | List inventory joined with product/warehouse        |
| PATCH  | `/api/inventory/:id/adjust`     | Stock in/out (`change_qty`, `reason`)                |
| GET    | `/api/suppliers`                | List suppliers with computed product counts          |
| GET    | `/api/analytics`                | All chart data (status/utilization/health/etc.)      |
| GET    | `/api/digital-twin`             | Combined `{ warehouses, shipments }` — the Digital Twin's initial load; live updates after that arrive over Socket.IO, not this endpoint |
| POST   | `/api/simulation/disrupt`       | `{ shipmentId, type }` → simulated disruption + AI insight |

Note: `/api/dashboard/summary`, not `/api/dashboard` — the summary is
nested under the resource in case other `/api/dashboard/*` endpoints are
added later (e.g. per-role dashboards).



```
logisphere-ai/
├── docker-compose.yml
├── .env.example
├── frontend/                # React + Vite app
│   ├── Dockerfile
│   ├── src/
│   │   ├── components/      # Reusable UI (Sidebar, Navbar, Cards, Tables)
│   │   ├── pages/            # Dashboard, Inventory, Shipments, Warehouses,
│   │   │                     # Suppliers, Analytics, DigitalTwin, Landing, Login
│   │   ├── three/            # All React Three Fiber scenes & 3D components
│   │   │   ├── scenes/       # LandingScene.tsx, DigitalTwinScene.tsx
│   │   │   ├── objects/      # Truck.tsx, Warehouse.tsx, Garage.tsx, Route.tsx
│   │   │   └── hooks/        # useCursorParallax.ts, useTruckAnimation.ts
│   │   ├── api/               # Axios/fetch clients per resource
│   │   ├── hooks/             # useShipmentSocket.ts (live updates), etc.
│   │   ├── types/             # Shared TS types (Shipment, Warehouse, ...)
│   │   └── styles/
│   └── ...
├── backend/                  # Express API
│   ├── Dockerfile
│   ├── src/
│   │   ├── config/            # env, db pool, redis client
│   │   ├── db/                # schema.sql, migrations, seed data
│   │   ├── models/            # DB query layer per table
│   │   ├── controllers/       # request handlers
│   │   ├── routes/            # /api/shipments, /api/warehouses, etc.
│   │   ├── middleware/        # auth (JWT + roles), error handling
│   │   ├── redis/             # publisher + subscriber setup
│   │   └── services/          # business logic (e.g. shipment status engine)
│   └── ...
└── README.md
```

## Roles

- **Admin** — full access to everything.
- **Warehouse Manager** — inventory, incoming/outgoing stock, warehouse ops.
- **Supplier** — purchase orders, delivery status updates, pending requests.

## Running the project

```bash
cp .env.example .env
docker compose down -v   # -v matters if you've run this before — the schema has changed
docker compose up --build
```

- Frontend → http://localhost:5173
- Backend API → http://localhost:4000
- PostgreSQL → localhost:5432
- Redis → localhost:6379

## How the Digital Twin works (the core unique feature)

**Initial load:** the Digital Twin page calls `GET /api/digital-twin` once
for its starting state (`{ warehouses, shipments }` in a single round trip).

**Live updates** after that:

1. A user (Admin/Warehouse Manager) updates a shipment's status via the API
   (e.g. `PATCH /api/shipments/:id/status` → `dispatched`). The request body
   is validated with zod, and the transition (e.g. `pending -> dispatched`)
   is checked against an explicit allowed-transitions map before anything
   is written.
2. The backend writes to PostgreSQL and commits. Only *after* the commit
   does it publish an event to a Redis channel (`shipment:events`) — if
   that publish fails, the already-committed database write is not rolled
   back or reported as a failure; the response includes
   `realtimeDelivered: false` instead so the client can flag it.
3. The backend's Socket.IO layer is subscribed to that Redis channel and
   pushes the event to connected frontend clients instantly.
4. The frontend's `useShipmentSocket` hook receives the event; both that
   event and the PATCH's own response are merged directly into React
   state (not a refetch, which would risk a race between a stale GET and
   a newer Socket.IO event). The `DigitalTwinScene` reacts by animating
   the corresponding 3D truck (drive out of garage → move along route →
   stop at destination / flash red if delayed).

This is what makes the 3D scene a *functional* digital twin rather than a
decorative animation — next step is building out `db/schema.sql` and the
Express API routes.
