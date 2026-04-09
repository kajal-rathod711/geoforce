# System Architecture - Geofence OS

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER BROWSER                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌────────────────────────┐                  │
│                    │   Next.js Frontend     │                  │
│                    │  (React 19 + Tailwind) │                  │
│                    └────────────────────────┘                  │
│                            │                                   │
│         ┌──────────────────┼──────────────────┐               │
│         │                  │                  │               │
│         ↓                  ↓                  ↓               │
│    ┌────────────┐  ┌──────────────┐  ┌──────────────┐       │
│    │ REST API   │  │  WebSocket   │  │  Mock Data   │       │
│    │  Client    │  │  Client      │  │  Fallback    │       │
│    └────────────┘  └──────────────┘  └──────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                    HTTP (REST) + WebSocket
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      Go Backend Server                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│              ┌────────────────────────────────┐               │
│              │   HTTP Request Router          │               │
│              │   (port 8080)                  │               │
│              └────────────────────────────────┘               │
│                         │                                      │
│    ┌────────────────────┼────────────────────┐               │
│    │                    │                    │               │
│    ↓                    ↓                    ↓               │
│ ┌──────────┐        ┌──────────┐        ┌──────────┐        │
│ │ Geofence │        │ Vehicle  │        │ Alert    │        │
│ │ Handlers │        │ Handlers │        │ Handlers │        │
│ └──────────┘        └──────────┘        └──────────┘        │
│    │                    │                    │               │
│    └────────────────────┼────────────────────┘               │
│                         │                                      │
│         ┌───────────────┼───────────────┐                    │
│         │               │               │                    │
│         ↓               ↓               ↓                    │
│   ┌─────────────────────────────────────────┐              │
│   │   Concurrent Data Access (sync.RWMutex) │              │
│   └─────────────────────────────────────────┘              │
│         │               │               │                   │
│         ↓               ↓               ↓                   │
│ ┌─────────────┐ ┌────────────┐ ┌──────────────┐           │
│ │  Geofence   │ │  Vehicle   │ │  Alert       │           │
│ │  Map        │ │  Map       │ │  Map         │           │
│ │ (in-memory) │ │ (in-memory)│ │ (in-memory)  │           │
│ └─────────────┘ └────────────┘ └──────────────┘           │
│         │               │               │                   │
│         └───────────────┼───────────────┘                   │
│                         │                                    │
│                    ┌────┴────┐                              │
│                    ↓         ↓                              │
│            ┌──────────────────────────┐                    │
│            │ Business Logic            │                    │
│            │ - Collision Detection     │                    │
│            │ - Violation Creation      │                    │
│            │ - Event Broadcasting      │                    │
│            └──────────────────────────┘                    │
│                         │                                    │
│                         ↓                                    │
│            ┌──────────────────────────┐                    │
│            │ WebSocket Broadcast       │                    │
│            │ (buffers events to all    │                    │
│            │  connected clients)       │                    │
│            └──────────────────────────┘                    │
│                         │                                    │
└─────────────────────────┼────────────────────────────────────┘
                          │
                    WebSocket Events
                          │
                          ↓
                   (back to browser)
```

---

## Data Flow Architecture

### Creation Flow

```
USER INTERACTION (Form Submit)
    ↓
VALIDATION (React Hook Form)
    ↓
API CALL (fetch POST /api/...)
    ↓
GO BACKEND HANDLER
    ├─ Parse JSON body
    ├─ Lock RWMutex
    ├─ Create object with UUID
    ├─ Store in map
    ├─ Unlock RWMutex
    └─ Broadcast event
    ↓
WEBSOCKET BROADCAST
    ├─ Create EventData struct
    ├─ Send to broadcast channel
    ├─ Goroutine distributes to all clients
    └─ Returns response JSON to originating request
    ↓
FRONTEND RECEIVES
    ├─ REST response (for immediate feedback)
    └─ WebSocket event (for real-time update)
    ↓
