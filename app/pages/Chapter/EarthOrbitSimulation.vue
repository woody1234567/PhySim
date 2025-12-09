<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-300 font-sans"
    @mousemove="handleResizeMove"
    @mouseup="handleResizeUp"
    @mouseleave="handleResizeUp"
  >
    <!-- 3D Simulation Panel -->
    <div
      ref="viewport"
      class="relative flex-grow h-full bg-black overflow-hidden"
      :style="{ minWidth: '300px' }"
    >
      <div
        class="absolute top-4 left-4 text-white/50 text-xs pointer-events-none select-none z-10"
      >
        Three.js + Cannon-es Physics
      </div>
    </div>

    <!-- Resize Handle -->
    <div
      class="w-1 h-full cursor-col-resize bg-base-content/20 hover:bg-primary transition-colors z-50 flex-shrink-0"
      @mousedown="handleResizeDown"
    ></div>

    <!-- Sidebar Control Panel -->
    <div
      class="flex flex-col bg-base-200 h-full overflow-y-auto shrink-0 border-l border-base-300 shadow-xl"
      :style="{
        width: sidebarWidth + 'px',
        minWidth: '260px',
        maxWidth: '600px',
      }"
    >
      <!-- Panel 1: Controls -->
      <div
        class="collapse collapse-arrow border-b border-base-300 rounded-none"
        :class="{ 'collapse-open': panels.controls }"
      >
        <div
          class="collapse-title text-sm font-bold bg-base-100 uppercase tracking-wide"
          @click="panels.controls = !panels.controls"
        >
          Control Panel
        </div>
        <div class="collapse-content bg-base-200 pt-4 space-y-4">
          <!-- Orbit Speed Scale -->
          <div class="form-control w-full">
            <label class="label pb-1">
              <span class="label-text font-bold">Orbit Speed Scale</span>
              <span class="label-text-alt">{{ orbitSpeedScale }}x</span>
            </label>
            <input
              type="range"
              min="0.1"
              max="5"
              step="0.1"
              class="range range-xs range-primary"
              v-model.number="orbitSpeedScale"
            />
          </div>

          <!-- Time Scale -->
          <div class="form-control w-full">
            <label class="label pb-1">
              <span class="label-text font-bold">Time Scale</span>
              <span class="label-text-alt">{{ timeScale }}x</span>
            </label>
            <input
              type="range"
              min="1"
              max="1000000"
              step="100"
              class="range range-xs range-secondary"
              v-model.number="timeScale"
            />
          </div>

          <!-- Show Vectors -->
          <div class="form-control">
            <label class="label cursor-pointer justify-start gap-4 p-0">
              <span class="label-text font-bold">Show Vectors</span>
              <input
                type="checkbox"
                class="toggle toggle-primary toggle-sm"
                v-model="showVectors"
              />
            </label>
          </div>

          <!-- Action Buttons -->
          <div class="flex gap-2 pt-2">
            <button class="btn btn-primary flex-1 btn-sm" @click="togglePause">
              {{ isPaused ? "Resume" : "Pause" }}
            </button>
            <button
              class="btn btn-warning flex-1 btn-sm"
              @click="restartSimulation"
            >
              Restart
            </button>
          </div>
        </div>
      </div>

      <!-- Panel 2: Live Data -->
      <div
        class="collapse collapse-arrow border-b border-base-300 rounded-none"
        :class="{ 'collapse-open': panels.data }"
      >
        <div
          class="collapse-title text-sm font-bold bg-base-100 uppercase tracking-wide"
          @click="panels.data = !panels.data"
        >
          Live Data
        </div>
        <div class="collapse-content bg-base-200 pt-4 space-y-2">
          <div class="grid grid-cols-2 gap-x-4 gap-y-1 text-xs font-mono">
            <span class="opacity-70">Times (s):</span>
            <span class="text-right">{{ dataDisplay.t }}</span>
            <span class="opacity-70">Pos X (AU):</span>
            <span class="text-right">{{ dataDisplay.x }}</span>
            <span class="opacity-70">Pos Y (AU):</span>
            <span class="text-right">{{ dataDisplay.y }}</span>
            <span class="opacity-70">Pos Z (AU):</span>
            <span class="text-right">{{ dataDisplay.z }}</span>
            <span class="opacity-70">Vel (km/s):</span>
            <span class="text-right">{{ dataDisplay.v }}</span>
          </div>
          <button
            class="btn btn-outline btn-xs w-full mt-2"
            @click="exportSimulationCSV"
          >
            Export Simulation CSV
          </button>
        </div>
      </div>

      <!-- Panel 3: Charts -->
      <div
        class="collapse collapse-arrow border-b border-base-300 rounded-none"
        :class="{ 'collapse-open': panels.charts }"
      >
        <div
          class="collapse-title text-sm font-bold bg-base-100 uppercase tracking-wide"
          @click="panels.charts = !panels.charts"
        >
          Chart Panel
        </div>
        <div class="collapse-content bg-base-200 pt-4 space-y-4">
          <div class="h-48 w-full bg-base-100 rounded p-2 relative">
            <canvas ref="chartCanvas"></canvas>
          </div>

          <!-- Window Slider -->
          <div class="form-control w-full">
            <label class="label pb-1">
              <span class="label-text font-bold">Window (Real Sec)</span>
              <span class="label-text-alt">{{ chartWindowSeconds }}s</span>
            </label>
            <input
              type="range"
              min="1"
              max="20"
              step="1"
              class="range range-xs range-accent"
              v-model.number="chartWindowSeconds"
            />
          </div>

          <div class="flex gap-2">
            <button
              class="btn btn-ghost btn-xs flex-1 border border-base-content/10"
              @click="resetCharts"
            >
              Plot Chart (Reset)
            </button>
            <button
              class="btn btn-ghost btn-xs flex-1 border border-base-content/10"
              @click="exportChartCSV"
            >
              Export Chart CSV
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, reactive, watch, nextTick } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";

