# Geospatial Frontend

> Covers: F04 MapLibre GL Integration · F05 Deck.gl Layer System · F06 Strategic Geo Layers · F07 Regional Preset System · F13 Map Event Markers · F30 Instability Heatmap Layer

---

## F04 — MapLibre GL Integration

### What Is Being Built

The core map component using MapLibre GL JS — an open-source, WebGL-powered map renderer that provides BIE's primary interface for visualizing global events spatially. The map loads a dark-themed vector tile basemap, supports smooth zooming from globe-level overviews to street-level detail, and serves as the canvas for all data layers.

### Why It Is Needed

BIE is fundamentally a *spatial* intelligence tool. Every event, vessel, military movement, and infrastructure location has geographic coordinates. Without a performant map, BIE is just another news feed. The map provides:
- **Immediate situational awareness** — "Where in the world should I pay attention right now?"
- **Pattern recognition** — clusters of events near chokepoints, borders, or strategic locations become visible
- **Context** — a maritime incident near Malacca Strait means something different than one near Norway

### What Use It Provides

- **Globe → street zoom** — seamless zoom from global overview to ground-level detail
- **WebGL performance** — handles thousands of markers and data overlays without lag
- **Dark theme** — Intel-aesthetic dark basemap with muted labels, matching BIE's design language
- **Vector tiles** — smooth zoom, custom styling, detailed at every level
- **3D perspective** — tiltable/rotatable for dramatic views of terrain (LAC mountains, IOR expanse)
- **Multiple basemap styles** — toggle between dark, satellite, and terrain views

### How It Is Built

1. **Install**: `npm install maplibre-gl`

2. **Map component** — `components/Map/MapView.js`:
   ```jsx
   import maplibregl from "maplibre-gl";
   import "maplibre-gl/dist/maplibre-gl.css";

   function MapView({ onMapReady }) {
     const mapContainer = useRef(null);
     const map = useRef(null);

     useEffect(() => {
       map.current = new maplibregl.Map({
         container: mapContainer.current,
         style: "https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json",
         center: [78.9, 22.5],  // India as initial focus
         zoom: 3.5,             // Globe-level view showing broader region
         pitch: 15,             // Slight tilt for depth
         bearing: 0,
         maxZoom: 18,
         minZoom: 1.5,          // Allow global zoom-out
         attributionControl: false,
       });

       map.current.addControl(new maplibregl.NavigationControl(), "top-right");
       map.current.on("load", () => onMapReady(map.current));

       return () => map.current?.remove();
     }, []);

     return <div ref={mapContainer} className="map-container" />;
   }
   ```

3. **Styling**: dark basemap (`dark-matter`) with muted labels. India and surrounding regions visible at default zoom, but user can zoom out to see the full globe or zoom into any region.

4. **Performance**: vector tiles stream progressively. Labels and minor features are only rendered at appropriate zoom levels.

---

## F05 — Deck.gl Layer System

### What Is Being Built

An overlay layer system using Deck.gl that renders all dynamic data on top of the MapLibre basemap. Deck.gl provides GPU-accelerated visualization layers for event markers, the instability heatmap, infrastructure icons, and knowledge graph connection arcs — across the entire globe.

### Why It Is Needed

MapLibre handles the basemap; Deck.gl handles the data. This separation is critical because:
- **GPU acceleration** — Deck.gl renders 10,000+ data points at 60fps via WebGL
- **Layer compositing** — multiple data layers (events, heatmap, vessels, arcs) can be toggled independently
- **Performance scaling** — layers can be disabled per device capability and zoom level

### What Use It Provides

- **ScatterplotLayer** — event markers, colored by threat level, sized by significance
- **HeatmapLayer** — continuous instability heatmap overlay across global regions
- **IconLayer** — infrastructure markers, vessel positions, military bases
- **ArcLayer** — knowledge graph relationships visualized as curved connections between locations
- **Layer toggling** — users enable/disable individual layers via a layer control panel
- **Zoom-adaptive rendering** — detail layers only visible at appropriate zoom levels (progressive disclosure)