UPDATE STATE
    ├─ Add to local state
    └─ Re-render component
    ↓
UI UPDATE
    └─ New item appears in list/map
```

### Real-Time Update Flow

```
BACKEND SIMULATION GOROUTINE
    ├─ Every 5 seconds:
    │  ├─ Lock RWMutex
    │  ├─ Update vehicle positions
    │  ├─ Check geofence violations
    │  ├─ Create alerts if needed
    │  └─ Unlock RWMutex
    └─ Broadcast events
    ↓
EVENT TYPES:
    ├─ vehicle_updated
    ├─ vehicle_in_restricted
    ├─ alert_created
    └─ geofence_crossed
    ↓
WEBSOCKET BROADCAST
    ├─ Buffer events (256 capacity)
    ├─ Send to all connected clients
    └─ Handle client disconnections
    ↓
FRONTEND WEBSOCKET LISTENER
    ├─ Receive event
    ├─ Parse event type
    ├─ Update local state
    ├─ Show toast notification
    └─ Re-render UI components
    ↓
USER SEES UPDATES LIVE
    ├─ Vehicle moves on map
    ├─ New alert appears
    ├─ Stats update
    └─ Toast notification slides in
```

---

## Component Architecture

### Frontend Layer

```
                        app/layout.tsx
                      (Root Layout)
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ↓                   ↓                   ↓
    Sidebar           MainContent           Toaster
    (Navigation)      (Page Content)        (Notifications)
        │                   │
        ├───────────────────┤
        │
    Pages (app/**/page.tsx):
    ├── dashboard/page.tsx ─────→ MapView + Stats + Alerts + Violations
    ├── geofences/page.tsx ─────→ GeofenceForm + GeofenceList + MapView
    ├── vehicles/page.tsx ──────→ VehicleForm + VehicleList + MapView
    ├── alerts/page.tsx ────────→ AlertConfigForm + AlertConfigList
    └── history/page.tsx ───────→ ViolationTable + Filters

    Components (components/):
    ├── MapView.tsx ────→ Leaflet Map (dynamic, ssr: false)
    ├── Sidebar.tsx ────→ Navigation + Connection Status
    ├── AlertsPanel.tsx ─→ Real-time Alert Feed
    ├── StatsCard.tsx ──→ Stat Display Cards
    ├── GeofenceForm.tsx ─→ Create/Edit Geofence
    ├── VehicleForm.tsx ──→ Register Vehicle
    └── AlertConfigForm.tsx → Configure Alert Rules

    Utilities (lib/):
    ├── api.ts ────────→ REST Client + Mock Fallback
    ├── types.ts ──────→ TypeScript Interfaces
    └── websocket.ts ──→ WebSocket Manager
```

### Backend Layer

```
                    main.go Entry Point
                            │
    ┌───────────────────────┼───────────────────────┐
    │                       │                       │
    ↓                       ↓                       ↓
Global State         HTTP Server            Goroutines
├─ geofences map     (port 8080)           ├─ broadcastLoop()
├─ vehicles map      ├─ CORS middleware    ├─ simulateVehicleMovement()
├─ alerts map        ├─ Request Router     └─ handleWebSocket()
├─ alertConfigs map  └─ Handlers
├─ clients map
├─ broadcast channel
└─ sync.RWMutex

Request Handlers (handleXXX functions):
├─ handleGeofences() ────→ GET/POST /api/geofences
├─ handleGeofenceDetail() → GET/DELETE /api/geofences/:id
├─ handleVehicles() ─────→ GET/POST /api/vehicles
├─ handleVehicleDetail() → GET/PUT /api/vehicles/:id
├─ handleAlerts() ──────→ GET/POST /api/alerts
├─ handleAlertDetail() ──→ GET/PUT /api/alerts/:id
├─ handleAlertConfigs() → GET/POST /api/alert-configs
├─ handleAlertConfigDetail() → GET/DELETE /api/alert-configs/:id
├─ handleStats() ───────→ GET /api/stats
├─ handleViolations() ──→ GET /api/violations
└─ handleWebSocket() ───→ WS /ws

