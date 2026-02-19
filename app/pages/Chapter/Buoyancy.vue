<template>
  <div
    class="flex h-full w-full bg-base-100 overflow-hidden"
    @mouseup="stopResize"
    @mousemove="onResize"
  >
    <!-- 3D Canvas Container -->
    <div
      ref="canvasContainer"
      class="flex-grow relative min-w-[300px] bg-base-300"
    >
      <canvas ref="canvasRef" class="w-full h-full block touch-none" />

      <!-- Overlay UI for Simulation Status -->
      <div
        class="absolute top-4 left-4 badge badge-lg opacity-80 backdrop-blur-sm"
        :class="isRunning ? 'badge-success' : 'badge-warning'"
      >
        {{ isRunning ? "RUNNING" : "PAUSED" }} (t = {{ time.toFixed(2) }}s)
      </div>

      <!-- Help Button (Top Right) -->
      <button
        @click="showHelpModal = true"
        class="absolute top-4 right-4 btn btn-circle btn-ghost btn-md z-10 text-primary-focus bg-base-100/50 backdrop-blur hover:bg-base-200 transition-all"
        title="Physics Explanation"
      >
        <Icon name="heroicons:question-mark-circle" class="text-3xl" />
      </button>

      <!-- Physics Help Modal -->
      <dialog class="modal" :class="{ 'modal-open': showHelpModal }">
        <div class="modal-box max-w-2xl bg-base-100 border border-base-300 text-base-content">
          <h3 class="font-bold text-2xl mb-4 flex items-center gap-2">
            <Icon name="heroicons:academic-cap" class="text-primary text-3xl" />
            Physics Concepts: Buoyancy & Archimedes' Principle
          </h3>

          <div class="space-y-4 text-sm leading-relaxed overflow-y-auto max-h-[70vh] pr-2">
            <section>
              <h4 class="font-bold text-lg text-secondary">1. Archimedes' Principle</h4>
              <p>
                Any object, wholly or partially immersed in a fluid, is buoyed up by a force equal to the weight of the fluid displaced by the object:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="F_b = \rho_f \cdot g \cdot V_{sub}" :inline="false" />
              </div>
              <p>
                where <MathLatex formula="\rho_f" /> is the fluid density and <MathLatex formula="V_{sub}" /> is the submerged volume.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">2. Equations of Motion</h4>
              <p>
                The net vertical force on the block is determined by gravity, buoyancy, and fluid drag:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="F_{net} = F_b - F_g + F_d" :inline="false" />
              </div>
              <ul class="list-disc ml-6 mt-2 space-y-1">
                <li><strong>Gravity:</strong> <MathLatex formula="F_g = m \cdot g = \rho_b \cdot V \cdot g" /></li>
                <li><strong>Buoyancy:</strong> <MathLatex formula="F_b = \rho_f \cdot g \cdot V_{sub}" /></li>
                <li><strong>Drag:</strong> <MathLatex formula="F_d = -c \cdot v" /> (Simplified linear drag for stability)</li>
              </ul>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">3. Floating vs. Sinking</h4>
              <ul class="list-disc ml-6 space-y-2">
                <li><strong>Floating:</strong> If <MathLatex formula="\rho_b < \rho_f" />, the object will oscillate and eventually settle at a depth where <MathLatex formula="F_b = F_g" />.</li>
                <li><strong>Sinking:</strong> If <MathLatex formula="\rho_b > \rho_f" />, the object will sink to the bottom as gravity exceeds the maximum possible buoyancy.</li>
              </ul>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">4. Controls Guide</h4>
              <ul class="list-disc ml-6 space-y-2">
                <li><strong>Fluid Density:</strong> Adjusting this changes the magnitude of the upward buoyant force.</li>
                <li><strong>Block Density:</strong> Adjusting this changes the weight of the object.</li>
                <li><strong>Drag Coeff:</strong> Higher values provide more damping, helping the system reach equilibrium faster.</li>
              </ul>
            </section>
          </div>

          <div class="modal-action mt-6">
            <button class="btn btn-primary" @click="showHelpModal = false">Got it!</button>
          </div>
        </div>
        <form method="dialog" class="modal-backdrop">
          <button @click="showHelpModal = false">close</button>
        </form>
      </dialog>
    </div>

    <!-- Resizer Handle -->
    <div
      class="w-2 cursor-col-resize bg-base-300 hover:bg-primary/50 transition-colors flex flex-col justify-center items-center select-none z-10"
      @mousedown="startResize"
    >
      <div class="h-8 w-1 bg-base-content/20 rounded-full"></div>
    </div>

    <!-- Sidebar Controls -->
    <div
      class="flex flex-col bg-base-200 border-l border-base-300 h-full overflow-y-auto overflow-x-hidden shadow-xl z-20"
      :style="{ width: sidebarWidth + 'px' }"
    >
      <!-- Panel 1: Controls -->
      <div
        class="collapse collapse-arrow bg-base-100 rounded-none border-b border-base-300"
      >
        <input type="radio" name="sidebar-accordion" checked />
        <div class="collapse-title text-xl font-medium flex items-center justify-between pr-12">
          <div class="flex items-center gap-2">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              class="h-6 w-6"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M12 6V4m0 2a2 2 0 100 4m0-4a2 2 0 110 4m-6 8a2 2 0 100-4m0 4a2 2 0 110-4m0 4v2m0-6V4m6 6v10m6-2a2 2 0 100-4m0 4a2 2 0 110-4m0 4v2m0-6V4"
              />
            </svg>
            <span>Controls</span>
          </div>
          <UButton
            icon="i-lucide-book-open"
            color="neutral"
            variant="ghost"
            size="xs"
            @click.stop="showHelpModal = true"
          />
        </div>
        <div class="collapse-content space-y-4 pt-2">
          <!-- Simulation Control Buttons -->
          <div class="grid grid-cols-2 gap-2 mb-4">
            <button class="btn btn-primary" @click="toggleSimulation">
              {{ isRunning ? "Pause" : "Resume" }}
            </button>
            <button class="btn btn-warning" @click="resetSimulation">
              Restart
            </button>
          </div>

          <div class="divider text-sm font-bold">Fluid Properties</div>

          <!-- Fluid Density -->
          <div class="form-control w-full">
            <label class="label">
              <span class="label-text">Fluid Density (ρf) [kg/m³]</span>
              <span class="label-text-alt font-mono">{{
                settings.fluidDensity
              }}</span>
            </label>
            <input
              type="range"
              min="500"
              max="2000"
              step="10"
              v-model.number="settings.fluidDensity"
              class="range range-xs range-info"
              @change="resetSimulation"
            />
          </div>

          <div class="divider text-sm font-bold">Block Properties</div>

          <!-- Block Density -->
          <div class="form-control w-full">
            <label class="label">
              <span class="label-text">Block Density (ρb) [kg/m³]</span>
              <span class="label-text-alt font-mono">{{
                settings.blockDensity
              }}</span>
            </label>
            <input
              type="range"
              min="100"
              max="2500"
              step="10"
              v-model.number="settings.blockDensity"
              class="range range-xs range-secondary"
              @change="resetSimulation"
            />
          </div>

          <!-- Initial Height -->
          <div class="form-control w-full">
            <label class="label">
              <span class="label-text">Initial Height (y0) [m]</span>
              <span class="label-text-alt font-mono">{{
                settings.initialHeight
              }}</span>
            </label>
            <input
              type="range"
              min="-2"
              max="5"
              step="0.1"
              v-model.number="settings.initialHeight"
              class="range range-xs"
              @change="resetSimulation"
            />
          </div>

          <!-- Drag Coefficient -->
          <div class="form-control w-full">
            <label class="label">
              <span class="label-text">Drag Coeff (Stability)</span>
              <span class="label-text-alt font-mono">{{
                settings.dragCoeff
              }}</span>
            </label>
            <input
              type="range"
              min="100"
              max="1000"
              step="100"
              v-model.number="settings.dragCoeff"
              class="range range-xs range-accent"
              @change="resetSimulation"
            />
          </div>

          <div class="text-xs text-base-content/50 mt-2 italic">
            Note: Changing parameters automatically restarts.
          </div>
        </div>
      </div>

      <!-- Panel 2: Live Data -->
      <div
        class="collapse collapse-arrow bg-base-100 rounded-none border-b border-base-300"
      >
        <input type="radio" name="sidebar-accordion" />
        <div class="collapse-title text-xl font-medium flex items-center gap-2">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            class="h-6 w-6"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z"
            />
          </svg>
          Live Data
        </div>
        <div class="collapse-content pt-2">
          <!-- Main Stats -->
          <div class="stats stats-vertical w-full shadow-sm bg-base-200 mb-4">
            <div class="stat p-2">
              <div class="stat-title text-xs">Vertical Position (y)</div>
              <div class="stat-value text-lg font-mono">
                {{ displayData.position.toFixed(3) }}
              </div>
              <div class="stat-desc">m</div>
            </div>
            <div class="stat p-2">
              <div class="stat-title text-xs">Velocity (v)</div>
              <div class="stat-value text-lg font-mono">
                {{ displayData.velocity.toFixed(3) }}
              </div>
              <div class="stat-desc">m/s</div>
            </div>
          </div>

          <!-- Forces Table -->
          <div class="overflow-x-auto">
            <table class="table table-xs w-full">
              <thead>
                <tr>
                  <th>Force</th>
                  <th>Value (N)</th>
                </tr>
              </thead>
              <tbody>
                <tr class="text-error">
                  <td>Gravity (Fg)</td>
                  <td class="font-mono">
                    {{ displayData.fg.toFixed(2) }}
                  </td>
                </tr>
                <tr class="text-info">
                  <td>Buoyancy (Fb)</td>
                  <td class="font-mono">
                    {{ displayData.fb.toFixed(2) }}
                  </td>
                </tr>
                <tr class="text-accent">
                  <td>Drag (Fd)</td>
                  <td class="font-mono">
                    {{ displayData.fd.toFixed(2) }}
                  </td>
                </tr>
                <tr class="font-bold">
                  <td>Net Force</td>
                  <td class="font-mono">
                    {{ displayData.fnet.toFixed(2) }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <button class="btn btn-outline btn-sm w-full mt-4" @click="exportCSV">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              class="h-4 w-4 mr-2"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4"
              />
            </svg>
            Export Log (CSV)
          </button>
        </div>
      </div>

      <!-- Panel 3: Charts -->
      <div
        class="collapse collapse-arrow bg-base-100 rounded-none border-b border-base-300"
      >
        <input type="radio" name="sidebar-accordion" />
        <div class="collapse-title text-xl font-medium flex items-center gap-2">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            class="h-6 w-6"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M7 12l3-3 3 3 4-4M8 21l4-4 4 4M3 4h18M4 4h16v12a1 1 0 01-1 1H5a1 1 0 01-1-1V4z"
            />
          </svg>
          Charts
        </div>
        <div class="collapse-content pt-2">
          <div class="form-control w-full max-w-xs">
            <label class="label">
              <span class="label-text">Variable</span>
            </label>
            <select
              class="select select-bordered select-sm"
              v-model="chartConfig.variable"
            >
              <option value="position">Position vs Time</option>
              <option value="velocity">Velocity vs Time</option>
              <option value="force">Forces vs Time</option>
            </select>
          </div>

          <div class="mt-4 bg-white rounded-lg p-2 h-48 w-full relative">
            <canvas ref="chartCanvasRef"></canvas>
          </div>

          <div class="grid grid-cols-2 gap-2 mt-4">
            <button class="btn btn-primary btn-sm" @click="updateChart">
              Plot Chart
            </button>
            <button class="btn btn-ghost btn-sm" @click="exportChartCSV">
              Export Data
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  ref,
  onMounted,
  onUnmounted,
  reactive,
  nextTick,
  shallowRef,
} from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { Water } from "three/examples/jsm/objects/Water.js";
import { Chart, registerables } from "chart.js";