### How It Is Built

1. **Install**: `npm install @deck.gl/core @deck.gl/layers @deck.gl/mapbox`

2. **Layer manager** — `components/Map/LayerManager.js`:
   ```javascript
   import { ScatterplotLayer, IconLayer, ArcLayer, HeatmapLayer } from "@deck.gl/layers";

   export function buildLayers({ events, heatmapData, infrastructure, graphArcs, zoom }) {
     const layers = [];

     // Event markers — always visible
     layers.push(new ScatterplotLayer({
       id: "events",
       data: events,
       getPosition: (d) => [d.location.lng, d.location.lat],
       getFillColor: (d) => THREAT_COLORS[d.threat_level],
       getRadius: (d) => d.threat_level === "CRITICAL" ? 12000 : 8000,
       pickable: true,
       radiusScale: 1,
       radiusMinPixels: 4,
       radiusMaxPixels: 20,
     }));

     // Heatmap — visible at zoom < 6 (overview mode)
     if (zoom < 6) {
       layers.push(new HeatmapLayer({
         id: "instability-heatmap",
         data: heatmapData,
         getPosition: (d) => [d.lng, d.lat],
         getWeight: (d) => d.score,
         radiusPixels: 60,
         intensity: 1.5,
         colorRange: [
           [0, 255, 0, 50],     // green — low
           [255, 255, 0, 100],   // yellow
           [255, 165, 0, 150],   // orange
           [255, 0, 0, 200],     // red — high
         ],
       }));
     }

     // Infrastructure — visible at zoom > 3
     if (zoom > 3) {
       layers.push(new IconLayer({
         id: "infrastructure",
         data: infrastructure,
         getPosition: (d) => [d.lng, d.lat],
         getIcon: (d) => infraIcon(d.type),
         getSize: 28,
         pickable: true,
       }));
     }

     return layers;
   }
   ```

3. **Threat color mapping**:
   ```javascript
   const THREAT_COLORS = {
     LOW: [46, 204, 113],        // green
     MEDIUM: [241, 196, 15],     // yellow
     HIGH: [231, 76, 60],        // red
     CRITICAL: [192, 57, 43],    // dark red, pulsing
   };
   ```

4. **Performance**: cap at 500 visible markers. Use viewport culling. Disable HeatmapLayer on devices with `navigator.hardwareConcurrency < 4`.

---

## F06 — Strategic Geo Layers

### What Is Being Built

Static GeoJSON layers that overlay strategically important boundaries, routes, and zones on the map. These provide essential geographic context for intelligence events — showing contested borders, maritime chokepoints, strategic corridors, and alliance boundaries.

### Why It Is Needed

Events near the LAC, in the Strait of Malacca, or along BRI routes carry specific strategic significance that generic map labels don't convey. These layers provide the "so what?" context:
- "This military exercise is happening 50km from the LAC" — visible only if the LAC is drawn
- "This port investment is part of China's String of Pearls" — visible only if those ports are marked
- "This chokepoint disruption affects India's oil supply" — visible only if chokepoints are displayed

### What Use It Provides

- **India-specific borders**: LAC (Line of Actual Control), LOC (Line of Control), India-claimed territory boundaries
- **Maritime chokepoints**: Strait of Malacca, Strait of Hormuz, Bab el-Mandeb, Suez Canal, Lombok Strait, Taiwan Strait
- **BRI corridors**: China-Pakistan Economic Corridor (CPEC), Maritime Silk Road, key BRI port investments
- **Military presence**: India's overseas military facilities (Agaléga, Assumption Island, Farkhor), Chinese bases (Djibouti, Gwadar, Hambantota, Ream)
- **Alliance boundaries**: Quad nations, SCO members, BRICS nations
- **All layers toggleable** via the layer control panel

### How It Is Built

