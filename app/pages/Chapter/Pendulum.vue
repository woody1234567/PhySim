<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-200 font-sans text-base-content"
  >
    <!-- Left Panel: 3D Simulation -->
    <div
      class="relative flex-grow h-full overflow-hidden"
      ref="canvasContainer"
    >
      <canvas ref="canvasRef" class="block w-full h-full outline-none"></canvas>

      <!-- Overlay: Status -->
      <div class="absolute top-4 left-4 pointer-events-none">
        <div v-if="!isRunning" class="badge badge-warning gap-2 shadow-lg">
          Paused
        </div>
        <div v-else class="badge badge-success gap-2 shadow-lg">
          Running (g = 9.8 m/s²)
        </div>
      </div>

      <!-- Overlay: Legend -->
      <div
        class="absolute top-4 right-4 pointer-events-none bg-base-100/80 p-2 rounded border border-base-300 text-xs shadow-lg backdrop-blur"
      >
        <div class="font-bold mb-1">Vectors</div>
        <div class="flex items-center gap-2">
          <span class="w-3 h-3 rounded-full bg-green-500"></span> Velocity
        </div>
        <div class="flex items-center gap-2">
          <span class="w-3 h-3 rounded-full bg-red-500"></span> Acceleration
        </div>
        <div class="flex items-center gap-2">
          <span class="w-3 h-3 rounded-full bg-yellow-400"></span> Tension
        </div>
      </div>
    </div>

    <!-- Resizer Handle -->
    <div
      class="w-2 h-full bg-base-300 hover:bg-primary cursor-col-resize flex items-center justify-center z-50 select-none transition-colors shadow-md"
      @mousedown="startResize"
    >
      <div class="h-8 w-1 bg-base-content/20 rounded"></div>
    </div>

    <!-- Right Panel: Sidebar -->
    <div
      class="flex flex-col h-full bg-base-100 border-l border-base-300 shadow-2xl shrink-0 z-40"
      :style="{ width: sidebarWidth + 'px' }"
    >
      <div
        class="p-4 border-b border-base-300 font-bold text-lg bg-base-200 flex justify-between items-center"
      >
        <span>Pendulum Lab</span>
        <div class="badge badge-ghost badge-sm">Physics Engine</div>
      </div>

      <div class="flex-grow overflow-y-auto p-2 space-y-2 custom-scrollbar">
        <!-- Panel 1: Controls -->
        <div
          class="collapse collapse-arrow bg-base-200 rounded-box border border-base-300"
        >
          <input type="radio" name="sidebar-accordion" checked="checked" />
          <div
            class="collapse-title text-sm font-bold uppercase tracking-wider"
          >
            Control Panel
          </div>
          <div class="collapse-content">
            <div class="flex flex-col gap-4 pt-2">
              <!-- Physics Parameters -->
              <div class="form-control w-full">
                <label class="label pt-0">
                  <span class="label-text font-semibold">Wire Length (m)</span>
                  <span class="label-text-alt font-mono text-primary">{{
                    params.length.toFixed(2)
                  }}</span>
                </label>
                <input
                  type="range"
                  min="1"
                  max="20"
                  step="0.1"
                  class="range range-xs range-primary"
                  v-model.number="params.length"
                  @input="requestReset"
                />
              </div>

              <div class="form-control w-full">
                <label class="label pt-0">
                  <span class="label-text font-semibold">Bob Mass (kg)</span>
                  <span class="label-text-alt font-mono text-secondary">{{
                    params.mass.toFixed(1)
                  }}</span>
                </label>
                <input
                  type="range"
                  min="0.1"
                  max="10"
                  step="0.1"
                  class="range range-xs range-secondary"
                  v-model.number="params.mass"
                  @input="requestReset"
                />
              </div>

              <div class="form-control w-full">
                <label class="label pt-0">
                  <span class="label-text font-semibold"
                    >Initial Angle (deg)</span
                  >
                  <span class="label-text-alt font-mono text-accent"
                    >{{ params.angle.toFixed(0) }}°</span
                  >
                </label>
                <input
                  type="range"
                  min="-90"
                  max="90"
                  step="1"
                  class="range range-xs range-accent"
                  v-model.number="params.angle"
                  @input="requestReset"
                />
              </div>

              <!-- Action Buttons -->
              <div class="divider my-1"></div>
              <div class="grid grid-cols-2 gap-2">
                <button
                  class="btn btn-sm btn-primary font-bold"
                  @click="toggleSimulation"
                >
                  {{ isRunning ? "Pause" : "Start" }}
                </button>
                <button
                  class="btn btn-sm btn-error btn-outline font-bold"
                  @click="resetSimulation"
                >
                  Restart
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- Panel 2: Live Data -->
        <div
          class="collapse collapse-arrow bg-base-200 rounded-box border border-base-300"
        >
          <input type="radio" name="sidebar-accordion" />
          <div
            class="collapse-title text-sm font-bold uppercase tracking-wider"
          >
            Live Data
          </div>
          <div class="collapse-content">
            <div class="overflow-x-auto pt-2">
              <table class="table table-xs table-zebra w-full font-mono">
                <thead>
                  <tr>
                    <th>Var</th>
                    <th class="text-right">Value</th>
                  </tr>
                </thead>
                <tbody>
                  <tr>
                    <td>Time (s)</td>
                    <td class="text-right font-bold">{{ displayData.t }}</td>
                  </tr>
                  <tr class="bg-yellow-500/10 font-bold text-yellow-600">
                    <td>Tension (N)</td>
                    <td class="text-right">{{ displayData.tension }}</td>
                  </tr>
                  <tr>
                    <td>Pos X</td>
                    <td class="text-right">{{ displayData.x }}</td>
                  </tr>
                  <tr>
                    <td>Pos Y</td>
                    <td class="text-right">{{ displayData.y }}</td>
                  </tr>
                  <tr>
                    <td>Vel X</td>
                    <td class="text-right">{{ displayData.vx }}</td>
                  </tr>
                  <tr>
                    <td>Vel Y</td>
                    <td class="text-right">{{ displayData.vy }}</td>
                  </tr>
                  <tr>
                    <td>Acc X</td>
                    <td class="text-right">{{ displayData.ax }}</td>
                  </tr>
                  <tr>
                    <td>Acc Y</td>
                    <td class="text-right">{{ displayData.ay }}</td>
                  </tr>
                </tbody>
              </table>
              <button
                class="btn btn-xs btn-block btn-ghost border-base-content/20 mt-4"
                @click="exportLogsToCSV"
              >
                📥 Export Simulation CSV
              </button>
            </div>
          </div>
        </div>

        <!-- Panel 3: Charts -->
        <div
          class="collapse collapse-arrow bg-base-200 rounded-box border border-base-300"
        >
          <input type="radio" name="sidebar-accordion" />
          <div
            class="collapse-title text-sm font-bold uppercase tracking-wider"
          >
            Charts
          </div>
          <div class="collapse-content">
            <div class="flex flex-col gap-3 pt-2">
              <div class="form-control">
                <label class="label cursor-pointer pt-0">
                  <span class="label-text">Plot Variable</span>
                  <select
                    class="select select-bordered select-xs"
                    v-model="chartVariable"
                  >
                    <option value="tension">Tension Force (N)</option>
                    <option value="y">Position Y (Height)</option>
                    <option value="vMag">Speed (|v|)</option>
                    <option value="aMag">Total Accel (|a|)</option>
                  </select>
                </label>
              </div>

              <div
                class="h-48 w-full bg-base-100 rounded border border-base-300 p-2 relative"
              >
                <canvas ref="chartCanvasRef"></canvas>
              </div>

              <div class="grid grid-cols-2 gap-2">
                <button
                  class="btn btn-xs btn-info text-white"
                  @click="updateChart"
                >
                  📈 Plot Chart
                </button>
                <button class="btn btn-xs btn-outline" @click="exportChartCSV">
                  📥 Export Data
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";