// 1. CONSTANTS (DSL)
// --------------------------------------------------------------------------
const CONSTANTS = {
  G: 6.674e-11,
  sunMass: 1.989e30,
  earthMass: 5.972e24,
  AU: 1.496e11,
  stopTime: 31557600,
};

// 2. STATE & REFS
// --------------------------------------------------------------------------
const viewport = ref<HTMLElement | null>(null);
const chartCanvas = ref<HTMLCanvasElement | null>(null);

// UI State
const sidebarWidth = ref(320);
const panels = reactive({ controls: true, data: true, charts: true });
const orbitSpeedScale = ref(1);
const timeScale = ref(5000);
const showVectors = ref(true);
const isPaused = ref(false);
const chartWindowSeconds = ref(5);

// Display State
const dataDisplay = reactive({ t: "0", x: "0", y: "0", z: "0", v: "0" });

// Simulation Vars
let reqId: number;
let simTime = 0;
let logs: any[] = [];

// Three.js Objects
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let sunMesh: THREE.Mesh;
let earthMesh: THREE.Mesh;
let vectorArrow: THREE.ArrowHelper;
let trailGeometry: THREE.BufferGeometry;
let trailPositions: Float32Array;
let trailIdx = 0;
const MAX_TRAIL_POINTS = 10000;

// Physics Objects
let world: CANNON.World;
let sunBody: CANNON.Body;
let earthBody: CANNON.Body;

// Chart Objects
let chartInstance: Chart | null = null;

// 3. INITIALIZATION
// --------------------------------------------------------------------------