1. **Directory structure**:
   ```
   frontend/public/geo/
   ├── india_borders/
   │   ├── lac.geojson          — Line of Actual Control
   │   ├── loc.geojson          — Line of Control
   │   └── state_borders.geojson
   ├── chokepoints/
   │   ├── malacca.geojson      — Strait of Malacca zone
   │   ├── hormuz.geojson       — Strait of Hormuz zone
   │   ├── bab_el_mandeb.geojson
   │   ├── suez.geojson
   │   ├── lombok.geojson
   │   └── taiwan_strait.geojson
   ├── bri_corridors/
   │   ├── cpec.geojson         — China-Pakistan Economic Corridor
   │   └── maritime_silk_road.geojson
   ├── military_bases/
   │   ├── india_overseas.geojson
   │   └── china_overseas.geojson
   └── alliances/
       ├── quad_nations.geojson
       └── sco_members.geojson
   ```

2. **Layer loading** — `components/Map/GeoLayers.js`:
   ```javascript
   const GEO_LAYERS = {
     lac: { url: "/geo/india_borders/lac.geojson", color: "#ff4444", dashArray: [4, 2], label: "LAC" },
     loc: { url: "/geo/india_borders/loc.geojson", color: "#ff8800", dashArray: [4, 2], label: "LOC" },
     chokepoints: { url: "/geo/chokepoints/all.geojson", color: "#00d4ff", fill: true, label: "Chokepoints" },
     bri: { url: "/geo/bri_corridors/cpec.geojson", color: "#ff0066", width: 3, label: "BRI/CPEC" },
     india_bases: { url: "/geo/military_bases/india_overseas.geojson", color: "#00ff88", label: "India Bases" },
     china_bases: { url: "/geo/military_bases/china_overseas.geojson", color: "#ff3333", label: "China Bases" },
   };

   async function loadGeoLayer(map, layerId) {
     const config = GEO_LAYERS[layerId];
     const resp = await fetch(config.url);
     const geojson = await resp.json();
     map.addSource(layerId, { type: "geojson", data: geojson });
     map.addLayer({
       id: layerId,
       type: config.fill ? "fill" : "line",
       source: layerId,
       paint: config.fill
         ? { "fill-color": config.color, "fill-opacity": 0.15 }
         : { "line-color": config.color, "line-width": 2, "line-dasharray": config.dashArray || [1] },
     });
   }
   ```

3. **Default visibility**: LAC and LOC always visible when zoomed into South Asia. Chokepoints visible at zoom < 6. BRI and bases toggled by user.

---

## F07 — Regional Preset System

### What Is Being Built

A set of one-click navigation buttons that fly the map camera to strategically important regions with pre-configured zoom, pitch, and bearing. Each preset activates relevant geo layers and filters the news feed to show events from that region.

### Why It Is Needed

Analysts shouldn't have to manually pan and zoom to find the South China Sea or the Strait of Hormuz. Presets provide instant navigation to India's key strategic theaters. During the demo, presets enable smooth transitions: *"Let me show you the LAC situation... now zoom out to the IOR... and here's the South China Sea."*

### What Use It Provides

- **One-click navigation** to 12+ strategic regions relevant to Indian interests
- **Animated camera transitions** — smooth fly-to with configurable duration
- **Auto-layer activation** — LAC preset auto-enables LAC border layer; IOR preset enables chokepoints
- **Auto-filter** — activating a preset filters the news feed to that region's events
- **URL encoding** — active preset encoded in URL hash for sharing: `?preset=south_china_sea`

### How It Is Built