// ==========================================
// TYPE DEFINITIONS
// ==========================================
interface LogEntry {
  t: number;
  x: number;
  y: number;
  z: number;
  vx: number;
  vy: number;
  vz: number;
  ax: number;
  ay: number;
  az: number;
  tension: number;
}

// ==========================================
// STATE MANAGEMENT
// ==========================================
const sidebarWidth = ref(340);
const canvasContainer = ref<HTMLElement | null>(null);
const isRunning = ref(false);

// Physics Configuration
const params = reactive({
  length: 5.0,
  mass: 2.0,
  angle: 45, // degrees
});

// Live Data Display
const displayData = reactive({
  t: "0.00",
  x: "0.00",
  y: "0.00",
  z: "0.00",
  vx: "0.00",
  vy: "0.00",
  vz: "0.00",
  ax: "0.00",
  ay: "0.00",
  az: "0.00",
  tension: "0.00",
});

// Logging System
const logs = ref<LogEntry[]>([]);
const timeStep = 1 / 60;
let totalTime = 0;

// Charting
const chartCanvasRef = ref<HTMLCanvasElement | null>(null);
let chartInstance: Chart | null = null;
const chartVariable = ref("tension");

// Three.js Resources
const canvasRef = ref<HTMLCanvasElement | null>(null);
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let animationId: number;

