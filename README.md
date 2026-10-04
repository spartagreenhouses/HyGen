# Solar Hydrogen Energy Storage System — 3D Digital Twin

Interactive 3D digital twin, process simulation and SCADA-style web HMI for a
**150 kWp PV → Enapter AEM Flex 120 → separate H₂ / O₂ storage** plant with electrolyzer heat recovery.

This is version 1 of the real plant-control web app: the simulation engine publishes exactly the same
`PlantTelemetry` contract that the real backend will publish, so the UI does not change when the
PLC / MQTT / sensors are connected.

```
PHYSICAL PLANT → PLC / CONTROLLER → MQTT / WebSocket → BACKEND → ┐
                                                                 ├─ TelemetryProvider ─► DIGITAL TWIN WEB APP
SIMULATION ENGINE ───────────────────────────────────────────────┘   (same interface)
```

---

## Quick start

```bash
npm install
npm run dev          # http://localhost:5173
```

Other scripts

| command | what it does |
|---|---|
| `npm run build` | type-check + production build to `dist/` |
| `npm test` | physics & topic-codec tests (speed independence, mass balances, interlocks, E-stop) |
| `npm run mock:ws` | stand-alone mock backend on `ws://localhost:8787` (same engine, real topics) |
| `npm run models:compress` | Draco-compress every GLB in `public/models/` |

Node 18+ (tested on Node 22).

### First things to try
1. **DEMO** (top bar) — one solar day at 360×: dawn standby → ramp from 12 % → full load at midday → evening ramp-down → night, gas stays stored.
2. **START SYSTEM**, then drag **Time of day** or switch to **MANUAL kW** (0–150 kW).
3. Click any machine (or its label) → equipment panel with live values, warnings, commands and parameters with their source.
4. **SIMULATION TOOLS → H₂ compressor trip** — watch the buffer pressure rise, the back-pressure interlock stop the electrolyzer, and the alarm appear.
5. 🔧 **Engineering mode** — connection points, pipe names, IDs, sensor tags, XYZ, bounding boxes, move gizmo, “Copy layout JSON”.
6. **DATA → LIVE** — the same UI fed by MQTT-style topics (in-browser mock broker by default), read-only.

---

## Your 3D models

The STEP files you supplied were converted to GLB (metres, Y-up, base on y = 0, X/Z centred, Draco-compressed)
with `tools/step2glb.py` and placed in `public/models/`:

| equipment id | GLB | from your CAD | notes |
|---|---|---|---|
| `electrolyzer` | `aem_flex_120.glb` | AEM-Flex-120.STEP | 5.9 × 2.68 × 1.5 m |
| `h2Compressor` | `h2_compressor.glb` | Booster.STEP | used as TBF-30-300-025 skid |
| `h2Storage` | `h2_storage.glb` | Gaznet-40ft-Iso-Standart-Type-IV.STEP | rotated 90° to lie along X |
| `o2Storage` | `o2_quad64.glb` | O2-Balon-Qwad-64.STEP | 4 instances on one manifold |
| `solar` | `solar.glb`, `solar_lod1.glb` | Solar-Panel.STEP | 273 instances (150 kWp ÷ 550 Wp), LOD for the array |
| `waterTank` | `water_tank.glb` | Watter-Tank.STEP | |
| `inverter` | `inverter.glb` | Invertor.STEP | CAD lies flat → rotated −90° X |
| `circulationPump` | `circulation_pump.glb` | Mator.STEP | heat-loop pump P-401 |

**Placeholders** (no CAD yet — procedural, same id / ports / sensors): `o2Compressor` (generic O₂ booster),
`h2Dryer`, `h2Buffer`, `o2Dryer`, `heatExchanger`, `thermalTank`, `heatConsumer` (shape follows the selected heat destination).

**Adding / replacing a model:** drop the GLB in `public/models/` with the filename configured in
`src/config/equipmentConfig.ts` (e.g. `o2_compressor.glb`, `heat_exchanger.glb`, `thermal_tank.glb`) and reload.
The loader probes the file and swaps the placeholder automatically — no code changes, the simulation is untouched.
If the model has its own materials, set `material.mode: 'model'` for that equipment.

Converting more STEP files:
```bash
pip install gmsh trimesh numpy fast-simplification
python3 tools/step2glb.py MyPart.STEP public/models/my_part.glb      # --size/--curv control mesh density
npm run models:compress
```

---

## Positioning equipment

Everything is in **`src/config/equipmentConfig.ts`** — one entry per equipment:

