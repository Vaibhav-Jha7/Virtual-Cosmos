# Cosmos Backend

Real-time proximity chat server for the **Cosmos** virtual space.

---

## Tech Stack & Justification

| Layer | Choice | Why |
|---|---|---|
| Runtime | **Node.js** | Single-threaded event loop is ideal for high-frequency WebSocket I/O — no thread-per-connection overhead |
| Framework | **Express** | Minimal, battle-tested, pairs seamlessly with `http.createServer` that Socket.IO needs |
| Real-time | **Socket.IO** | Handles WebSocket + long-polling fallback, rooms, acknowledgements, and reconnection out-of-the-box |
| Database | **MongoDB + Mongoose** | Schema-flexible, native JSON, TTL indexes for session cleanup, aggregation pipeline for chat room queries |
| In-memory state | **Custom StateManager** | Position updates arrive ~60×/s per user — hitting MongoDB at that rate is wasteful. Hot state lives in RAM; only meaningful events (join, disconnect, message) are persisted |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        CLIENTS                          │
│   Browser ←──── Socket.IO ────→ Browser                │
└────────────────────┬────────────────────────────────────┘
                     │ WebSocket / HTTP
┌────────────────────▼────────────────────────────────────┐
│                  EXPRESS SERVER                         │
│                                                         │
│  ┌─────────────────┐    ┌──────────────────────────┐   │
│  │  REST API        │    │  Socket.IO handlers       │   │
│  │  /api/users      │    │  cosmos:join              │   │
│  │  /api/chat       │    │  cosmos:move              │   │
│  │  /api/health     │    │  cosmos:proximity         │   │
│  └────────┬─────────┘    │  cosmos:message           │   │
│           │              └──────────┬───────────────┘   │
│           │                         │                   │
│           │         ┌───────────────▼──────────┐        │
│           │         │    StateManager (RAM)     │        │
│           │         │  users Map  rooms Map     │        │
│           │         └───────────────┬──────────┘        │
│           │                         │                   │
│  ┌────────▼─────────────────────────▼──────────┐        │
│  │                  MongoDB                     │        │
│  │   users collection   chatmessages collection │        │
│  └─────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────┘
```

### Proximity flow

```
Client A moves
      │
      ▼
cosmos:move ──────► StateManager.updatePosition(A)
                         │ broadcast to all clients
                         ▼
                  cosmos:user_moved → Client B, C…

Client A runs distance check every 200 ms
      │
      ├─ User B entered radius?
      │       cosmos:proximity { entered: [B] }
      │             │
      │             ▼
      │       StateManager.openRoom(A, B)
      │       ├─ emit cosmos:connected → A
      │       ├─ emit cosmos:connected → B
      │       └─ emit cosmos:history  → both
      │
      └─ User B exited radius?
              cosmos:proximity { exited: [B] }
                    │
                    ▼
              StateManager.closeRoom(A, B)
              ├─ emit cosmos:disconnected → A
              └─ emit cosmos:disconnected → B
```

---

## Quick Start

### 1. Prerequisites

- Node.js ≥ 18
- MongoDB running locally **or** a MongoDB Atlas URI

### 2. Install

```bash
git clone <repo>
cd cosmos-backend
npm install
```

### 3. Configure

```bash
cp .env.example .env
# Edit .env — set MONGODB_URI and ALLOWED_ORIGINS
```

### 4. Run

```bash
# Development (auto-restart on save)
npm run dev

# Production
npm start
```

### 5. Open the frontend

Open `cosmos-frontend.html` in your browser (or serve it with any static server).  
Make sure `BACKEND_URL` in the HTML matches your server address.

---

## Project Structure

```
cosmos-backend/
├── src/
│   ├── server.js               ← Express + Socket.IO entry point
│   ├── config/
│   │   └── db.js               ← MongoDB connection with retry
│   ├── models/
│   │   ├── User.js             ← User schema (identity, position, connections)
│   │   └── ChatMessage.js      ← Chat message schema (per-room, TTL 7d)
│   ├── routes/
│   │   └── api.js              ← REST endpoints
│   ├── socket/
│   │   ├── socketHandler.js    ← All Socket.IO event handlers
│   │   └── StateManager.js     ← In-memory hot state (singleton)
│   └── middleware/
│       └── rateLimiter.js      ← Sliding-window rate limiter
├── tests/
│   ├── unit/
│   │   └── StateManager.test.js
│   └── integration/
│       └── api.test.js
├── cosmos-frontend.html        ← Drop-in frontend (no build step)
├── .env.example
├── jest.config.json
└── package.json
```

---

## Socket.IO Event Reference

### Client → Server

| Event | Payload | Description |
|---|---|---|
| `cosmos:join` | `{ userId, name, color, position }` | Register on entry |
| `cosmos:move` | `{ x, y }` | Broadcast position (throttled to 20fps) |
| `cosmos:proximity` | `{ entered: string[], exited: string[] }` | Report proximity changes |
| `cosmos:message` | `{ roomId, text }` | Send a chat message |

### Server → Client

| Event | Payload | Description |
|---|---|---|
| `cosmos:welcome` | `{ user, onlineUsers[] }` | Sent to joiner only |
| `cosmos:user_joined` | `{ user }` | Broadcast to all others |
| `cosmos:user_moved` | `{ userId, position }` | Broadcast position update |
| `cosmos:connected` | `{ roomId, peer }` | Proximity room opened |
| `cosmos:disconnected` | `{ roomId, peerId }` | Proximity room closed |
| `cosmos:history` | `{ roomId, messages[] }` | Chat history on connect |
| `cosmos:message` | `{ roomId, message }` | New message delivery |
| `cosmos:user_left` | `{ userId }` | User disconnected |
| `cosmos:online_count` | `{ count }` | Live user count update |

---

## REST API Reference

| Method | Path | Description |
|---|---|---|
| GET | `/api/health` | Liveness check |
| GET | `/api/users/online` | All online users (live from RAM) |
| GET | `/api/users/:userId` | Single user profile |
| GET | `/api/chat/history?userA=&userB=` | Message history for a pair |
| GET | `/api/chat/rooms/:userId` | All rooms for a user |

---

## Running Tests

```bash
npm test

# With coverage report
npm test -- --coverage
```

---

## Scaling Notes

For a single-server deployment this architecture handles hundreds of concurrent users comfortably. To scale horizontally:

1. **Sticky sessions** — configure your load balancer to route each client to the same Node process (Socket.IO requires this without an adapter).
2. **Socket.IO Redis Adapter** — replace the in-process broadcast with `@socket.io/redis-adapter` so events fan out across multiple Node processes.
3. **StateManager** — migrate to Redis (e.g. `ioredis`) for the hot user/room state so all processes share one source of truth.
4. **MongoDB** — already horizontally scalable via replica sets / Atlas.