// Simulation Meshes
let pivotMesh: THREE.Mesh;
let bobMesh: THREE.Mesh;
let wireLine: THREE.Line;
let traceLine: THREE.Line;
let traceGeometry: THREE.BufferGeometry;
const tracePoints: number[] = [];

// Vectors (Helpers)
let velocityArrow: THREE.ArrowHelper;
let accelArrow: THREE.ArrowHelper;
let tensionArrow: THREE.ArrowHelper;

// Physics Resources
let world: CANNON.World;
let pivotBody: CANNON.Body;
let bobBody: CANNON.Body;
let constraint: CANNON.DistanceConstraint;

// ==========================================
// LIFECYCLE
// ==========================================
onMounted(() => {
  if (typeof window !== "undefined") {
    initThree();
    initPhysics();
    initOrbitControls();
    resetSimulation();
    startRenderLoop();
    window.addEventListener("resize", handleWindowResize);
  }
});

onUnmounted(() => {
  if (typeof window !== "undefined") {
    cancelAnimationFrame(animationId);
    window.removeEventListener("resize", handleWindowResize);
    if (chartInstance) chartInstance.destroy();
    if (renderer) renderer.dispose();
  }
});

// ==========================================
// INITIALIZATION: THREE.JS
// ==========================================
function initThree() {
  if (!canvasRef.value || !canvasContainer.value) return;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x0f172a); // Slate-900

  const grid = new THREE.GridHelper(30, 30, 0x334155, 0x1e293b);
  grid.position.y = -10;
  scene.add(grid);

  // Camera
  const w = canvasContainer.value.clientWidth;
  const h = canvasContainer.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, w / h, 0.1, 1000);
  camera.position.set(0, 5, 25);

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.setSize(w, h);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;

  // Lights
  const ambient = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambient);

  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(10, 20, 10);
  dirLight.castShadow = true;
  scene.add(dirLight);

  // Objects
  // Pivot
  const pivotGeo = new THREE.BoxGeometry(0.5, 0.5, 0.5);
  const pivotMat = new THREE.MeshStandardMaterial({ color: 0x94a3b8 });
  pivotMesh = new THREE.Mesh(pivotGeo, pivotMat);
  scene.add(pivotMesh);

  // Bob
  const bobGeo = new THREE.SphereGeometry(0.8, 32, 32);
  const bobMat = new THREE.MeshStandardMaterial({
    color: 0x3b82f6,
    metalness: 0.3,
    roughness: 0.4,
  });
  bobMesh = new THREE.Mesh(bobGeo, bobMat);
  bobMesh.castShadow = true;
  scene.add(bobMesh);

  // Wire
  const wireGeo = new THREE.BufferGeometry();
  const wireMat = new THREE.LineBasicMaterial({
    color: 0xffffff,
    linewidth: 2,
  });
  wireLine = new THREE.Line(wireGeo, wireMat);
  scene.add(wireLine);

  // Trace
  traceGeometry = new THREE.BufferGeometry();
  const traceMat = new THREE.LineBasicMaterial({ color: 0xf43f5e });
  traceLine = new THREE.Line(traceGeometry, traceMat);
  traceLine.frustumCulled = false;
  scene.add(traceLine);

  // Vectors
  velocityArrow = new THREE.ArrowHelper(
    new THREE.Vector3(1, 0, 0),
    new THREE.Vector3(0, 0, 0),
    1,
    0x22c55e // Green
  );
  scene.add(velocityArrow);

  accelArrow = new THREE.ArrowHelper(
    new THREE.Vector3(1, 0, 0),
    new THREE.Vector3(0, 0, 0),
    1,
    0xef4444 // Red
  );
  scene.add(accelArrow);

  tensionArrow = new THREE.ArrowHelper(
    new THREE.Vector3(1, 0, 0),
    new THREE.Vector3(0, 0, 0),
    1,
    0xfacc15 // Yellow
  );
  scene.add(tensionArrow);
}