```ts
{ id: 'electrolyzer', model: '/models/aem_flex_120.glb',
  position: [0, 0, 0], rotation: [0, 0, 0] /* degrees */, scale: [1, 1, 1],
  ports: [{ id: 'H2_OUT', position: [2.95, 2.2, -0.35], dir: [1, 0, 0], medium: 'H2' }, …],
  sensors: [{ tag: 'TT-101', signal: 'electrolyzer.stackTempC', … }] }
```

World axes: **+X east, +Y up, −Z north** (PV field north of the process area). Ports are in the equipment's local frame.
The nozzle positions on your CAD are estimates — use Engineering mode to see the port markers and adjust.

Workflow: Engineering mode → select equipment → drag gizmo → pipes re-route live → **Copy layout JSON** → paste positions
into `equipmentConfig.ts`. (Gizmo moves are also remembered in the browser until “Reset moves”.)

## Pipelines

`src/config/pipelineConfig.ts` connects named ports, e.g. `electrolyzer.H2_OUT → h2Dryer.IN`.
Automatic orthogonal routing (`elevation`, `order: 'xz' | 'zx'`), or hand-routed `waypoints`.

| medium | colour | pattern / particle / tag |
|---|---|---|
| Electricity | yellow | dashed cable, fast travelling pulses, “⚡ kW” |
| Water | blue | solid pipe, droplets, “H₂O” |
| Hydrogen | green | double band rings, spheres, “H₂” |
| Oxygen | white | single band rings, cubes, “O₂” |
| Electrolyzer coolant | red | chevrons, “COOLANT” (return lines darker + “RET”) |
| Thermal water | orange | chevrons, “HEAT” |

H₂ and O₂ have completely separate lines, compressors and storage. There is no mixed-gas vessel anywhere.
Particle speed and visibility follow the live flow of each line. Valves (XV-101/201/301/401) are clickable.

---

## Configuration & engineering accuracy

**All plant constants live in `src/config/plantSpec.ts`** and every value carries its source:

| source | meaning | UI badge |
|---|---|---|
| `MANUFACTURER_SPEC` | from a datasheet you supplied (AEM Flex 120, Gaznet, Quad 64) | MANUFACTURER SPEC |
| `PROJECT_DESIGN` | your plant design values (150 kWp, 5000 L, 4 bundles, 300 bar…) | PROJECT DESIGN |
| `ENGINEERING_ASSUMPTION` | simulation values until real data exists | ⚠ SIMULATION ASSUMPTION |
| `PHYSICAL_CONSTANT` | H₂ LHV 33.3 kWh/kg, densities | PHYSICAL CONSTANT |
| `LIVE_SENSOR_VALUE` | measured values (LIVE mode) | LIVE SENSOR |

The **PARAMETERS** tab lists every parameter with its source and note. Assumptions to verify first:
compressor electrical power (K-201 7.5 kW, K-301 5.5 kW — configurable, no manufacturer data), TBF flow/inlet
inferred from the model code, Quad 64 water volume (640 L/bundle), recoverable heat ratio 0.85, heat-recovery
efficiency 0.80, buffer tank 2000 L, PV peak AC fraction 0.88, site latitude (Yerevan by default).

### Key calculations
- H₂ kg/h = P_electrolyzer / 51.3, clamped to 2.16 kg/h. Load follows available PV (minus aux & compressors) within 12–100 %, with hysteresis; never draws more than PV supplies.
- Water = 19.4 L/h × (H₂ / 2.16). O₂ = 8 × H₂ (spec approximation).
- Storage pressure from mass with real-gas correction Z ≈ 1 + α·p (H₂ 300 bar ⇔ 645 kg).
- Production stops at the H₂ safety limit (95 %, restart below 93 %). O₂ full → vent (configurable: or stop).
- **Process heat is estimated** as P_el − H₂·LHV (no manufacturer heat value). Recoverable = × ratio, transferred = × efficiency, delivered = min(consumer demand, buffer).
- **H₂ CONVERSION EFFICIENCY** = H₂ LHV ÷ electrolyzer input (≈ 64–65 %).
- **TOTAL USEFUL ENERGY UTILIZATION** = (H₂ LHV + useful heat) ÷ total plant input. Not a round-trip efficiency.
- Fixed 1 s integration sub-steps → identical results at 1× and 1440× (tested).

