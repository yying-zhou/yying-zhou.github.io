# 3DGS-PPGIS Value Annotation Instrument

A single-file browser demo for annotating place-based values directly onto a 3D Gaussian Splatting scene. Built as a prototype research instrument for comparing 2D map-based vs. 3D immersive PPGIS methods in urban green space planning.

Part of Yingying Zhou's PhD research plan — *Making values visible: design-oriented spatial representation for multispecies-just urban planning* — targeting Prof. Nora Fagerholm's Sustainable Landscape Systems Research Group, University of Turku.

---

## What this file is

`index.html` is a **self-contained, zero-dependency demo** that runs in any browser with no build step, no server, and no installation. It simulates the core interaction of the real instrument: opening a 3D scene, tapping to place an annotation pin, selecting a value type, adding a note, and logging the result with scene coordinates and dwell time.

The scene in the demo is procedurally generated (a simulated photogrammetric splat). The section [Replacing the demo scene with your own Luma AI files](#replacing-the-demo-scene-with-your-own-luma-ai-files) explains how to swap it for a real `.splat` + `.glb` from your own photos.

---

## How it works, step by step

### Step 1 — Page loads, condition is assigned

When a participant opens the URL, the app assigns them to either the **3D condition** (this file) or the **2D condition** (a Leaflet/orthophoto version). In the full instrument this is done by a URL parameter or device-ID hash so the split is random and reproducible. The condition label is shown in the header badge.

The `sceneLoadTime` timestamp is recorded at this point. Every pin will store `dwell_ms = Date.now() - sceneLoadTime` — the time from opening the scene to placing that annotation. This is one of the key behavioural metrics in the 2D vs. 3D comparison study.

### Step 2 — The 3DGS scene renders

In the demo, the scene is drawn onto an HTML `<canvas>` element using the Canvas 2D API to simulate what a real Spark.js / THREE.js 3DGS scene looks like: splat particles composited over a park background, procedural trees, a gravel path.

In the real implementation, this canvas is replaced by a **WebGL2 canvas managed by Spark.js** (World Labs' THREE.js-based 3DGS renderer). The `.splat` file from Luma AI is streamed and rendered as a cloud of Gaussian ellipsoids. The user can navigate using mouse drag (desktop) or gyroscope/touch (mobile).

### Step 3 — User taps the scene to open the annotation sheet

Clicking anywhere on the canvas fires a `click` event handler. In the demo this records the canvas pixel coordinates `(cx, cy)` and converts them to fake scene coordinates `(scene_x, scene_y, scene_z)`.

In the real implementation, this step uses **THREE.js raycasting**:

```js
raycaster.setFromCamera(pointer, camera);
const hits = raycaster.intersectObject(proxyMesh, true);
if (hits.length) {
  pendingPos = hits[0].point; // THREE.Vector3 — real 3D position on the proxy mesh
}
```

The `proxyMesh` is the `.glb` file exported by Luma AI alongside the `.splat`. It is a low-polygon mesh that is geometrically aligned with the splat cloud but kept invisible (`proxyMesh.visible = false`). Its only job is to give the raycaster a surface to intersect — the splat itself has no geometry the raycaster can hit.

Once a hit point is found, the bottom sheet slides up.

### Step 4 — Participant annotates the location

The bottom sheet presents four value type buttons:

| Value type | Meaning |
|---|---|
| **Recreational** | Active use — sports, walking, play |
| **Aesthetic** | Visual, sensory, or spiritual appreciation |
| **Wildlife habitat** | Observed or inferred non-human species presence |
| **Avoidance** | A place the participant avoids and why |

These four categories align with the PPGIS value typology used in Fagerholm et al. (2021) and the multispecies justice dimensions in Raymond et al. (2025). The participant selects one, optionally adds a free-text note, and taps **Save annotation**.

### Step 5 — Pin is saved and rendered

`savePin()` creates a pin record and adds it to the in-memory `pins` array:

```js
{
  cx, cy,              // canvas pixel position (for re-drawing)
  valueType,           // selected category
  note,                // free text
  scene_x, scene_y, scene_z,  // 3D coordinates from raycast hit
  dwell_ms,            // time from page load to this pin
  ts                   // local time string
}
```

A colour-coded circle + pointer marker is drawn at the hit location and persists across subsequent interactions. The pin appears in the sidebar list with its coordinates and dwell time.

In the real implementation, the pin is also **written to Supabase** with a PostGIS geometry column:

```js
await supabase.from('pins').insert({
  user_id:    deviceHash(),
  condition:  '3D',
  value_type: selectedVal,
  note:       noteText,
  scene_x: pos.x, scene_y: pos.y, scene_z: pos.z,
  geom:    `POINT(${lon} ${lat})`,   // if georeferenced via GCPs
  dwell_ms: Date.now() - sceneLoadTime,
  ts:       new Date().toISOString()
});
```

### Step 6 — Overlay toggles (ecological proxy layers)

Two buttons in the top-right corner of the scene toggle overlay layers:

- **Bio overlay** — highlights zones of high biodiversity score (from LAJI.fi species occurrence data) in green. In the demo these are hardcoded radial zones; in the real instrument, per-splat scores are passed as a `Float32Array` shader uniform to Spark.js and applied in a GLSL fragment shader, colour-mixing each splat's base RGB toward green proportional to its score.
- **Canopy layer** — simulates the city LiDAR canopy cover layer, darkening the mid-scene vegetation zone.

These overlays demonstrate the core design concept: ecological proxy data is not placed as a 2D layer *on top of* the scene but rendered *into* the splat material itself, making non-human values spatially and perceptually present within the photorealistic environment.

### Step 7 — Data is available for analysis

The sidebar's **pins** tab lists every annotation with its value type, note, coordinates, and dwell time. The stats bar at the bottom shows total pin count, wildlife pin count, and average dwell time — the three primary metrics for the 2D vs. 3D comparison study.

In the full instrument, a separate researcher dashboard (built in D3.js) shows a live map of pin distributions per condition, allowing real-time monitoring of the A/B study.

---

## Replacing the demo scene with your own Luma AI files

This is the step that turns the prototype into a real research instrument. You need two files exported from Luma AI from the same capture session:

| File | What it is | How it is used |
|---|---|---|
| `scene.splat` | The 3DGS point cloud | Rendered visually by Spark.js |
| `scene.glb` | A low-poly proxy mesh | Used for raycasting only — kept invisible |

### Capture your park photos

- Walk through the space shooting **150–300 overlapping photos** with your phone. Cover all angles: ground level, eye level, looking up at canopies.
- More overlap = better reconstruction. Aim for >60% overlap between consecutive frames.
- Avoid fast movement and motion blur — the Gaussian splatting algorithm needs sharp, consistent keypoints.
- For Kupittaa Park or similar Finnish urban parks, a 20-minute walk with continuous shooting is sufficient for a prototype scene.

### Process with Luma AI

1. Create a free account at [lumalabs.ai](https://lumalabs.ai)
2. Upload your photos to a new scene — processing takes 20–30 minutes
3. When complete, export:
   - **Gaussian Splat (.splat)** — the main scene file
   - **Mesh (.glb)** — the proxy mesh for raycasting

Place both files in the same folder as `index.html`.

### Swap in Spark.js for the canvas

Replace the `<canvas id="sceneCanvas">` drawing code with a Spark.js initialisation. The minimal setup is:

```html
<script type="module">
  import { WebGLRenderer, PerspectiveCamera, Scene } from 'three';
  import { SplatLoader } from '@sparkjsdev/spark'; // Spark.js via npm or CDN

  const renderer = new WebGLRenderer({ canvas: document.getElementById('sceneCanvas') });
  const camera   = new PerspectiveCamera(60, canvas.width/canvas.height, 0.1, 100);
  const scene    = new Scene();

  // Load the splat (visual layer)
  const splat = await SplatLoader.load('scene.splat');
  scene.add(splat);

  // Load the proxy mesh (invisible, raycasting only)
  const { GLTFLoader } = await import('three/addons/loaders/GLTFLoader.js');
  new GLTFLoader().load('scene.glb', (gltf) => {
    proxyMesh = gltf.scene;
    proxyMesh.visible = false;
    scene.add(proxyMesh);
  });

  renderer.setAnimationLoop(() => renderer.render(scene, camera));
</script>
```

Once `proxyMesh` is loaded, the `canvas click` → raycast → bottom sheet flow in the existing code works without further changes. The pin coordinates will now be real 3D positions in the scene's local space.

### Optional: georeference the scene

If you want pin coordinates in WGS84 (real latitude/longitude) rather than local scene space — required for integration with LAJI.fi biodiversity data and city GIS layers — you need **Ground Control Points (GCPs)**:

1. Before your photo walk, mark 4–6 points on the ground with tape or spray chalk.
2. Record the GPS coordinate of each point with your phone or a GNSS receiver.
3. In Luma AI's advanced export settings, enter the GCP coordinates. Luma will solve the transform from scene space to WGS84.
4. The exported `.splat` will then include georeferencing metadata, and the `hits[0].point` from the raycaster can be projected to lat/lon using the transform matrix.

For the Phase 2 prototype, you can skip georeferencing entirely and use scene-local coordinates — the 2D vs. 3D comparison study only requires that pin positions are *comparable within each condition*, not that they map to real-world coordinates.

---

## Deploying to GitHub Pages

### If you already have a `username.github.io` repository

```bash
# Clone your repo
git clone https://github.com/username/username.github.io
cd username.github.io

# Copy the demo files
cp /path/to/index.html .
cp /path/to/scene.splat .   # if using real Luma AI files
cp /path/to/scene.glb .

# Commit and push
git add .
git commit -m "add 3DGS-PPGIS demo"
git push origin main
```

Live at: `https://username.github.io`

### If you don't have a GitHub Pages site yet

1. Go to [github.com/new](https://github.com/new)
2. Name the repository exactly `username.github.io` (replacing `username` with your GitHub username)
3. Set visibility to Public
4. Upload `index.html` (and your `.splat` / `.glb` if applicable)
5. Go to **Settings → Pages → Source → Deploy from branch → main → / (root)**
6. Live in approximately one minute

### Serving locally for development

Since the Spark.js splat loader uses `fetch()`, you need a local HTTP server — opening `index.html` directly from the filesystem will fail due to CORS restrictions on file:// URLs.

```bash
# Python (built in, no install needed)
cd /path/to/your/folder
python3 -m http.server 8000
# Open http://localhost:8000
```

Or with Node:

```bash
npx serve .
```

The demo HTML (without real Luma files) works fine opened directly as a file because it has no external asset fetches.

---

## File structure for the full instrument

When you expand this into the complete research instrument described in the research plan, the folder structure will look like:

```
ppgis-instrument/
├── index.html          ← this file (3D condition)
├── map.html            ← 2D Leaflet condition (identical annotation UI)
├── scene.splat         ← from Luma AI
├── scene.glb           ← from Luma AI (proxy mesh)
├── dashboard.html      ← researcher dashboard (D3.js, live pin map)
├── bio-scores.json     ← per-splat biodiversity scores from LAJI.fi
└── README.md           ← this file
```

The `bio-scores.json` file is a lookup table mapping splat point indices to biodiversity scores, generated by spatially joining LAJI.fi species occurrence records with the georeferenced splat point cloud. This is the file that drives the bio overlay shader uniform.

---

## Key references

- Fagerholm et al. (2021). A methodological framework for analysis of participatory mapping data. *IJGIS*, 35(9).
- Hasanzadeh et al. (2023). A methodological framework for analyzing PPGIS data collected in 3D. *International Journal of Digital Earth*.
- Linkola, Fagerholm & Eilola (2025). 3D Geovisualizations and Participatory Land-Use Planning. *Planning Theory and Practice*.
- Raymond et al. (2025). Applying multispecies justice in nature-based solutions. *npj Urban Sustainability*, 5.
- World Labs (2026, April 14). Streaming 3DGS worlds on the web: Spark 2.0. worldlabs.ai/blog/spark-2.0