function initOrbitControls() {
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
}

// ==========================================
// INITIALIZATION: PHYSICS
// ==========================================
function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, -9.8, 0); // Fixed Gravity

  // Solver configuration for stability with rigid constraints
  world.solver.iterations = 20;
  world.defaultContactMaterial.contactEquationStiffness = 1e9;
  world.defaultContactMaterial.contactEquationRelaxation = 3;
}

function resetSimulation() {
  isRunning.value = false;
  totalTime = 0;

  // Clear logs
  logs.value = [];
  tracePoints.length = 0;
  traceGeometry.setFromPoints([]);

  // Clear Physics
  world.bodies = [];
  world.constraints = [];

  // Pivot Position
  const pivotY = 10;
  const pivotPos = new CANNON.Vec3(0, pivotY, 0);

  // 1. Pivot Body (Static)
  pivotBody = new CANNON.Body({ mass: 0 });
  pivotBody.position.copy(pivotPos);
  world.addBody(pivotBody);
  pivotMesh.position.copy(pivotPos as any);

  // Calculate Bob Position
  const theta = params.angle * (Math.PI / 180);
  const bobX = params.length * Math.sin(theta);
  const bobY = pivotY - params.length * Math.cos(theta);
  const bobZ = 0;

  // 2. Bob Body (Dynamic)
  bobBody = new CANNON.Body({
    mass: params.mass,
    position: new CANNON.Vec3(bobX, bobY, bobZ),
    shape: new CANNON.Sphere(0.8),
    linearDamping: 0.001, // Minimal air resistance
  });
  world.addBody(bobBody);

  // 3. Wire (Distance Constraint)
  // Forces distance between Pivot and Bob to be exactly 'params.length'
  constraint = new CANNON.DistanceConstraint(pivotBody, bobBody, params.length);
  world.addConstraint(constraint);

  // Initial State Update
  updateVisuals(new CANNON.Vec3(0, 0, 0), new CANNON.Vec3(0, 0, 0));
  updateDataUI(new CANNON.Vec3(0, 0, 0), new CANNON.Vec3(0, 0, 0));
}

// ==========================================
// MAIN LOOP
// ==========================================
function startRenderLoop() {
  const tick = () => {
    animationId = requestAnimationFrame(tick);

    if (isRunning.value) {
      stepPhysics();
    }

    controls.update();
    renderer.render(scene, camera);
  };
  tick();
}

