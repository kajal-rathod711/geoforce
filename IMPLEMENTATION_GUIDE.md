# Implementation Guide - Geofence OS

## System Overview

This document explains how the geofencing and vehicle tracking system works, from frontend to backend.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Browser / Client                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Next.js 16 Frontend (React 19)              │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │  Pages: Dashboard, Geofences, Vehicles, Alerts      │  │
│  │  Components: MapView, Forms, Cards                  │  │
│  │  Styling: Tailwind CSS with Dark Theme              │  │
│  │  Styling: Tailwind CSS with Dark Theme              │  │
│  └──────────────────────────────────────────────────────┘  │
│           │              │              │                  │
│           ├──────────────┼──────────────┤                  │
│           ↓              ↓              ↓                  │
│       ┌────────┐   ┌─────────┐   ┌──────────┐             │
│       │ REST   │   │WebSocket│   │Mock Data │             │
│       │ Client │   │ Client  │   │Fallback  │             │
│       └────────┘   └─────────┘   └──────────┘             │
│
└─────────────────────────────────────────────────────────────┘
                    HTTP + WebSocket
                          │
┌─────────────────────────────────────────────────────────────┐
│                   Go Backend API Server                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │            REST API Endpoints                        │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │  /api/geofences  /api/vehicles  /api/alerts         │  │
│  │  /api/stats      /api/violations /health            │  │
│  └──────────────────────────────────────────────────────┘  │
│           │                          │                     │
│           ├──────────────────────────┤                     │
│           ↓                          ↓                     │
│  ┌──────────────────┐    ┌──────────────────┐             │
│  │ Request Handlers │    │ WebSocket Router │             │
│  └──────────────────┘    └──────────────────┘             │
│           │                          │                     │
│           └──────────────────────────┘                     │
│                      ↓                                      │
│        ┌─────────────────────────────┐                     │
│        │   In-Memory Data Storage    │                     │
│        │  (Maps: Geofences,          │                     │
│        │   Vehicles, Alerts, etc.)   │                     │
│        └─────────────────────────────┘                     │
│                      │                                      │
│        ┌─────────────┴─────────────┐                       │
│        ↓                           ↓                       │
│  ┌─────────────┐         ┌──────────────────┐             │
│  │  Business   │         │    Broadcast     │             │
│  │   Logic     │         │   System (for    │             │
│  │  (collision │         │   WebSocket      │             │
│  │ detection)  │         │   events)        │             │
│  └─────────────┘         └──────────────────┘             │
│
└─────────────────────────────────────────────────────────────┘
```

---

## Data Flow

### 1. Creating a Geofence

```
User Input (Geofence Form)
    ↓
React Hook Form Validation
    ↓
API Call: createGeofence(data)
    ↓
Fetch POST /api/geofences
    ↓
Go Backend: handleGeofences()
    ↓
New Geofence Created in Memory
    ↓
WebSocket Broadcast: "geofence_created"
    ↓
Frontend Receives via WebSocket
    ↓
Update Local State + Re-render
    ↓
Map Updates with New Zone
```

### 2. Vehicle Location Tracking

```
Vehicle GPS Updates
    ↓
Backend: simulateVehicleMovement() goroutine
    ↓
Update Vehicle in Memory (lat, lng)
    ↓
Check if In Restricted Zone (Point-in-Polygon)
    ↓
If Violation: Create Alert
    ↓
Broadcast Alert via WebSocket
    ↓
Frontend Alert Listener
    ↓
Toast Notification + Update Feed
    ↓
Dashboard Map Reflects New Position
```

### 3. Alert System

```
Alert Config Created
    ↓
Stored in Backend Memory
    ↓
Backend Monitors Geofence/Vehicle
    ↓
Condition Triggered (entry/exit/violation)
    ↓
Create Alert Object
    ↓
Broadcast: "alert_created"
    ↓
WebSocket Listener Activated
    ↓
Frontend: Toast + Update Alerts Feed
    ↓
Add to Unresolved Alerts Count
    ↓
User Can Resolve Alert
    ↓
PUT /api/alerts/:id with resolved: true
    ↓
Broadcast: "alert_updated"
    ↓
Frontend Updates UI
```

---

## Component Deep Dive

### MapView Component

```typescript
// Location: components/MapView.tsx

Features:
- Dynamic import with ssr: false (prevents Leaflet hydration issues)
- Leaflet map with CartoDB tile layer
- Polygon rendering for geofences
- CircleMarker rendering for vehicles
- Draw mode for creating new geofences
- Color coding by geofence category

Props:
- geofences: Geofence[]
- vehicles: Vehicle[]
- onMapClick: (lat, lng) => void
- drawMode: boolean
- onPolygonComplete: (coords) => void
- height: string