1. **Preset definitions** — `config/presets.js`:
   ```javascript
   export const REGIONAL_PRESETS = {
     // ----- India & Immediate Neighborhood -----
     aksai_chin: {
       label: "Aksai Chin / LAC",
       center: [79.0, 35.0], zoom: 7, pitch: 45, bearing: -15,
       layers: ["lac"], filter: { geo_focus: "SOUTH_ASIA" },
       description: "Line of Actual Control — India-China disputed border"
     },
     kashmir: {
       label: "Kashmir / LOC",
       center: [74.8, 34.0], zoom: 7.5, pitch: 40, bearing: 0,
       layers: ["loc"], filter: { geo_focus: "SOUTH_ASIA" },
       description: "Line of Control — India-Pakistan disputed border"
     },
     siachen: {
       label: "Siachen Glacier",
       center: [77.1, 35.5], zoom: 9, pitch: 60, bearing: 30,
       layers: ["lac", "loc"], filter: { geo_focus: "SOUTH_ASIA" },
       description: "World's highest battleground"
     },
     northeast: {
       label: "Northeast India",
       center: [93.0, 26.0], zoom: 6.5, pitch: 30, bearing: 0,
       layers: ["lac"], filter: { geo_focus: "SOUTH_ASIA" },
       description: "India's strategic corridor to Southeast Asia"
     },

     // ----- Indian Ocean Region -----
     indian_ocean: {
       label: "Indian Ocean Region",
       center: [72.0, 0.0], zoom: 3, pitch: 10, bearing: 0,
       layers: ["chokepoints", "india_bases", "china_bases"],
       filter: { geo_focus: "IOR", category: "MARITIME" },
       description: "India's primary maritime domain"
     },

     // ----- Indo-Pacific -----
     south_china_sea: {
       label: "South China Sea",
       center: [115.0, 12.0], zoom: 5, pitch: 20, bearing: 0,
       layers: ["chokepoints", "china_bases"],
       filter: { geo_focus: "INDO_PACIFIC" },
       description: "Most contested waterway — 9-dash line, island militarization"
     },
     taiwan_strait: {
       label: "Taiwan Strait",
       center: [119.5, 24.0], zoom: 6.5, pitch: 30, bearing: 0,
       layers: ["chokepoints", "china_bases"],
       filter: { geo_focus: "INDO_PACIFIC" },
       description: "Potential flashpoint — PLA military pressure on Taiwan"
     },
     malacca_strait: {
       label: "Strait of Malacca",
       center: [101.0, 2.5], zoom: 6, pitch: 15, bearing: 0,
       layers: ["chokepoints"],
       filter: { geo_focus: "INDO_PACIFIC", category: "MARITIME" },
       description: "80% of India's oil imports transit here"
     },

     // ----- Middle East -----
     persian_gulf: {
       label: "Persian Gulf / Hormuz",
       center: [54.0, 26.0], zoom: 5.5, pitch: 20, bearing: 0,
       layers: ["chokepoints"],
       filter: { geo_focus: "MIDDLE_EAST" },
       description: "Strait of Hormuz — 20% of global oil transit"
     },
     red_sea: {
       label: "Red Sea / Bab el-Mandeb",
       center: [42.0, 14.0], zoom: 5, pitch: 15, bearing: 0,
       layers: ["chokepoints"],
       filter: { geo_focus: "MIDDLE_EAST", category: "MARITIME" },
       description: "Houthi threat zone — Suez Canal access"
     },

     // ----- Africa -----
     horn_of_africa: {
       label: "Horn of Africa",
       center: [45.0, 8.0], zoom: 5, pitch: 15, bearing: 0,
       layers: ["china_bases"],
       filter: { geo_focus: "AFRICA" },
       description: "China's Djibouti base, piracy corridor, Ethiopia/Somalia dynamics"
     },

     // ----- Central Asia -----
     central_asia: {
       label: "Central Asia / Afghanistan",
       center: [67.0, 36.0], zoom: 5, pitch: 20, bearing: 0,
       layers: [],
       filter: { geo_focus: "CENTRAL_ASIA" },
       description: "Afghanistan instability, SCO dynamics, India's connectivity aspirations"
     },
   };
   ```

2. **Preset navigation**:
   ```javascript
   function flyToPreset(map, presetId) {
     const preset = REGIONAL_PRESETS[presetId];
     map.flyTo({
       center: preset.center,
       zoom: preset.zoom,
       pitch: preset.pitch,
       bearing: preset.bearing,
       duration: 2000,
       essential: true,
     });
     // Activate relevant layers, apply feed filters
   }
   ```