const initPhysics = () => {
  world = new CANNON.World();
  world.gravity.set(0, 0, 0); // N-body gravity is manual
  world.defaultContactMaterial.friction = 0;
  world.defaultContactMaterial.restitution = 0;

  // Sun (Static)
  sunBody = new CANNON.Body({ mass: 0 });
  sunBody.position.set(0, 0, 0);
  world.addBody(sunBody);

  // Earth
  // Start at (AU, 0, 0)
  // Velocity (0, 29780 * scale, 0) - standard circular orbit
  const vScale = 29780 * orbitSpeedScale.value;
  earthBody = new CANNON.Body({ mass: CONSTANTS.earthMass });
  earthBody.position.set(CONSTANTS.AU, 0, 0);
  earthBody.velocity.set(0, vScale, 0);
  earthBody.linearDamping = 0;
  earthBody.angularDamping = 0;
  world.addBody(earthBody);
};

const initThree = () => {
  if (!viewport.value) return;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x050505);

  // Camera
  // Look from top-ish side. Distance ~ 2-3 AU
  const aspect = viewport.value.clientWidth / viewport.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, aspect, 1e8, 1e14);
  camera.position.set(0, 0, 3 * CONSTANTS.AU);
  camera.up.set(0, 1, 0);
  camera.lookAt(0, 0, 0);

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(viewport.value.clientWidth, viewport.value.clientHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  viewport.value.appendChild(renderer.domElement);

  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.screenSpacePanning = true;

  // Objects
  // Scale for visibility: AU is 1.5e11. Sun R is 7e8.
  // We make Sun larger manually for diagram purpose.
  // Let's use 2e9 (~3x real sun) for visibility?
  // DSL said sphere(30)... likely arbitrary units.
  // Let's use a scale factor where 1 unit = 1e9 meters
  // No, let's keep physics units and just make geometry huge.
  const visualScale = 1e9;

  const sunGeo = new THREE.SphereGeometry(30 * visualScale, 32, 32);
  const sunMat = new THREE.MeshBasicMaterial({ color: 0xffff00 });
  sunMesh = new THREE.Mesh(sunGeo, sunMat);
  scene.add(sunMesh);

  const earthGeo = new THREE.SphereGeometry(10 * visualScale, 32, 32);
  const earthMat = new THREE.MeshPhongMaterial({
    color: 0x4488ff,
    shininess: 30,
  });
  earthMesh = new THREE.Mesh(earthGeo, earthMat);
  scene.add(earthMesh);

  // Trail
  trailGeometry = new THREE.BufferGeometry();
  trailPositions = new Float32Array(MAX_TRAIL_POINTS * 3);
  trailGeometry.setAttribute(
    "position",
    new THREE.BufferAttribute(trailPositions, 3)
  );
  const trailMat = new THREE.LineBasicMaterial({
    color: 0xffffff,
    opacity: 0.4,
    transparent: true,
  });
  const trail = new THREE.Line(trailGeometry, trailMat);
  trail.frustumCulled = false;
  scene.add(trail);

  // Arrow
  // Direction is velocity
  vectorArrow = new THREE.ArrowHelper(
    new THREE.Vector3(1, 0, 0),
    new THREE.Vector3(0, 0, 0),
    1,
    0xff0000
  );
  scene.add(vectorArrow);

  // Light
  const ambient = new THREE.AmbientLight(0x333333);
  scene.add(ambient);
  const point = new THREE.PointLight(0xffffff, 2, 0, 0);
  scene.add(point);

  // Starfield logic optional... skip for constraint "One file".
};

const initChart = () => {
  if (!chartCanvas.value) return;
  const ctx = chartCanvas.value.getContext("2d");
  if (!ctx) return;

  chartInstance = new Chart(ctx, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        {
          label: "Earth Distance (AU)",
          data: [],
          borderColor: "rgb(75, 192, 192)",
          borderWidth: 1.5,
          pointRadius: 0,
          tension: 0.1,
        },
        {
          label: "Earth Speed (km/s)",
          data: [],
          borderColor: "rgb(255, 99, 132)",
          borderWidth: 1.5,
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
      plugins: { legend: { labels: { boxWidth: 10 } } },
      scales: {
        x: { display: false },
        y: { beginAtZero: false, ticks: { maxTicksLimit: 5 } },
      },
    },
  });
};