State:
- drawnCoords: Drawn polygon points
```

### AlertsPanel Component

```typescript
// Location: components/AlertsPanel.tsx

Features:
- Real-time WebSocket connection
- Auto-reconnect on disconnect
- Toast notifications for new alerts
- Slide-in animation for new alerts
- Severity color coding

State:
- alerts: Alert[]
- isConnected: boolean
- loading: boolean
```

### Sidebar Component

```typescript
// Location: components/Sidebar.tsx

Features:
- Navigation to all pages
- Connection status indicator
- Pulsing dot when connected
- Fixed 240px width
- Responsive collapse on mobile

Navigation Items:
- Dashboard (/dashboard)
- Geofences (/geofences)
- Vehicles (/vehicles)
- Alerts (/alerts)
- History (/history)
```

---

## Backend Handler Structure

### Generic Handler Pattern

All handlers follow this pattern:

```go
func handleResource(w http.ResponseWriter, r *http.Request) {
  // 1. Set CORS headers
  w.Header().Set("Access-Control-Allow-Origin", "*")
  w.Header().Set("Content-Type", "application/json")

  // 2. Handle OPTIONS
  if r.Method == http.MethodOptions {
    w.WriteHeader(http.StatusOK)
    return
  }

  // 3. Lock for concurrent access
  mu.RLock()
  defer mu.RUnlock()

  // 4. Route based on method
  if r.Method == http.MethodGet {
    // List or get by ID
  } else if r.Method == http.MethodPost {
    // Create new resource
  } else if r.Method == http.MethodPut {
    // Update resource
  } else if r.Method == http.MethodDelete {
    // Delete resource
  }

  // 5. Return JSON response
  json.NewEncoder(w).Encode(result)
}
```

### WebSocket Broadcasting

```go
// Global channels
var broadcast = make(chan EventData, 256)
var clients = make(map[*websocket.Conn]bool)

// Broadcast goroutine
go func() {
  for event := range broadcast {
    // Send to all connected clients
    for client := range clients {
      client.WriteJSON(event)
    }
  }
}()

// When something changes
broadcast <- EventData{
  Type: "alert_created",
  Payload: alertObject,
}
```

---

## API Layer (Frontend)

### Error Handling Strategy

```typescript
// Location: lib/api.ts

// Pattern: Try real API, fall back to mock data

export const getGeofences = async (): Promise<Geofence[]> => {
  try {
    // Try real API
    return await fetch(`${API_URL}/api/geofences`).then(r => r.json());
  } catch (error) {
    console.log('[v0] API failed, using mock data');
    // Return mock data for development
    return MOCK_GEOFENCES;
  }
};
```

This allows:
- ✅ Full functionality without backend
- ✅ Seamless switch to real API when available
- ✅ Development without deployment
- ✅ Better error messages

---

## Styling System

### CSS Variables

All colors defined as CSS variables in `app/globals.css`:

```css
:root {
  --bg-primary: #0a0f1e;      /* Navy background */
  --bg-card: #1a2235;         /* Darker cards */
  --bg-secondary: #111827;    /* For sidebar */
  --border-color: rgba(6, 182, 212, 0.2);
  --text-primary: #f1f5f9;
  --text-secondary: #94a3b8;
  --cyan: #06b6d4;
  --amber: #f59e0b;
  --red: #ef4444;
  --green: #10b981;
}
```

### Animations

```css
@keyframes pulse-red {
  0%, 100% { opacity: 0.3; }
  50% { opacity: 0.6; }
}

@keyframes slide-in {
  from { transform: translateX(100%); opacity: 0; }
  to { transform: translateX(0); opacity: 1; }
}