function stepPhysics() {
  const vPrev = bobBody.velocity.clone();

  world.step(timeStep);
  totalTime += timeStep;

  const vCurr = bobBody.velocity;

  // Calculate Total Acceleration (a = dv/dt)
  const accel = vCurr.vsub(vPrev).scale(1 / timeStep);

  // Calculate Tension Force
  // F_net = F_gravity + F_tension
  // m*a = m*g + T
  // T = m*(a - g)
  const gravity = new CANNON.Vec3(0, -9.8, 0);
  const a_minus_g = accel.vsub(gravity);
  const tensionVec = a_minus_g.scale(params.mass);

  // Log & Update
  logFrame(accel, tensionVec);
  updateVisuals(accel, tensionVec);

  if (Math.floor(totalTime * 60) % 4 === 0) {
    updateDataUI(accel, tensionVec);
  }
}

function updateVisuals(accel: CANNON.Vec3, tension: CANNON.Vec3) {
  // Sync Meshes
  bobMesh.position.copy(bobBody.position as any);
  bobMesh.quaternion.copy(bobBody.quaternion as any);

  // Wire
  const points = [pivotMesh.position, bobMesh.position];
  wireLine.geometry.setFromPoints(points);

  // Trace
  tracePoints.push(bobMesh.position.x, bobMesh.position.y, bobMesh.position.z);
  if (tracePoints.length > 9000) tracePoints.splice(0, 3);
  traceGeometry.setAttribute(
    "position",
    new THREE.Float32BufferAttribute(tracePoints, 3)
  );
  traceGeometry.computeBoundingSphere();

  // Velocity Vector (Green)
  const v = bobBody.velocity;
  const vLen = v.length();
  if (vLen > 0.01) {
    velocityArrow.visible = true;
    velocityArrow.setDirection(new THREE.Vector3(v.x, v.y, v.z).normalize());
    velocityArrow.setLength(vLen * 0.3);
    velocityArrow.position.copy(bobMesh.position);
  } else {
    velocityArrow.visible = false;
  }

  // Acceleration Vector (Red)
  const aLen = accel.length();
  if (aLen > 0.01) {
    accelArrow.visible = true;
    accelArrow.setDirection(
      new THREE.Vector3(accel.x, accel.y, accel.z).normalize()
    );
    accelArrow.setLength(aLen * 0.2);
    accelArrow.position.copy(bobMesh.position);
  } else {
    accelArrow.visible = false;
  }

  // Tension Vector (Yellow)
  const tLen = tension.length();
  if (tLen > 0.01) {
    tensionArrow.visible = true;
    // Tension pulls ON the bob TOWARDS the pivot
    tensionArrow.setDirection(
      new THREE.Vector3(tension.x, tension.y, tension.z).normalize()
    );
    tensionArrow.setLength(tLen * 0.05); // Tension can be high, scale down
    tensionArrow.position.copy(bobMesh.position);
  } else {
    tensionArrow.visible = false;
  }
}

// ==========================================
// DATA HANDLING
// ==========================================
function logFrame(accel: CANNON.Vec3, tension: CANNON.Vec3) {
  logs.value.push({
    t: totalTime,
    x: bobBody.position.x,
    y: bobBody.position.y,
    z: bobBody.position.z,
    vx: bobBody.velocity.x,
    vy: bobBody.velocity.y,
    vz: bobBody.velocity.z,
    ax: accel.x,
    ay: accel.y,
    az: accel.z,
    tension: tension.length(),
  });
}

function updateDataUI(accel: CANNON.Vec3, tension: CANNON.Vec3) {
  displayData.t = totalTime.toFixed(2);

  displayData.x = bobBody.position.x.toFixed(2);
  displayData.y = bobBody.position.y.toFixed(2);
  displayData.z = bobBody.position.z.toFixed(2);

  displayData.vx = bobBody.velocity.x.toFixed(2);
  displayData.vy = bobBody.velocity.y.toFixed(2);

  displayData.ax = accel.x.toFixed(2);
  displayData.ay = accel.y.toFixed(2);

  displayData.tension = tension.length().toFixed(2);
}