Utility Functions:
├─ countActiveVehicles()
├─ countUnresolvedAlerts()
├─ isPointInPolygon()
└─ corsMiddleware()
```

---

## Data Model Architecture

### Type Definitions (Shared)

```
TypeScript (lib/types.ts) ←→ Go Structs (backend/main.go)

Geofence
├─ id: string
├─ name: string
├─ category: string
├─ coordinates: [][2]float64
└─ created_at: Date

Vehicle
├─ id: string
├─ number: string
├─ status: string
├─ lat: float64
├─ lng: float64
├─ speed: float64
├─ battery: int
└─ updated_at: Date

Alert
├─ id: string
├─ type: string
├─ severity: string
├─ vehicle_id: string
├─ geofence_id: string
├─ message: string
├─ timestamp: Date
└─ resolved: boolean

AlertConfig
├─ id: string
├─ name: string
├─ type: string
├─ geofences: []string
├─ vehicles: []string
├─ enabled: boolean
└─ created_at: Date
```

---

## State Management Flow

### Frontend State

```
React State (useState hooks):
├─ Page Level
│  ├─ geofences: Geofence[]
│  ├─ vehicles: Vehicle[]
│  ├─ alerts: Alert[]
│  ├─ loading: boolean
│  └─ formOpen: boolean
├─ Form Level (React Hook Form)
│  ├─ name: string
│  ├─ category: string
│  └─ formState.errors
└─ WebSocket Level
   ├─ isConnected: boolean
   ├─ receivedAlerts: Alert[]
   └─ lastEventTime: Date

State Updates:
├─ Via API calls → setGeofences(), setVehicles(), etc.
├─ Via WebSocket → real-time updates from backend
├─ Via Form → local state management
└─ Via Component Props → parent to child data flow
```

### Backend State

```
In-Memory Maps (protected by RWMutex):
├─ geofences: map[string]*Geofence
├─ vehicles: map[string]*Vehicle
├─ alerts: map[string]*Alert
└─ alertConfigs: map[string]*AlertConfig

Connected Clients:
├─ clients: map[*websocket.Conn]bool
├─ broadcast: chan EventData (buffer: 256)
└─ mu: sync.RWMutex (protects all above)

Event Broadcasting:
├─ When data changes → send to broadcast channel
├─ broadcastLoop() goroutine → distributes to all clients
├─ Client disconnection → remove from clients map
└─ Graceful closure on server shutdown
```

---

## Styling System Architecture

```
app/globals.css
├─ CSS Variables (4 sections)
│  ├─ Color System (8 colors + border)
│  ├─ Animation Definitions (3 keyframes)
│  ├─ Grid Overlay (tactical aesthetic)
│  └─ Font Configuration (in layout.tsx)
│
├─ Color Tokens
│  ├─ --bg-primary: #0a0f1e (main background)
│  ├─ --bg-card: #1a2235 (component backgrounds)
│  ├─ --cyan: #06b6d4 (primary accent)
│  ├─ --amber: #f59e0b (warning)
│  ├─ --red: #ef4444 (alert/error)
│  ├─ --green: #10b981 (success)
│  ├─ --text-primary: #f1f5f9 (main text)
│  └─ --text-secondary: #94a3b8 (secondary text)
│
└─ Tailwind CSS v4
   ├─ Responsive prefixes (sm:, md:, lg:, xl:)
   ├─ Utility classes (flex, grid, gap, etc.)
   ├─ Custom shadows & borders
   └─ Dark mode support
```

---

## Concurrency Architecture

### Backend Concurrency

```
Main Thread:
├─ http.ListenAndServe (blocks until shutdown)
└─ Each request gets own goroutine from HTTP server

