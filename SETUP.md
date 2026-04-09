# Setup Guide - Geofence OS

## Option 1: Quick Start (Recommended for Development)

### Prerequisites
- Node.js 18+ (for frontend)
- Go 1.21+ (for backend)
- pnpm (or npm/yarn)

### Step 1: Start the Backend

```bash
cd backend
go mod download
go run main.go
```

You should see:
```
Server starting on :8080
```

### Step 2: Start the Frontend (in another terminal)

```bash
# From project root
pnpm install
pnpm dev
```

You should see:
```
✓ Ready in 418ms
- Local: http://localhost:3000
```

### Step 3: Open the Dashboard

Navigate to `http://localhost:3000` in your browser.

---

## Option 2: Using Docker (Recommended for Production)

### Prerequisites
- Docker
- Docker Compose

### One-Command Startup

```bash
docker-compose up --build
```

This will:
1. Build the Go backend
2. Build the Next.js frontend
3. Start both services
4. Frontend available at: `http://localhost:3000`
5. Backend available at: `http://localhost:8080`

### Stopping Services

```bash
docker-compose down
```

---

## Option 3: Frontend Only (Without Backend)

If you only want to develop the frontend, it works perfectly with mock data:

```bash
pnpm install
pnpm dev
```

The app will use hardcoded mock data for all operations.

---

## Verify Installation

### Test Backend Health

```bash
curl http://localhost:8080/health
```

Should return:
```json
{"status": "ok"}
```

### Test Frontend Loading

Open `http://localhost:3000` and verify the dashboard loads with:
- 4 stat cards showing stats
- Interactive map with geofences and vehicles
- Live alerts feed
- Recent violations table

---

## Configuration

### Environment Variables

Create `.env.local` in the project root:

```env
# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:8080
NEXT_PUBLIC_WS_URL=ws://localhost:8080

# For Docker, use service names:
# NEXT_PUBLIC_API_URL=http://backend:8080
# NEXT_PUBLIC_WS_URL=ws://backend:8080
```

### Backend Configuration

The backend uses environment variables. Edit `backend/main.go` to customize:
- Port (currently hardcoded to `:8080`)
- CORS settings
- Data persistence layer

---

## Development Workflow

### Adding a New Feature

1. **Create types** in `lib/types.ts`
2. **Add API endpoints** in `backend/main.go`
3. **Add API functions** in `lib/api.ts`
4. **Build UI components** in `components/`
5. **Create pages** in `app/`

### Running Tests

Frontend:
```bash
pnpm test
```

Backend:
```bash
cd backend
go test ./...
```

### Build for Production

Frontend:
```bash
pnpm build
pnpm start
```

Backend:
```bash
cd backend
go build -o geofence-api main.go
./geofence-api
```

---

## Troubleshooting

### Port Already in Use

**Frontend (3000)**
```bash
# Find process using port 3000
lsof -i :3000

# Kill the process (replace PID)
kill -9 <PID>
```

**Backend (8080)**
```bash
# Find process using port 8080
lsof -i :8080

# Kill the process
kill -9 <PID>
```

### Module Not Found Errors

**Frontend:**
```bash
pnpm install
# or clear node_modules
rm -rf node_modules pnpm-lock.yaml
pnpm install
```

**Backend:**
```bash
cd backend
go mod tidy
go mod download
```

### CORS Errors

The backend is configured to accept requests from any origin. If you still see CORS errors:

1. Check that `NEXT_PUBLIC_API_URL` points to correct backend
2. Verify backend is running on port 8080
3. Check browser console for actual error message

### WebSocket Connection Failed

The app will work without WebSocket (using polling instead). If you want real-time updates:

1. Ensure backend is running
2. Check `NEXT_PUBLIC_WS_URL` is correct
3. Verify WebSocket `/ws` endpoint is accessible

---

## Next Steps

- Read the main [README.md](./README.md) for feature overview
- Check [backend/README.md](./backend/README.md) for API documentation
- Review component files in `components/` for UI patterns
- Explore pages in `app/` to see full feature implementation

---

## Support

For detailed information:
- Next.js docs: https://nextjs.org/docs
- Go docs: https://golang.org/doc
- Leaflet docs: https://leafletjs.com/
- React Hook Form: https://react-hook-form.com/