function exportLogsToCSV() {
  if (logs.value.length === 0) return;

  const header = [
    "t",
    "x",
    "y",
    "z",
    "vx",
    "vy",
    "vz",
    "ax",
    "ay",
    "az",
    "tension_N",
  ];
  const csvRows = logs.value.map((row) =>
    [
      row.t,
      row.x,
      row.y,
      row.z,
      row.vx,
      row.vy,
      row.vz,
      row.ax,
      row.ay,
      row.az,
      row.tension,
    ].join(",")
  );

  const content = [header.join(","), ...csvRows].join("\n");
  downloadFile(content, "pendulum_data.csv");
}

function downloadFile(content: string, fileName: string) {
  const blob = new Blob([content], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.href = url;
  link.setAttribute("download", fileName);
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

// ==========================================
// CHARTING
// ==========================================
function updateChart() {
  if (!chartCanvasRef.value) return;

  if (chartInstance) chartInstance.destroy();

  const data = logs.value.filter((_, i) => i % 5 === 0);
  const labels = data.map((d) => d.t.toFixed(2));

  let dataset: number[] = [];
  let labelText = "";
  let color = "#3b82f6";

  switch (chartVariable.value) {
    case "tension":
      dataset = data.map((d) => d.tension);
      labelText = "Tension (N)";
      color = "#facc15";
      break;
    case "y":
      dataset = data.map((d) => d.y);
      labelText = "Height Y (m)";
      color = "#3b82f6";
      break;
    case "vMag":
      dataset = data.map((d) => Math.sqrt(d.vx ** 2 + d.vy ** 2 + d.vz ** 2));
      labelText = "Speed (m/s)";
      color = "#10b981";
      break;
    case "aMag":
      dataset = data.map((d) => Math.sqrt(d.ax ** 2 + d.ay ** 2 + d.az ** 2));
      labelText = "Acceleration (m/s²)";
      color = "#ef4444";
      break;
  }

  chartInstance = new Chart(chartCanvasRef.value, {
    type: "line",
    data: {
      labels,
      datasets: [
        {
          label: labelText,
          data: dataset,
          borderColor: color,
          backgroundColor: color + "33",
          borderWidth: 2,
          pointRadius: 0,
          tension: 0.1,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: false,
      interaction: { mode: "index", intersect: false },
      scales: {
        x: { ticks: { maxTicksLimit: 8 } },
      },
    },
  });
}

function exportChartCSV() {
  if (!logs.value.length) return;
  const rows = logs.value.map((l) => {
    let val = 0;
    if (chartVariable.value === "tension") val = l.tension;
    // ... handles other cases via simple filter if needed or export full log
    return `${l.t},${val}`;
  });
  downloadFile(
    `time,${chartVariable.value}\n` + rows.join("\n"),
    `chart_${chartVariable.value}.csv`
  );
}

// ==========================================
// UI & RESIZER
// ==========================================
function toggleSimulation() {
  isRunning.value = !isRunning.value;
}
function requestReset() {
  resetSimulation();
}
function startResize(e: MouseEvent) {
  e.preventDefault();
  document.addEventListener("mousemove", onResize);
  document.addEventListener("mouseup", stopResize);
  document.body.style.cursor = "col-resize";
}
function onResize(e: MouseEvent) {
  const newSidebarW = window.innerWidth - e.clientX;
  const clamped = Math.max(260, Math.min(newSidebarW, 600));
  sidebarWidth.value = Math.min(clamped, window.innerWidth - 300);
  handleWindowResize();
}
function stopResize() {
  document.removeEventListener("mousemove", onResize);
  document.removeEventListener("mouseup", stopResize);
  document.body.style.cursor = "";
}
function handleWindowResize() {
  if (!canvasContainer.value || !camera || !renderer) return;
  const w = canvasContainer.value.clientWidth;
  const h = canvasContainer.value.clientHeight;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
}
</script>

<style scoped>
.custom-scrollbar::-webkit-scrollbar {
  width: 6px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: #1e293b;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background: #475569;
  border-radius: 10px;
}
canvas {
  display: block;
}
</style>
