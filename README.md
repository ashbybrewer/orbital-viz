<p align="center">
  <img src="https://img.shields.io/badge/Three.js-r159-00e5ff?style=for-the-badge&logo=threedotjs&logoColor=white" />
  <img src="https://img.shields.io/badge/Node.js-18+-339933?style=for-the-badge&logo=node.js&logoColor=white" />
  <img src="https://img.shields.io/badge/WebGL-Powered-FF0000?style=for-the-badge&logo=opengl&logoColor=white" />
  <img src="https://img.shields.io/badge/Space--Track.org-TLE%20Data-003366?style=for-the-badge" />
  <img src="https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge" />
</p>

<h1 align="center">🛰 Orbital Viz</h1>
<p align="center"><strong>Production-grade real-time 3D satellite tracking & orbital debris visualization</strong></p>
<p align="center">
  Track hundreds of Earth-orbiting objects in a zero-install browser demo — and scale to 5,000+ live via Space-Track ingestion when you add credentials. Powered by SGP4 propagation, Space-Track.org TLE data,<br/>
  Three.js WebGL rendering, and WebSocket real-time position broadcasting.
</p>

---

## ✨ Features

| Category | Details |
|---|---|
| **3D Visualization** | WebGL point cloud rendering of hundreds of satellites in the demo at 60fps (the ingestion backend scales to 5,000+). Earth sphere with 8K day/night textures, specular, normal, and cloud layers. Bloom post-processing. |
| **Real-Time Tracking** | SGP4 orbital propagation running in a Web Worker at ~15fps. WebSocket pushes live position updates to all connected clients every 2 seconds. |
| **Live TLE Data** | Ingests from Space-Track.org every 15 minutes. Rate-limit-aware (28 req/hr hard cap). Session cookie caching for auth efficiency. |
| **Demo Mode** | Works out-of-the-box **without credentials** — hundreds of real and catalog-derived TLEs built in (ISS, HST, GPS, GOES, Starlink, Tiangong, Iridium debris, Cosmos debris, FengYun debris, and more). |
| **Object Selection** | GPU raycasting click-to-select. Displays lat/lon/alt, velocity, orbital period, apogee/perigee, eccentricity, inclination, epoch. Draws 2-hour orbit trail. |
| **Pass Prediction** | REST endpoint computes overhead passes from observer lat/lon/alt. |
| **Conjunction Screening** | Screens up to 2,000 objects for close approaches within configurable threshold (km). |
| **Ground Tracks** | Generates geodetic ground track paths for any object. |
| **Type Filtering** | Toggle payloads (cyan), rocket bodies (amber), debris (red), and unknowns (gray) independently. |
| **Search** | Debounced autocomplete search by name or NORAD ID. |
| **Time Control** | Log-scale time slider from 1× to 3,600× (1 hour per second). Real Earth sidereal rotation (86,164s period). |

---

## 🚀 Quick Start