@keyframes pulse-dot {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}
```

### Grid Overlay

```css
body::before {
  content: '';
  background-image: 
    linear-gradient(rgba(6,182,212,0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(6,182,212,0.03) 1px, transparent 1px);
  background-size: 40px 40px;
  position: fixed;
  inset: 0;
  pointer-events: none;
}
```

Creates tactical grid overlay throughout app.

---

## Page Implementations

### Dashboard Page

```typescript
// Location: app/dashboard/page.tsx

1. Fetch on mount:
   - getGeofences()
   - getVehicles()
   - getAlerts()
   - getViolations()

2. Display:
   - 4 Stat Cards
   - Interactive Map
   - Live Alerts Feed
   - Recent Violations Table

3. Auto-refresh:
   - 30-second interval
   - Cleanup on unmount
```

### Geofences Page

```typescript
// Location: app/geofences/page.tsx

1. List geofences with filtering
2. Map with draw mode
3. Create form
4. Edit/delete functionality
5. Category filtering
```

### History Page

```typescript
// Location: app/history/page.tsx

1. Filter by:
   - Date range
   - Vehicle
   - Geofence
   - Severity

2. Table with:
   - Pagination
   - Sorting
   - Export capability
   - Detailed view
```

---

## Type Safety

### TypeScript Interfaces

All data models typed in `lib/types.ts`:

```typescript
interface Geofence {
  id: string;
  name: string;
  category: 'delivery_zone' | 'restricted_zone' | ...;
  coordinates: [number, number][];
  created_at: Date;
}

interface Vehicle {
  id: string;
  number: string;
  status: 'active' | 'idle' | 'offline';
  lat: number;
  lng: number;
  speed: number;
  battery: number;
  updated_at: Date;
}
```

### Go Structs

Matching structs in `backend/main.go`:

```go
type Geofence struct {
  ID          string      `json:"id"`
  Name        string      `json:"name"`
  Category    string      `json:"category"`
  Coordinates [][2]float64 `json:"coordinates"`
  CreatedAt   time.Time   `json:"created_at"`
}

type Vehicle struct {
  ID        string    `json:"id"`
  Number    string    `json:"number"`
  Status    string    `json:"status"`
  Lat       float64   `json:"lat"`
  Lng       float64   `json:"lng"`
  Speed     float64   `json:"speed"`
  Battery   int       `json:"battery"`
  UpdatedAt time.Time `json:"updated_at"`
}
```

JSON tags ensure proper serialization between frontend/backend.

---

## Performance Considerations

### Frontend Optimizations

1. **Code Splitting**
   - Pages split by route
   - Components lazy loaded

2. **Dynamic Imports**
   ```typescript
   const MapView = dynamic(() => import('@/components/MapView'), {
     ssr: false,
     loading: () => <LoadingPlaceholder />,
   });
   ```

3. **Suspense Boundaries**
   - Heavy components wrapped in Suspense
   - Smooth loading experience

4. **API Caching**
   - 30-second refresh intervals
   - Prevents unnecessary requests

### Backend Optimizations

1. **Goroutines**
   - Concurrent request handling
   - WebSocket broadcast in separate goroutine

2. **Mutex Locking**
   - RWMutex for concurrent reads
   - Prevents race conditions

3. **Buffered Channels**
   - Broadcast channel: 256 buffer
   - Prevents goroutine blocking

4. **Efficient Algorithms**
   - Point-in-polygon for collision detection
   - O(n) violation checking

---

## Scaling Recommendations

### Phase 1: Development (Current)
- In-memory storage
- Single Go process
- Local development
- Mock data fallback

### Phase 2: Beta
- Add PostgreSQL
- Persistent storage
- Multiple geofences
- Vehicle fleet management
- Real GPS data

### Phase 3: Production
- Redis caching
- Load balancing
- Docker/Kubernetes
- Authentication
- Rate limiting
- Monitoring (Prometheus)
- Logging (ELK)

---

## Testing Strategy

### Manual Testing

1. **Geofence Creation**
   - Create geofence
   - Verify appears on map
   - Verify in list

2. **Vehicle Tracking**
   - Register vehicle
   - Update location
   - Verify on map

3. **Violation Detection**
   - Place vehicle in restricted zone
   - Verify alert generated
   - Verify appears in history

### Automated Testing

```go
// Backend tests
go test ./...

// Frontend tests
pnpm test
```

---

## Troubleshooting Guide

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Map container already initialized" | React 19 strict mode | Use dynamic import + ssr: false |
| API 404 | Wrong endpoint | Check `/api/` prefix |
| WebSocket won't connect | Backend down | Check port 8080 |
| Type mismatch errors | Old types cached | Clear node_modules, reinstall |
| CORS errors | Missing headers | Backend adds CORS headers |

---

## Git Workflow

```bash
# Create feature branch
git checkout -b feature/new-feature

# Make changes
git add .
git commit -m "Add new feature"

# Push to remote
git push origin feature/new-feature

# Create pull request
# Review and merge
```

---

## Summary

The system is built on:

1. **Frontend** - Next.js 16 with React 19, dynamic Leaflet maps, form validation
2. **Backend** - Go HTTP server with WebSocket support, in-memory data, broadcasting
3. **Communication** - REST API + WebSocket for real-time updates
4. **Styling** - Tailwind CSS with dark theme, CSS variables, animations
5. **Type Safety** - TypeScript frontend, typed Go backend, JSON serialization
6. **Error Handling** - Mock data fallback, proper error messages, graceful degradation
7. **Performance** - Code splitting, efficient algorithms, proper locking

Everything works together to provide a smooth, responsive, real-time geofencing experience.