// Register Chart.js components
if (import.meta.client) {
  Chart.register(...registerables);
}

// --- Constants ---
const GRAVITY = 9.81;
const BLOCK_SIZE = 1;

// --- State Management ---
const canvasRef = ref<HTMLCanvasElement | null>(null);
const chartCanvasRef = ref<HTMLCanvasElement | null>(null);
const sidebarWidth = ref(320);
const isRunning = ref(false);
const time = ref(0);
const showHelpModal = ref(true); // Auto-show on first load

// Drag Resizing Logic
const isDragging = ref(false);
function startResize() {
  isDragging.value = true;
}
function stopResize() {
  isDragging.value = false;
}
function onResize(e: MouseEvent) {
  if (!isDragging.value) return;
  const newWidth = document.body.clientWidth - e.clientX;
  if (newWidth > 260 && newWidth < Math.min(600, window.innerWidth - 300)) {
    sidebarWidth.value = newWidth;
    nextTick(() => window.dispatchEvent(new Event("resize")));
  }
}

// System Settings
const settings = reactive({
  fluidDensity: 1000,
  blockDensity: 500,
  initialHeight: 2,
  dragCoeff: 600,
});

// Physics State (Custom)
const physicsState = reactive({
  position: 0,
  velocity: 0,
  acceleration: 0,
  mass: 0,
});