3. **UI**: horizontal scrollable preset bar at the top of the map. Each button shows the region name with a subtle flag/icon.

---

## F13 — Map Event Markers

### What Is Being Built

Interactive markers on the map representing intelligence events, rendered via Deck.gl's ScatterplotLayer. Each marker is color-coded by threat level (green/yellow/red/pulsing red), shows a category icon, and opens a detail panel on click.

### Why It Is Needed

The map is how analysts spot geographic patterns. Clusters of red markers near the LAC or a sudden cluster in the South China Sea are immediately visible patterns that a text feed cannot convey.

### What Use It Provides

- **Threat-level coloring** — instant visual assessment of severity
- **Category icons** — distinguish military from diplomatic from maritime at a glance
- **Click-to-detail** — opens the EventDetailPanel with full NLP analysis, entities, infrastructure proximity
- **Click-to-fly** — clicking a marker from the news feed flies the map to that location
- **Viewport filtering** — only events within the visible map area are rendered
- **Smart clustering** — at low zoom levels, nearby markers cluster with count badges

### How It Is Built

1. **Event marker layer** (Deck.gl ScatterplotLayer):
   ```javascript
   new ScatterplotLayer({
     id: "event-markers",
     data: viewportEvents,
     getPosition: (d) => [d.location.lng, d.location.lat],
     getFillColor: (d) => THREAT_COLORS[d.threat_level],
     getRadius: (d) => THREAT_RADII[d.threat_level],
     pickable: true,
     radiusMinPixels: 4,
     radiusMaxPixels: 20,
     onClick: ({ object }) => {
       onEventSelect(object);
       map.flyTo({ center: [object.location.lng, object.location.lat], zoom: 8 });
     },
   });
   ```

2. **Performance caps**: maximum 500 markers visible. At zoom < 4, cluster into region-level aggregates. Prioritize Tier 1 and HIGH/CRITICAL events when cap is reached.

3. **Pulsing animation**: CRITICAL events get a CSS pulse animation overlay via a separate HTML marker layer.

---

## F30 — Instability Heatmap Layer

### What Is Being Built

A continuous color-coded heatmap overlay that visualizes the BIE Instability Index (BII) scores across regions of Indian strategic interest globally. High-instability regions glow red; stable regions are green or transparent.

### Why It Is Needed

Individual event markers show *what* is happening; the heatmap shows *how worried you should be* about a region overall. It transforms dozens of discrete events into a continuous threat assessment overlay — enabling instant visual comparison: "Is the LAC hotter than the South China Sea right now?"

### What Use It Provides

- **Global threat assessment** — one glance shows which theaters are active
- **Continuous overlay** — smooth color blending between data points
- **Dynamic updates** — recalculates as new events are ingested
- **Complements markers** — markers show events, heatmap shows aggregate threat
- **India-priority weighting** — regions closer to India or of higher strategic relevance show with greater intensity

### How It Is Built

1. **Data source**: BII scores from `GET /api/instability` — returns per-region scores.

2. **Heatmap layer** (Deck.gl HeatmapLayer):
   ```javascript
   new HeatmapLayer({
     id: "instability-heatmap",
     data: heatmapData,  // Array of {lat, lng, score} from BII
     getPosition: (d) => [d.lng, d.lat],
     getWeight: (d) => d.score,
     radiusPixels: 60,
     intensity: 1.5,
     threshold: 0.1,
     colorRange: [
       [0, 255, 0, 50],       // green — low
       [255, 255, 0, 100],     // yellow — moderate
       [255, 165, 0, 150],     // orange — elevated
       [255, 0, 0, 200],       // red — high
     ],
   });
   ```

3. **Visibility strategy**: heatmap is most useful at low zoom (global/regional overview). At high zoom levels (zoom > 7), individual event markers provide better detail — so the heatmap fades out via `opacity: Math.max(0, 1 - (zoom - 5) * 0.3)`.

4. **Performance**: disable on low-GPU devices. Limit data points to top 100 highest-scoring cells.

---

← Back to [prd.md](./prd.md)
