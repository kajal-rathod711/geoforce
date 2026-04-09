# Geofence OS - Industrial Vehicle Tracking Dashboard

A production-grade geofencing and vehicle tracking system with real-time alerts, built with Next.js (frontend) and Go (backend).

## Features

✅ **Real-time Vehicle Tracking** - Live location updates via WebSocket  
✅ **Geofence Management** - Create, edit, and monitor geofences with multiple categories  
✅ **Smart Alerts** - Configurable alert rules with severity levels  
✅ **Interactive Map** - Leaflet-based map with vehicle markers and geofence boundaries  
✅ **Violation History** - Comprehensive tracking with date filtering and pagination  
✅ **Tactical Dark UI** - Mission control aesthetic with cyan accents  
✅ **Responsive Design** - Mobile-first, works on all devices  

## Project Structure

```
.
├── app/                    # Next.js 16 frontend
│   ├── dashboard/         # Dashboard with real-time map
│   ├── geofences/         # Geofence management
│   ├── vehicles/          # Vehicle registry
│   ├── alerts/            # Alert configuration
│   ├── history/           # Violation history
│   └── layout.tsx         # Root layout with sidebar
├── components/            # React components
│   ├── MapView.tsx        # Interactive Leaflet map
│   ├── Sidebar.tsx        # Navigation sidebar
│   ├── AlertsPanel.tsx    # Real-time alerts feed
│   └── Forms/             # Create/edit forms
├── lib/
│   ├── types.ts           # TypeScript interfaces
│   ├── api.ts             # API layer with mock fallback
│   └── websocket.ts       # WebSocket client
├── backend/               # Go backend
│   ├── main.go            # REST API + WebSocket server
│   ├── go.mod             # Go dependencies
│   └── README.md           # Backend documentation
└── public/                # Static assets
```

## Quick Start

### Frontend Setup

```bash
# Install dependencies
npm install
# or
pnpm install

# Start dev server
npm run dev
# or
pnpm dev
```

Frontend runs on: `http://localhost:3000`

### Backend Setup

```bash
cd backend

# Download dependencies
go mod download

# Run the server
go run main.go
```

Backend runs on: `http://localhost:8080`

> **Note**: The frontend works without a backend - it will use mock data if the API is unavailable.

## Environment Variables

Create a `.env.local` file in the root:

```env
NEXT_PUBLIC_API_URL=http://localhost:8080
NEXT_PUBLIC_WS_URL=ws://localhost:8080
```

## API Endpoints

### Geofences
- `GET /api/geofences` - List all geofences
- `POST /api/geofences` - Create new geofence
- `GET /api/geofences/:id` - Get geofence details
- `DELETE /api/geofences/:id` - Delete geofence

### Vehicles
- `GET /api/vehicles` - List all vehicles
- `POST /api/vehicles` - Register new vehicle
- `GET /api/vehicles/:id` - Get vehicle details
- `PUT /api/vehicles/:id` - Update vehicle location/status

### Alerts
- `GET /api/alerts` - List all alerts
- `POST /api/alerts` - Create new alert
- `GET /api/alerts/:id` - Get alert details
- `PUT /api/alerts/:id` - Update alert (resolve)

### Alert Configurations
- `GET /api/alert-configs` - List alert configurations
- `POST /api/alert-configs` - Create alert config
- `DELETE /api/alert-configs/:id` - Delete alert config

### Utilities
- `GET /api/stats` - System statistics
- `GET /api/violations` - Current geofence violations
- `GET /health` - Health check
- `WS /ws` - WebSocket for real-time events

## Technology Stack

### Frontend
- **Framework**: Next.js 16 (App Router)
- **Styling**: Tailwind CSS v4
- **Maps**: Leaflet + React Leaflet
- **Forms**: React Hook Form
- **Notifications**: Sonner
- **Icons**: Lucide React
- **Date**: date-fns

### Backend
- **Language**: Go 1.21+
- **HTTP**: Go standard library
- **WebSocket**: Gorilla WebSocket
- **UUID**: Google UUID

## Key Features Explained

### Real-time Map
- Interactive Leaflet map showing vehicle locations
- Color-coded geofence zones (delivery, restricted, toll, customer areas)
- Live vehicle position updates
- Draw mode for creating new geofences
- Pulsing alerts for restricted zone violations

### Alert System
- Create rules for specific geofences and vehicles
- Three severity levels: info, warning, critical
- Real-time WebSocket updates
- Toast notifications for new alerts
- Mark alerts as resolved

### Dashboard
- 4 stat cards showing key metrics
- Live map view with vehicle tracking
- Real-time alerts feed
- Recent violations table
- Auto-refresh every 30 seconds

### Vehicle Management
- Register new vehicles
- Track location and battery level
- View current geofence status
- Update vehicle status (active/idle)
- Real-time speed information

### Geofence Management
- Create multiple geofence types
- Draw polygon boundaries on map
- Categorize zones (delivery, restricted, toll, customer)
- Monitor violations in real-time
- Export/import geofences

## Styling System

The app uses a tactical dark theme with CSS variables:
- `--bg-primary`: Deep navy (#0a0f1e)
- `--bg-card`: Darker card backgrounds (#1a2235)
- `--cyan`: Primary accent color (#06b6d4)
- `--amber`: Warning color (#f59e0b)
- `--red`: Alert/error color (#ef4444)
- `--green`: Success color (#10b981)

Grid overlay provides tactical aesthetic. Animations include:
- `pulse-red`: Pulsing effect for violations
- `pulse-dot`: Connection status indicator
- `slide-in`: New alert notifications

## Deployment

### Frontend (Vercel)
1. Push to GitHub
2. Connect repository to Vercel
3. Set environment variables
4. Deploy

### Backend (Any server)
```bash
# Build Go binary
go build -o geofence-api main.go

# Run with environment variables
./geofence-api
```

Or use Docker:
```dockerfile
FROM golang:1.21
WORKDIR /app
COPY . .
RUN go build -o geofence-api main.go
CMD ["./geofence-api"]
```

## Development Tips

### Adding Mock Data
Edit `lib/api.ts` to add more mock geofences/vehicles/alerts for testing.

### Extending API
Add new endpoints in `backend/main.go` and corresponding functions in `lib/api.ts`.

### Custom Themes
Modify CSS variables in `app/globals.css` to change colors throughout the app.

### Testing Maps
The MapView component has a `drawMode` prop for testing geofence drawing functionality.

## Performance Considerations

- **Frontend**: Uses dynamic imports for Leaflet (heavy library)
- **Backend**: In-memory storage (replace with database for production)
- **Real-time**: WebSocket connection auto-reconnects
- **API**: Mock fallback prevents app breakage when backend is down

## Future Enhancements

- PostgreSQL database integration
- User authentication & authorization
- Database persistence for all data
- Advanced analytics & reporting
- Mobile app (React Native)
- SMS/Email alerts
- GPS accuracy analysis
- Route optimization
- Driver behavior monitoring

## License

MIT

## Support

For issues or questions, refer to:
- [Frontend docs](./app/README.md)
- [Backend docs](./backend/README.md)
- Issue tracker