### Prerequisites
- Node.js ≥ 18
- (Optional) [Space-Track.org](https://www.space-track.org/auth/createAccount) free account for live data

### 1. Clone & Install

```bash
git clone https://github.com/YOUR_USERNAME/orbital-viz.git
cd orbital-viz
npm install
```

### 2. Configure Environment

```bash
cp .env.example .env
```

Edit `.env`:
```env
# Optional — app runs in demo mode without these
SPACETRACK_USERNAME=your_email@example.com
SPACETRACK_PASSWORD=your_password

PORT=3000
MAX_OBJECTS=5000
INGEST_CRON=*/15 * * * *
LOG_LEVEL=info
```

### 3. Download Earth Textures (Optional but Recommended)

```bash
node scripts/download-textures.js
```

Downloads NASA Blue Marble textures (~80MB). Falls back to solid color sphere if skipped.

### 4. Launch

```bash
npm start
```

Open **http://localhost:3000**

> **No credentials?** No problem — the app launches in demo mode with a curated set of real satellites pre-loaded including ISS, Hubble, GPS, GOES, Starlink, Tiangong, and major debris clouds.

---

## 🏗 Architecture

```
orbital-viz/
├── src/
│   ├── server.js                    # Express + WebSocket server
│   ├── ingestion/
│   │   └── ingestor.js              # Space-Track.org TLE pipeline
│   ├── mechanics/
│   │   ├── orbital.js               # SGP4 propagation + coordinate math
│   │   └── orbital.test.js          # Jest unit tests
│   ├── visualization/
│   │   └── engine.js                # Three.js WebGL engine (ES module)
│   ├── workers/
│   │   └── propagation.worker.js    # Off-thread SGP4 Web Worker
│   └── utils/
│       ├── logger.js                # Winston structured logger
│       └── websocket-client.js      # Resilient WS client
├── public/
│   ├── index.html                   # Self-contained frontend (no bundler)
│   └── textures/                    # NASA earth textures (after download)
├── scripts/
│   └── download-textures.js         # NASA texture downloader
├── .env.example
└── package.json
```

### Data Flow

```
Space-Track.org
      │  TLE fetch (every 15 min)
      ▼
SpaceTrackIngestor ──► In-memory TLE store (Map<noradId, ParsedTLE>)
      │                         │
      │                         ▼
      │                  orbital.js (SGP4)
      │                  buildSatrecCache()
      │                         │
      ▼                         ▼
Express REST API         propagation.worker.js
GET /api/objects              │ Float32Array (stride 4)
GET /api/positions            │ via transferable postMessage
GET /api/groundtrack          │
POST /api/passes              ▼
POST /api/conjunctions   VisualizationEngine (Three.js)
      │                  - Point cloud shader
      │                  - Earth sphere
      │                  - Orbit trails
      ▼                  - Bloom pass
WebSocket /ws ──────────────────────────────────────────►  Browser
  POSITIONS broadcast every 2s                            (no bundler)
```

---

## 🌐 REST API Reference

### `GET /api/status`
Server health, ingestion statistics, WebSocket client count.

```json
{
  "status": "ok",
  "uptime": 3600.12,
  "ingestion": {
    "totalObjects": 4823,
    "payloads": 2100,
    "rocketBodies": 650,
    "debris": 1890,
    "unknown": 183,
    "demoMode": false
  },
  "wsClients": 4
}
```

### `GET /api/objects`
All tracked objects. Query params: `?type=payload|debris|rocket_body|unknown`, `?limit=N`, `?search=NAME_OR_NORAD_ID`

### `GET /api/objects/:noradId`
Single object with current propagated position, velocity, orbit classification.

### `GET /api/positions`
Current ECI scene positions of all objects. Optimized for visualization polling.

### `GET /api/groundtrack/:noradId`
Geodetic ground track. Query params: `?minutes=120&step=60`

### `POST /api/passes`
Overhead pass prediction.
```json
{
  "noradId": 25544,
  "lat": 36.17,
  "lon": -86.78,
  "alt": 0,
  "hours": 24,
  "minElevation": 10
}
```

### `POST /api/conjunctions`
Screen objects for close approaches.
```json
{
  "noradIds": [25544, 20580],
  "thresholdKm": 50
}
```

### `GET /api/tle/:noradId`
Raw TLE strings for a NORAD ID.

### `POST /api/ingest/trigger`
Manually trigger ingestion cycle.

---

## 🔌 WebSocket Protocol

Connect to `ws://localhost:3000/ws`

### Server → Client Messages

| Type | Description |
|---|---|
| `INIT` | Initial state snapshot on connect |
| `POSITIONS` | `[[noradId, x, y, z], ...]` every 2 seconds |
| `DATA_REFRESH` | Notifies clients of completed ingestion cycle |
| `PONG` | Reply to client PING |

### Client → Server Messages

| Type | Fields | Description |
|---|---|---|
| `SUBSCRIBE_OBJECT` | `noradId` | Request per-object position updates |
| `UNSUBSCRIBE` | — | Cancel subscription |
| `PING` | — | Keepalive |

---

## 🧮 Orbital Mechanics Module

`src/mechanics/orbital.js` is a pure, side-effect-free math module wrapping `satellite.js` (SGP4/SDP4):

```js
const orbital = require('./src/mechanics/orbital');

// Initialize satellite record from TLE
const satrec = orbital.initSatelliteRecord(line1, line2);

// Propagate to a specific time
const { position, velocity } = orbital.propagateToDate(satrec, new Date());
// position → ECI km {x, y, z}

// Convert to geodetic
const { latDeg, lonDeg, altKm } = orbital.propagateToGeodetic(satrec, new Date());

// Convert ECI to Three.js scene units (1 unit = 6371 km)
const scenePos = orbital.eciToScene(position);

// Classify orbit
orbital.classifyOrbit(408);  // → 'LEO'
orbital.classifyOrbit(20200); // → 'MEO'

// Orbital period
orbital.orbitalPeriodMinutes(15.5);  // → ~92.9 min

// Ground track (next 2 hours, 60s steps)
const track = orbital.generateGroundTrack(satrec, new Date(), 120, 60);

// Pass prediction
const passes = orbital.findPasses(satrec, 36.17, -86.78, 0, new Date(), 24, 10);

// Conjunction screening
const events = orbital.screenConjunctions(objects, new Date(), 50);

// Batch propagate → Float32Array for GPU transfer (stride 4: x,y,z,typeCode)
const buf = orbital.batchPropagateToFloat32(objects, new Date(), satrecCache);
```

---

## ⚙️ Configuration Reference

| Variable | Default | Description |
|---|---|---|
| `SPACETRACK_USERNAME` | — | Space-Track.org login email |
| `SPACETRACK_PASSWORD` | — | Space-Track.org password |
| `PORT` | `3000` | HTTP/WS server port |
| `MAX_OBJECTS` | `5000` | Maximum objects to ingest |
| `CACHE_TTL` | `900` | API response cache TTL (seconds) |
| `INGEST_CRON` | `*/15 * * * *` | TLE refresh schedule (cron syntax) |
| `LOG_LEVEL` | `info` | Winston log level |
| `NODE_ENV` | `development` | Set to `production` for CORS lockdown |
| `ALLOWED_ORIGIN` | `*` | CORS origin in production |

---

## 🛸 Demo Mode Satellite Roster

When running without Space-Track credentials, a curated set of real objects is loaded, including:

| Object | NORAD ID | Type | Orbit |
|---|---|---|---|
| ISS (ZARYA) | 25544 | Payload | LEO 408km |
| Hubble Space Telescope | 20580 | Payload | LEO 547km |
| NOAA 18 | 28654 | Payload | LEO SSO |
| Tiangong CSS (TIANHE) | 48274 | Payload | LEO 390km |
| GOES 16 | 41866 | Payload | GEO |
| GOES 18 | 51850 | Payload | GEO |
| GPS BIIR-2, -3, -4 | various | Payload | MEO 20,200km |
| Starlink-1007 through -2033 | various | Payload | LEO 550km |
| Terra, Aqua | 25994, 27424 | Payload | LEO SSO |
| Sentinel-2A, -2B | 40697, 42063 | Payload | LEO SSO |
| Landsat 8 | 39084 | Payload | LEO SSO |
| Vanguard 1 (oldest satellite, 1958) | 5 | Payload | LEO elliptical |
| Iridium 33 Debris (2009 collision) | 33442, 33591 | Debris | LEO |
| Cosmos 2251 Debris (2009 collision) | 33818+ | Debris | LEO |
| FengYun 1C Debris (2007 ASAT test) | 29228+ | Debris | LEO SSO |
| Cosmos 1408 Debris (2021 ASAT test) | 49271–49274 | Debris | LEO |
| SL-16, SL-8, SL-4 Rocket Bodies | various | Rocket Body | LEO/MEO |
| Molniya 1-69 R/B | 8195 | Rocket Body | HEO |

---

## 🧪 Testing

```bash
npm test
```

Jest suite covers: SGP4 initialization, propagation, coordinate transforms, orbit classification, ground track generation, pass prediction, conjunction screening, batch propagation.

---

## 🖥 Frontend Controls

| Control | Action |
|---|---|
| **Left drag** | Rotate globe |
| **Scroll / pinch** | Zoom |
| **Right drag** | Pan |
| **Click object** | Select — shows orbital data panel |
| **Search bar** | Filter by name or NORAD ID |
| **Type toggles** | Show/hide payloads, rocket bodies, debris, unknowns |
| **Time scale slider** | 1× to 3,600× speed |
| **Auto-rotate** | Toggle Earth rotation |
| **Bloom** | Toggle post-processing bloom effect |
| **Reset camera** | Return to default view |

---

## 🛠 Development

```bash
npm run dev        # nodemon auto-restart
npm test           # Jest test suite
npm run lint       # ESLint
```

---

## 📡 Data Sources

- **TLE Data**: [Space-Track.org](https://www.space-track.org) — official US Space Force catalog (requires free account for live data)
- **SGP4 Propagator**: [satellite.js](https://github.com/shashwatak/satellite-js) — JavaScript port of the AFSPC SGP4/SDP4 implementation
- **Earth Textures**: [NASA Blue Marble](https://visibleearth.nasa.gov/collection/1484/blue-marble) — 8K day, night, cloud, specular, and normal maps

---

## 📄 License

MIT © 2024

---

<p align="center">
  Built with Three.js · SGP4 · Space-Track.org · Express · WebSocket<br/>
  <em>Tracking humanity's debris field since the first Sputnik.</em>
</p>