Dedicated Goroutines:
├─ broadcastLoop() 
│  ├─ Receives from broadcast channel
│  ├─ Distributes to all WebSocket clients
│  └─ Removes disconnected clients
│
├─ simulateVehicleMovement()
│  ├─ Timer: every 5 seconds
│  ├─ Lock RWMutex
│  ├─ Update vehicle positions
│  ├─ Check for violations
│  ├─ Broadcast events
│  └─ Unlock RWMutex
│
└─ Per-WebSocket Connection
   ├─ handleWebSocket() goroutine
   ├─ Reads messages from client
   ├─ Can send data to broadcast channel
   └─ Cleanup on disconnect

Synchronization:
├─ sync.RWMutex (mu)
│  ├─ Multiple readers allowed (Lock/Unlock for reads)
│  ├─ Single writer (Lock/Unlock for writes)
│  └─ Prevents race conditions on shared state
├─ Channels
│  ├─ broadcast chan (buffered, 256 events)
│  └─ Allows goroutine communication
└─ Atomic operations where needed
```

---

## API Contract

### Request/Response Pattern

```
REQUEST (Client → Server):
GET /api/geofences HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{} (empty body for GET)

RESPONSE (Server → Client):
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Content-Type: application/json

[
  {
    "id": "geo1",
    "name": "Delivery Zone",
    "category": "delivery_zone",
    "coordinates": [[40.7128, -74.006], ...],
    "created_at": "2026-04-09T07:31:00Z"
  }
]
```

---

## Deployment Architecture

### Docker Compose Setup

```
docker-compose.yml
├─ backend service
│  ├─ Build from backend/Dockerfile
│  ├─ Exposes port 8080
│  ├─ Named: "backend"
│  └─ Network: geofence-network
│
├─ frontend service
│  ├─ Build from ./Dockerfile
│  ├─ Exposes port 3000
│  ├─ Environment variables:
│  │  ├─ NEXT_PUBLIC_API_URL=http://backend:8080
│  │  └─ NEXT_PUBLIC_WS_URL=ws://backend:8080
│  ├─ Depends on: backend (healthy)
│  └─ Network: geofence-network
│
└─ Network: geofence-network
   └─ Services communicate internally
```

---

## Performance Optimization Strategy

```
Frontend Optimizations:
├─ Dynamic Imports
│  └─ Leaflet loaded only on client (ssr: false)
├─ Suspense Boundaries
│  └─ MapView loads in background
├─ API Caching
│  └─ 30-second refresh intervals
├─ Code Splitting
│  └─ Each page separate bundle
└─ Mock Data Fallback
   └─ App works without backend

Backend Optimizations:
├─ Goroutines
│  └─ Concurrent request handling
├─ RWMutex
│  └─ Multiple concurrent readers
├─ Buffered Channels
│  └─ Broadcast buffer (256 events)
├─ In-Memory Storage
│  └─ O(1) lookups by ID
└─ Efficient Algorithms
   └─ Point-in-polygon for collisions
```

---

## Error Handling Flow

```
FRONTEND ERROR HANDLING:
API Call
├─ Success → Update State
├─ Network Error → Use Mock Data
├─ Parse Error → Log + Show Toast
├─ Validation Error → Show Form Error
└─ Server Error → Show Toast + Retry

BACKEND ERROR HANDLING:
Request
├─ Invalid Method → 405 Method Not Allowed
├─ Invalid JSON → 400 Bad Request
├─ Resource Not Found → 404 Not Found
├─ Server Error → 500 Internal Server Error
└─ Success → 200/201 OK/Created

WEBSOCKET ERROR HANDLING:
Connection
├─ Failed to connect → Use polling fallback
├─ Connection lost → Auto-reconnect
├─ Invalid message → Log + ignore
└─ Server shutdown → Graceful close
```

---

This architecture provides:
- ✅ Separation of concerns
- ✅ Type safety end-to-end
- ✅ Real-time capabilities
- ✅ Scalability path
- ✅ Fault tolerance
- ✅ Developer experience