// Live Data Display
const displayData = reactive({
  position: 0,
  velocity: 0,
  fg: 0,
  fb: 0,
  fd: 0,
  fnet: 0,
});

// Logging System
interface LogEntry {
  time: number;
  position: number;
  velocity: number;
  fg: number;
  fb: number;
  fd: number;
  fnet: number;
}
let dataLogs: LogEntry[] = [];

// Chart State
let chartInstance: Chart | null = null;
const chartConfig = reactive({
  variable: "position",
});

// --- Three.js Variables ---
const scene = shallowRef<THREE.Scene | null>(null);
const camera = shallowRef<THREE.PerspectiveCamera | null>(null);
const renderer = shallowRef<THREE.WebGLRenderer | null>(null);
const controls = shallowRef<OrbitControls | null>(null);
const blockMesh = shallowRef<THREE.Mesh | null>(null);
const waterObj = shallowRef<Water | null>(null);
const arrowFg = shallowRef<THREE.ArrowHelper | null>(null);
const arrowFb = shallowRef<THREE.ArrowHelper | null>(null);

let animationFrameId: number;

// --- Initialization ---

onMounted(() => {
  if (import.meta.client) {
    initThree();
    initChart();
    resetSimulation();
    renderLoop();
    window.addEventListener("resize", handleResize);
  }
});

