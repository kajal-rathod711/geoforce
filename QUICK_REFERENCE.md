# Quick Reference - Geofence OS

## 🚀 Start Development (2 min)

```bash
# Terminal 1: Backend
cd backend && go run main.go

# Terminal 2: Frontend (new terminal)
pnpm install && pnpm dev

# Open http://localhost:3000
```

## 📊 Dashboard Pages

| Page | URL | Purpose |
|------|-----|---------|
| Dashboard | `/dashboard` | Real-time map with vehicles & alerts |
| Geofences | `/geofences` | Create and manage zones |
| Vehicles | `/vehicles` | Register and track vehicles |
| Alerts | `/alerts` | Configure alert rules |
| History | `/history` | View violation history |

## 🔌 API Endpoints

```
GET    /api/geofences              → List all geofences
POST   /api/geofences              → Create geofence
DELETE /api/geofences/:id          → Delete geofence

GET    /api/vehicles               → List all vehicles
POST   /api/vehicles               → Register vehicle
PUT    /api/vehicles/:id           → Update vehicle

GET    /api/alerts                 → List alerts
POST   /api/alerts                 → Create alert
PUT    /api/alerts/:id             → Resolve alert

GET    /api/alert-configs          → List configs
POST   /api/alert-configs          → Create config
DELETE /api/alert-configs/:id      → Delete config

GET    /api/stats                  → System stats
GET    /api/violations             → Current violations
GET    /health                     → Health check
WS     /ws                         → WebSocket events
```

## 🎨 Styling System

```css
/* Colors */
--bg-primary: #0a0f1e;      /* Deep navy background */
--bg-card: #1a2235;         /* Card backgrounds */
--cyan: #06b6d4;            /* Primary accent */
--amber: #f59e0b;           /* Warning color */
--red: #ef4444;             /* Alert color */
--green: #10b981;           /* Success color */
--text-primary: #f1f5f9;    /* Main text */
--text-secondary: #94a3b8;  /* Secondary text */
--border-color: rgba(6, 182, 212, 0.2);
```

## 📁 Key Files

```
Frontend
├── app/page.tsx              → Entry (redirects to dashboard)
├── app/dashboard/page.tsx    → Dashboard main page
├── components/MapView.tsx    → Leaflet map
├── components/AlertsPanel.tsx→ Alert feed
├── lib/api.ts               → API client
└── lib/types.ts             → Type definitions

Backend
├── backend/main.go          → Complete API server
├── backend/go.mod           → Dependencies
└── backend/go.sum           → Lock file
```

## 🔧 Common Tasks

### Add a New Page
```typescript
// app/mypage/page.tsx
'use client';
import { useState, useEffect } from 'react';

export default function MyPage() {
  return (
    <div className="p-8">
      <h1 className="text-3xl font-bold">My Page</h1>
    </div>
  );
}
```

### Create a Form
```typescript
import { useForm } from 'react-hook-form';

export default function MyForm() {
  const { register, handleSubmit } = useForm();
  
  const onSubmit = async (data) => {
    const result = await createSomething(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Add API Endpoint (Backend)
```go
// In backend/main.go
func handleMyRoute(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Access-Control-Allow-Origin", "*")
  w.Header().Set("Content-Type", "application/json")

  if r.Method == http.MethodGet {
    json.NewEncoder(w).Encode(map[string]string{"message": "ok"})
  }
}

// In main():
http.HandleFunc("/api/myroute", handleMyRoute)
```

## 🐛 Troubleshooting

| Issue | Solution |
|-------|----------|
| Port 3000 in use | Kill process: `lsof -i :3000` then `kill -9 <PID>` |
| Port 8080 in use | Kill process: `lsof -i :8080` then `kill -9 <PID>` |
| Map not loading | Check MapView is dynamic: `dynamic(() => import(...), {ssr: false})` |
| API 404 errors | Ensure backend running on 8080, check endpoint in `lib/api.ts` |
| TypeScript errors | Run `pnpm install` to install all deps |
| Build fails | Delete `.next` folder: `rm -rf .next` and rebuild |

## 📦 Dependencies

### Frontend (Top 10)
```json
{
  "next": "16.2.0",
  "react": "19.2.4",
  "tailwindcss": "4.0.0-alpha.15",
  "leaflet": "1.9.4",
  "react-leaflet": "4.2.1",
  "react-hook-form": "^7.49.1",
  "zod": "^3.22.4",
  "sonner": "^1.2.3",
  "lucide-react": "^0.376.0",
  "date-fns": "^2.30.0"
}
```

### Backend (All)
```
github.com/google/uuid v1.6.0
github.com/gorilla/websocket v1.5.1
```

## 🌐 Environment Variables

```env
NEXT_PUBLIC_API_URL=http://localhost:8080
NEXT_PUBLIC_WS_URL=ws://localhost:8080
```

## 🚢 Deployment

### Docker (Recommended)
```bash
docker-compose up --build
# Frontend: http://localhost:3000
# Backend: http://localhost:8080
```

### Manual
```bash
# Backend
cd backend && go build -o api . && ./api

# Frontend
pnpm build && pnpm start
```

## 📚 Documentation

- `README.md` - Feature overview
- `SETUP.md` - Detailed setup instructions
- `BUILD_SUMMARY.md` - What was built
- `backend/README.md` - API documentation
- This file - Quick reference

## 💡 Tips

1. **Use Mock Data** - App works without backend, perfect for UI development
2. **Dynamic Maps** - Leaflet components use `ssr: false` to avoid hydration issues
3. **WebSocket** - Auto-reconnecting, works with or without backend
4. **Hot Reload** - Next.js/Go both support hot reloading
5. **Type Safety** - TypeScript frontend + typed Go backend

## 🎯 Next Steps

1. Run the app: `cd backend && go run main.go` + `pnpm dev`
2. Create a geofence in `/geofences`
3. Register a vehicle in `/vehicles`
4. Set up an alert in `/alerts`
5. View everything on `/dashboard`

---

**Ready to build? Start with:** `cd backend && go run main.go` (Terminal 1) + `pnpm dev` (Terminal 2)