Plant modes: OFF · STARTING · RUNNING · PARTIAL_LOAD (< 50 %) · FULL_LOAD (≥ 95 %) · STOPPING · FAULT.
While the system is RUNNING without enough sun the electrolyzer status is **STANDBY**.

---

## Data modes, MQTT & WebSocket

`DATA: SIMULATION | LIVE` in the top bar. The UI only uses the `TelemetryProvider` interface
(`src/services/telemetry.ts`): `SimulationTelemetryProvider` and `LiveTelemetryProvider`.

LIVE transport is set in `src/config/liveConfig.ts` / `.env.local` (see `.env.example`):

| `VITE_LIVE_TRANSPORT` | source |
|---|---|
| `mock` (default) | in-browser mock broker — no server needed |
| `websocket` | `VITE_WS_URL`, frames `{topic, payload}` — try `npm run mock:ws` |
| `mqtt` | MQTT over WebSockets (`VITE_MQTT_URL`, subscribes `plant/#`) |

Topics (full list in `src/services/topics.ts`), payload `{"v": value, "ts": epochMs}` or a raw value:

```
plant/solar/power                 plant/h2/compressor/status|inlet_pressure|outlet_pressure|flow|power|temperature
plant/electrolyzer/status         plant/h2/storage/pressure|level|mass
plant/electrolyzer/power          plant/o2/compressor/…   plant/o2/storage/level|mass
plant/electrolyzer/h2_flow        plant/o2/storage/bundle/{1..4}/level|pressure
plant/electrolyzer/water_flow     plant/water/level|volume|flow
plant/electrolyzer/temperature    plant/thermal/power|energy|buffer_temp|destination
plant/mode   plant/time           plant/energy/…   plant/counters/…   plant/valve/{XV-…}
plant/alarm  (single alarm JSON)  plant/alarms (full list JSON)
plant/cmd  →  plant/cmd/ack       (commands, only when LIVE control is enabled)
```

### Commands & safety
`PlantCommand` (start/stop system, E-stop, electrolyzer start/stop/load, compressors start/stop, thermal destination,
valves, alarm ack) is defined in `src/types/commands.ts`. In this version **no command reaches hardware**:
LIVE mode is read-only (`VITE_LIVE_CONTROL=false`). If enabled later, the app only publishes a *request* on
`plant/cmd` and waits for `plant/cmd/ack`; the PLC must remain the authority for interlocks and permissives,
and the backend must add authentication / authorisation before forwarding anything.

---

## Project structure

```
src/
  config/        plantSpec.ts (all constants + sources) · equipmentConfig.ts · pipelineConfig.ts · valves.ts · liveConfig.ts
  simulation/    simulationEngine.ts · solarModel.ts · hydrogenModel.ts · oxygenModel.ts · thermalModel.ts
                 compressorModel.ts · alarmModel.ts · demoScenario.ts · __tests__/
  services/      telemetry.ts (providers) · topics.ts · websocket.ts · mqtt.ts · mockBroker.ts · history.ts
  state/         plantStore.ts (Zustand)
  types/         telemetry.ts (PlantTelemetry contract) · commands.ts
  components/
    equipment/   SolarPlant · WaterTank · Electrolyzer · HydrogenCompressor · HydrogenStorage · OxygenCompressor
                 OxygenStorage · HeatExchanger · ThermalStorage · GasTreatment · Inverter · CirculationPump · HeatConsumer
                 EquipmentShell (transform, label, engineering overlays, gizmo) · indicators
    pipes/       Pipeline · Valve · routing · flowSignals
    scene/       PlantScene · ModelLoader (GLB + Draco + placeholder fallback) · Placeholder · transforms
    panels/      ControlPanel · KpiPanel · EquipmentInfoPanel · AlarmPanel · EventLog · ParameterTable · BottomDrawer
    charts/      TrendPanel (Live / 24 h / 7 d / 30 d)
    ui/          TopBar · FlowLegend · DemoCaption · EngineeringToolbar · SourceBadge · Toast
server/          mockTelemetryServer.ts (Node WebSocket backend stand-in)
tools/           step2glb.py · compressModels.mjs
public/models/   GLB files      public/draco/   local Draco decoder (works offline)
```

## Performance notes
Draco-compressed GLBs (4.2 MB → 0.9 MB), PV array as instanced meshes with a low-poly LOD (273 modules in a
handful of draw calls), flow particles instanced per pipe, 3D scene / charts / MQTT code-split and lazy-loaded,
model files probed once and cached, one 2048² shadow map, telemetry read inside `useFrame` (no React re-render per frame).