onUnmounted(() => {
  if (import.meta.client) {
    cancelAnimationFrame(animationFrameId);
    window.removeEventListener("resize", handleResize);
    renderer.value?.dispose();
    chartInstance?.destroy();
  }
});

function initThree() {
  if (!canvasRef.value) return;

  // Scene
  scene.value = new THREE.Scene();
  scene.value.background = new THREE.Color(0x202025);

  // Camera
  camera.value = new THREE.PerspectiveCamera(
    45,
    (window.innerWidth - sidebarWidth.value) / window.innerHeight,
    0.1,
    100,
  );
  camera.value.position.set(4, 4, 6);
  camera.value.lookAt(0, 0, 0);

  // Renderer
  renderer.value = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.value.shadowMap.enabled = true;
  renderer.value.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.value.setSize(
    window.innerWidth - sidebarWidth.value,
    window.innerHeight,
  );

  // Controls
  controls.value = new OrbitControls(camera.value, renderer.value.domElement);
  controls.value.enableDamping = true;

  // Lighting
  const ambLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.value.add(ambLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
  dirLight.position.set(5, 10, 5);
  dirLight.castShadow = true;
  scene.value.add(dirLight);

  // --- Water ---
  const waterGeometry = new THREE.PlaneGeometry(4, 4);
  const canvas = document.createElement("canvas");
  canvas.width = 512;
  canvas.height = 512;
  const ctx = canvas.getContext("2d");
  if (ctx) {
    ctx.fillStyle = "#8080ff";
    ctx.fillRect(0, 0, 512, 512);
    for (let i = 0; i < 20000; i++) {
      ctx.fillStyle = Math.random() > 0.5 ? "#8888ff" : "#7777ff";
      ctx.fillRect(Math.random() * 512, Math.random() * 512, 2, 2);
    }
  }
  const texture = new THREE.CanvasTexture(canvas);
  texture.wrapS = THREE.RepeatWrapping;
  texture.wrapT = THREE.RepeatWrapping;

  waterObj.value = new Water(waterGeometry, {
    textureWidth: 512,
    textureHeight: 512,
    waterNormals: texture,
    alpha: 0.8,
    sunDirection: new THREE.Vector3(),
    sunColor: 0xffffff,
    waterColor: 0x001e0f,
    distortionScale: 3.7,
    fog: scene.value.fog !== undefined,
  });

  waterObj.value.rotation.x = -Math.PI / 2;
  scene.value.add(waterObj.value);

  // --- Tank ---
  const tankGeo = new THREE.BoxGeometry(4.1, 3, 4.1);
  const glassMat = new THREE.MeshPhysicalMaterial({
    color: 0xffffff,
    metalness: 0,
    roughness: 0,
    transmission: 0.95,
    transparent: true,
    opacity: 0.3,
    side: THREE.DoubleSide,
  });
  const tank = new THREE.Mesh(tankGeo, glassMat);
  tank.position.y = -0.5;
  scene.value.add(tank);

  // --- Block ---
  const blockGeo = new THREE.BoxGeometry(BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE);
  const blockMat = new THREE.MeshStandardMaterial({
    color: 0xaa5522,
    roughness: 0.4,
    metalness: 0.1,
  });
  blockMesh.value = new THREE.Mesh(blockGeo, blockMat);
  blockMesh.value.castShadow = true;
  blockMesh.value.receiveShadow = true;
  scene.value.add(blockMesh.value);

  // --- Arrows ---
  arrowFg.value = new THREE.ArrowHelper(
    new THREE.Vector3(0, -1, 0),
    new THREE.Vector3(0, 0, 0),
    1,
    0xff4444,
  );
  scene.value.add(arrowFg.value);

  arrowFb.value = new THREE.ArrowHelper(
    new THREE.Vector3(0, 1, 0),
    new THREE.Vector3(0, 0, 0),
    1,
    0x4444ff,
  );
  scene.value.add(arrowFb.value);
}

function resetSimulation() {
  isRunning.value = false;
  time.value = 0;
  dataLogs = [];
  physicsState.position = settings.initialHeight;
  physicsState.velocity = 0;
  physicsState.acceleration = 0;
  physicsState.mass = settings.blockDensity * (BLOCK_SIZE ** 3);
  updateVisuals();
  updateDisplayData();
}

function stepSimulation() {
  const dt = 1 / 60;
  const Fg = -physicsState.mass * GRAVITY;
  const yBottom = physicsState.position - BLOCK_SIZE / 2;
  const yTop = physicsState.position + BLOCK_SIZE / 2;
  let submergedFactor = 0;
  if (yTop < 0) submergedFactor = 1;
  else if (yBottom > 0) submergedFactor = 0;
  else submergedFactor = -yBottom / BLOCK_SIZE;
  submergedFactor = Math.max(0, Math.min(1, submergedFactor));
  const Vsub = submergedFactor * (BLOCK_SIZE ** 3);
  const Fb = settings.fluidDensity * GRAVITY * Vsub;
  const Fd = -settings.dragCoeff * physicsState.velocity;
  const Fnet = Fg + Fb + Fd;
  const acc = Fnet / physicsState.mass;
  physicsState.velocity += acc * dt;
  physicsState.position += physicsState.velocity * dt;
  physicsState.acceleration = acc;
  time.value += dt;
  displayData.position = physicsState.position;
  displayData.velocity = physicsState.velocity;
  displayData.fg = Math.abs(Fg);
  displayData.fb = Fb;
  displayData.fd = Fd;
  displayData.fnet = Fnet;
  updateVisuals();
  logData();
}

function updateDisplayData() {
  displayData.position = physicsState.position;
  displayData.velocity = physicsState.velocity;
  displayData.fg = Math.abs(-physicsState.mass * GRAVITY);
  displayData.fb = 0;
  displayData.fd = 0;
  displayData.fnet = 0;
}

function updateVisuals() {
  if (!blockMesh.value) return;
  blockMesh.value.position.y = physicsState.position;
  if (arrowFg.value) {
    arrowFg.value.position.set(0, physicsState.position, 0);
    const fgLen = Math.min((physicsState.mass * GRAVITY) / 1000, 2) + 0.5;
    arrowFg.value.setLength(fgLen);
  }
  if (arrowFb.value) {
    arrowFb.value.position.set(0.6, physicsState.position, 0);
    const fbLen = Math.min(displayData.fb / 1000, 2) + 0.5;
    arrowFb.value.visible = displayData.fb > 1;
    if (arrowFb.value.visible) arrowFb.value.setLength(fbLen);
  }
}

function logData() {
  dataLogs.push({
    time: time.value,
    position: displayData.position,
    velocity: displayData.velocity,
    fg: displayData.fg,
    fb: displayData.fb,
    fd: displayData.fd,
    fnet: displayData.fnet,
  });
}

function initChart() {
  if (!chartCanvasRef.value) return;
  const ctx = chartCanvasRef.value.getContext("2d");
  if (!ctx) return;
  chartInstance = new Chart(ctx, {
    type: "line",
    data: {
      labels: [],
      datasets: [{
        label: "Data",
        data: [],
        borderColor: "rgb(75, 192, 192)",
        borderWidth: 2,
        pointRadius: 0,
        tension: 0.1,
      }],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: false,
      scales: { x: { title: { display: true, text: "Time (s)" }, ticks: { maxTicksLimit: 10 } } },
    },
  });
}

function updateChart() {
  if (!chartInstance) return;
  const step = Math.ceil(dataLogs.length / 500) || 1;
  const plottedData = dataLogs.filter((_, i) => i % step === 0);
  const labels = plottedData.map((l) => l.time.toFixed(2));
  let data: number[] = [];
  let label = "";
  if (chartConfig.variable === "position") { data = plottedData.map((l) => l.position); label = "Position (y)"; }
  else if (chartConfig.variable === "velocity") { data = plottedData.map((l) => l.velocity); label = "Velocity (v)"; }
  else if (chartConfig.variable === "force") { data = plottedData.map((l) => l.fnet); label = "Net Force"; }
  chartInstance.data.labels = labels;
  chartInstance.data.datasets[0].data = data;
  chartInstance.data.datasets[0].label = label;
  chartInstance.update();
}

function exportCSV() {
  const headers = ["Time", "Position", "Velocity", "Fg", "Fb", "Fd", "Fnet"];
  const rows = dataLogs.map((l) =>
    [l.time.toFixed(3), l.position.toFixed(3), l.velocity.toFixed(3), l.fg.toFixed(2), l.fb.toFixed(2), l.fd.toFixed(2), l.fnet.toFixed(2)].join(",")
  );
  const csvContent = "data:text/csv;charset=utf-8," + [headers, ...rows].join("\n");
  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", "buoyancy_simulation_log.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function exportChartCSV() { exportCSV(); }

function renderLoop() {
  animationFrameId = requestAnimationFrame(renderLoop);
  if (isRunning.value) stepSimulation();
  if (waterObj.value && waterObj.value.material.uniforms) waterObj.value.material.uniforms["time"].value += 1.0 / 60.0;
  if (controls.value) controls.value.update();
  if (renderer.value && scene.value && camera.value) renderer.value.render(scene.value, camera.value);
}

function toggleSimulation() { isRunning.value = !isRunning.value; }

function handleResize() {
  if (!renderer.value || !camera.value) return;
  const w = window.innerWidth - sidebarWidth.value;
  const h = window.innerHeight;
  camera.value.aspect = w / h;
  camera.value.updateProjectionMatrix();
  renderer.value.setSize(w, h);
}
</script>

<style scoped></style>