// 4. LOGIC LOOPS
// --------------------------------------------------------------------------

const stepSimulation = () => {
  // 1. Calculate Gravity
  const dx = sunBody.position.x - earthBody.position.x;
  const dy = sunBody.position.y - earthBody.position.y;
  const dz = sunBody.position.z - earthBody.position.z;
  const r2 = dx * dx + dy * dy + dz * dz;
  const r = Math.sqrt(r2);

  // F = G * M1 * M2 / r^2
  const F = (CONSTANTS.G * CONSTANTS.sunMass * CONSTANTS.earthMass) / r2;

  // Direction: Earth -> Sun
  // Force vector components
  const fx = (F * dx) / r;
  const fy = (F * dy) / r;
  const fz = (F * dz) / r;

  earthBody.force.set(fx, fy, fz);

  // 2. Step World
  // Variable timestep support
  const dt = (1 / 60) * timeScale.value;
  // Cannon step(dt)
  world.step(dt);

  simTime += dt;

  return { r, v: earthBody.velocity.length() };
};

const updateVisuals = () => {
  // Sync Meshes
  earthMesh.position.copy(earthBody.position as any);

  // Trail
  // Using append and shift logic
  if (trailIdx < MAX_TRAIL_POINTS * 3) {
    trailPositions[trailIdx++] = earthBody.position.x;
    trailPositions[trailIdx++] = earthBody.position.y;
    trailPositions[trailIdx++] = earthBody.position.z;
    trailGeometry.setDrawRange(0, trailIdx / 3);
    trailGeometry.attributes.position.needsUpdate = true;
  } else {
    // Shift every frame (expensive but consistent loop)
    trailPositions.set(trailPositions.subarray(3));
    const i = (MAX_TRAIL_POINTS - 1) * 3;
    trailPositions[i] = earthBody.position.x;
    trailPositions[i + 1] = earthBody.position.y;
    trailPositions[i + 2] = earthBody.position.z;
    trailGeometry.attributes.position.needsUpdate = true;
  }

  // Vector
  if (showVectors.value) {
    vectorArrow.visible = true;
    vectorArrow.position.copy(earthBody.position as any);
    const v = new THREE.Vector3(
      earthBody.velocity.x,
      earthBody.velocity.y,
      earthBody.velocity.z
    );
    const len = v.length();
    vectorArrow.setDirection(v.normalize());
    // Scale: 30km/s -> 3e4 m/s. Need ~1000x of that = 0.2AU
    // 0.2AU = 0.3e11. 3e4 * 1e6 = 3e10.
    vectorArrow.setLength(len * 1e6);
  } else {
    vectorArrow.visible = false;
  }
};

const logFrame = (r: number, v: number) => {
  const now = Date.now();
  const log = {
    timestamp: now,
    t: simTime,
    t_fmt: simTime.toFixed(0),
    dist: r / CONSTANTS.AU,
    speed: v / 1000,
    x: earthBody.position.x,
    y: earthBody.position.y,
    z: earthBody.position.z,
    vx: earthBody.velocity.x,
    vy: earthBody.velocity.y,
    vz: earthBody.velocity.z,
  };
  logs.push(log);

  dataDisplay.t = log.t_fmt;
  dataDisplay.x = log.dist.toFixed(3);
  dataDisplay.y = (log.y / CONSTANTS.AU).toFixed(3);
  dataDisplay.z = (log.z / CONSTANTS.AU).toFixed(3);
  dataDisplay.v = log.speed.toFixed(3);
};

const updateChart = () => {
  if (!chartInstance) return;

  const now = Date.now();
  const cutoff = now - chartWindowSeconds.value * 1000;

  // Quick filter
  // Since logs accumulate, find the first index >= cutoff
  // Let's just filter for safety
  const visibleLogs = logs.filter((l) => l.timestamp >= cutoff);

  if (visibleLogs.length === 0) return;

  // Decimation: limit to ~300 points
  const step = Math.ceil(visibleLogs.length / 300) || 1;
  const decimatedNodes = [];
  for (let i = 0; i < visibleLogs.length; i += step) {
    decimatedNodes.push(visibleLogs[i]);
  }

  chartInstance.data.labels = decimatedNodes.map((l) => l.t_fmt);
  chartInstance.data.datasets[0].data = decimatedNodes.map((l) => l.dist);
  chartInstance.data.datasets[1].data = decimatedNodes.map((l) => l.speed);

  chartInstance.update("none");
};

const loop = () => {
  reqId = requestAnimationFrame(loop);

  if (controls) controls.update();
  if (renderer && scene && camera) renderer.render(scene, camera);

  if (isPaused.value) return;
  if (simTime > CONSTANTS.stopTime) {
    isPaused.value = true;
    return;
  }

  const { r, v } = stepSimulation();
  updateVisuals();
  logFrame(r, v);
  updateChart();
};

// 5. SIDEBAR & ACTIONS
// --------------------------------------------------------------------------

// Resizing
let isResizing = false;
const handleResizeDown = () => (isResizing = true);
const handleResizeUp = () => {
  if (isResizing) {
    isResizing = false;
    if (chartInstance) chartInstance.resize();
  }
};
const handleResizeMove = (e: MouseEvent) => {
  if (!isResizing) return;
  const newW = window.innerWidth - e.clientX;
  if (newW >= 260 && newW <= 600 && window.innerWidth - newW >= 300) {
    sidebarWidth.value = newW;
    nextTick(() => {
      if (viewport.value) {
        camera.aspect =
          viewport.value.clientWidth / viewport.value.clientHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(
          viewport.value.clientWidth,
          viewport.value.clientHeight
        );
      }
    });
  }
};

// Actions
const togglePause = () => (isPaused.value = !isPaused.value);

const restartSimulation = () => {
  isPaused.value = true;
  simTime = 0;
  logs = [];
  trailIdx = 0;
  trailPositions.fill(0);
  trailGeometry.setDrawRange(0, 0);

  if (chartInstance) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets[0].data = [];
    chartInstance.data.datasets[1].data = [];
    chartInstance.update();
  }

  initPhysics();
  Object.assign(dataDisplay, { t: "0", x: "0", y: "0", z: "0", v: "0" });
  isPaused.value = false;
};

const resetCharts = () => {
  if (chartInstance) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets.forEach((d) => (d.data = []));
    chartInstance.update();
  }
};

const exportCSV = (data: any[], filename: string) => {
  if (!data.length) return;
  const headers = Object.keys(data[0]).join(",");
  const rows = data.map((obj) => Object.values(obj).join(","));
  const csvContent =
    "data:text/csv;charset=utf-8," + [headers, ...rows].join("\n");
  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", filename);
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
};

const exportSimulationCSV = () => {
  if (logs.length) exportCSV(logs, `sim_data_${Date.now()}.csv`);
};

const exportChartCSV = () => {
  const now = Date.now();
  const cutoff = now - chartWindowSeconds.value * 1000;
  const visible = logs.filter((l) => l.timestamp >= cutoff);
  if (visible.length) exportCSV(visible, `chart_data_${Date.now()}.csv`);
};

// 6. LIFECYCLE
// --------------------------------------------------------------------------
onMounted(() => {
  if (import.meta.client) {
    initPhysics();
    initThree();
    initChart();
    loop();

    window.addEventListener("resize", () => {
      if (renderer && viewport.value) {
        camera.aspect =
          viewport.value.clientWidth / viewport.value.clientHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(
          viewport.value.clientWidth,
          viewport.value.clientHeight
        );
      }
    });
  }
});

onUnmounted(() => {
  if (import.meta.client) {
    cancelAnimationFrame(reqId);
    if (renderer) renderer.dispose();
    if (chartInstance) chartInstance.destroy();
  }
});
</script>

<style scoped>
.collapse-title,
.collapse-content {
  transition: all 0.2s;
}
.range-xs {
  height: 1rem;
}
</style>
