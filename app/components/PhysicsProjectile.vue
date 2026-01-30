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
              d="M12 6V4m0 2a2 2 0 100 4m0-4a2 2 0 110 4m-6 8a2 2 0 100-4m0 4a2 2 0 110-4m0 4v2m0-6V4m6 6v10m6-2a2 2 0 100-4m0 4a2 2 0 110-4m0 4v2m0-6V4"
            />
          </svg>
          Controls
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

          <div class="divider text-sm font-bold">System Properties</div>

          <div class="alert alert-info shadow-sm text-xs py-2">
            <div>
              <svg
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
                class="stroke-current flex-shrink-0 w-6 h-6"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
                ></path>
              </svg>
              <span>Perfect Elastic Collision (e=1.0)</span>
            </div>
          </div>

          <!-- Gravity Toggle -->
          <div class="form-control">
            <label class="label cursor-pointer">
              <span class="label-text">Gravity (Earth 9.81)</span>
              <input
                type="checkbox"
                v-model="settings.useGravity"
                class="toggle toggle-sm toggle-secondary"
              />
            </label>
          </div>

          <!-- Sphere 1 -->
          <div class="divider text-sm font-bold text-error">Sphere 1 (Red)</div>

          <div class="form-control w-full">
            <label class="label"
              ><span class="label-text">Mass (m1) [kg]</span></label
            >
            <input
              type="number"
              v-model.number="settings.body1.mass"
              class="input input-bordered input-sm"
              step="0.1"
              min="0.1"
              @change="resetSimulation"
            />
          </div>

          <div class="grid grid-cols-3 gap-2">
            <div class="form-control">
              <label class="label p-0"
                ><span class="label-text-alt">Vx</span></label
              >
              <input
                type="number"
                v-model.number="settings.body1.velocity.x"
                class="input input-bordered input-xs px-1"
                @change="resetSimulation"
              />
            </div>
            <div class="form-control">
              <label class="label p-0"
                ><span class="label-text-alt">Vy</span></label
              >
              <input
                type="number"
                v-model.number="settings.body1.velocity.y"
                class="input input-bordered input-xs px-1"
                @change="resetSimulation"
              />
            </div>
            <div class="form-control">
              <label class="label p-0"
                ><span class="label-text-alt">Vz</span></label
              >
              <input
                type="number"
                v-model.number="settings.body1.velocity.z"
                class="input input-bordered input-xs px-1"
                @change="resetSimulation"
              />
            </div>
          </div>

          <!-- Sphere 2 -->
          <div class="divider text-sm font-bold text-info">Sphere 2 (Blue)</div>

          <div class="form-control w-full">
            <label class="label"
              ><span class="label-text">Mass (m2) [kg]</span></label
            >
            <input
              type="number"
              v-model.number="settings.body2.mass"
              class="input input-bordered input-sm"
              step="0.1"
              min="0.1"
              @change="resetSimulation"
            />
          </div>

          <div class="grid grid-cols-3 gap-2">
            <div class="form-control">
              <label class="label p-0"
                ><span class="label-text-alt">Vx</span></label
              >
              <input
                type="number"
                v-model.number="settings.body2.velocity.x"
                class="input input-bordered input-xs px-1"
                @change="resetSimulation"
              />
            </div>
            <div class="form-control">
              <label class="label p-0"
                ><span class="label-text-alt">Vy</span></label
              >
              <input
                type="number"
                v-model.number="settings.body2.velocity.y"
                class="input input-bordered input-xs px-1"
                @change="resetSimulation"
              />
            </div>
            <div class="form-control">
              <label class="label p-0"
                ><span class="label-text-alt">Vz</span></label
              >
              <input
                type="number"
                v-model.number="settings.body2.velocity.z"
                class="input input-bordered input-xs px-1"
                @change="resetSimulation"
              />
            </div>
          </div>

          <div class="text-xs text-base-content/50 mt-2 italic">
            Note: Changing simulation parameters automatically restarts.
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
          <!-- System Totals -->
          <div class="stats stats-vertical w-full shadow-sm bg-base-200 mb-4">
            <div class="stat p-2">
              <div class="stat-title text-xs">Total Momentum (P)</div>
              <div class="stat-value text-lg font-mono">
                {{ displayData.totalMomentum.toFixed(3) }}
              </div>
              <div class="stat-desc">kg·m/s</div>
            </div>
            <div class="stat p-2">
              <div class="stat-title text-xs">Total Kinetic Energy (K)</div>
              <div class="stat-value text-lg font-mono">
                {{ displayData.totalEnergy.toFixed(3) }}
              </div>
              <div class="stat-desc">Joules</div>
            </div>
          </div>

          <!-- Velocities -->
          <div class="overflow-x-auto">
            <table class="table table-xs w-full">
              <thead>
                <tr>
                  <th>Obj</th>
                  <th>Vel (m/s)</th>
                  <th>P (kg·m/s)</th>
                </tr>
              </thead>
              <tbody>
                <tr class="text-error">
                  <td>S1</td>
                  <td class="font-mono">{{ displayData.v1.toFixed(2) }}</td>
                  <td class="font-mono">{{ displayData.p1.toFixed(2) }}</td>
                </tr>
                <tr class="text-info">
                  <td>S2</td>
                  <td class="font-mono">{{ displayData.v2.toFixed(2) }}</td>
                  <td class="font-mono">{{ displayData.p2.toFixed(2) }}</td>
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
              <option value="momentum">Total Momentum</option>
              <option value="energy">Total Kinetic Energy</option>
              <option value="v1">Velocity 1 (Mag)</option>
              <option value="v2">Velocity 2 (Mag)</option>
              <option value="pos_x">Position Separation (X)</option>
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
import Chart from "chart.js/auto";

// --- Types & Interfaces ---
interface Vector3 {
  x: number;
  y: number;
  z: number;
}
interface PhysicsBody {
  mass: number;
  radius: number;
  position: Vector3;
  velocity: Vector3;
}

// --- State Management ---
const canvasRef = ref<HTMLCanvasElement | null>(null);
const chartCanvasRef = ref<HTMLCanvasElement | null>(null);
const sidebarWidth = ref(320);
const isRunning = ref(false);
const time = ref(0);

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
  useGravity: false,
  body1: {
    mass: 1.0,
    velocity: { x: 5, y: 0, z: 0 },
    radius: 1,
    color: 0xff4444,
  },
  body2: {
    mass: 1.0,
    velocity: { x: -5, y: 0, z: 0 },
    radius: 1,
    color: 0x4444ff,
  },
});

// Physics State (Custom)
const bodies = shallowRef<PhysicsBody[]>([
  {
    mass: 1,
    radius: 1,
    position: { x: 0, y: 0, z: 0 },
    velocity: { x: 0, y: 0, z: 0 },
  },
  {
    mass: 1,
    radius: 1,
    position: { x: 0, y: 0, z: 0 },
    velocity: { x: 0, y: 0, z: 0 },
  },
]);

// Live Data Display
const displayData = reactive({
  totalMomentum: 0,
  totalEnergy: 0,
  v1: 0,
  v2: 0,
  p1: 0,
  p2: 0,
});

// Logging System
interface LogEntry {
  time: number;
  p_tot: number;
  k_tot: number;
  v1_mag: number;
  v2_mag: number;
  pos1_x: number;
  pos2_x: number;
}
let dataLogs: LogEntry[] = [];

// Chart State
let chartInstance: Chart | null = null;
const chartConfig = reactive({
  variable: "momentum",
});

// --- Three.js Variables ---
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let mBody1: THREE.Mesh, mBody2: THREE.Mesh;
let animationFrameId: number;

// --- Initialization ---

onMounted(() => {
  if (process.client) {
    initThree();
    initChart();
    resetSimulation();
    renderLoop();
    window.addEventListener("resize", handleResize);
  }
});

onUnmounted(() => {
  if (process.client) {
    cancelAnimationFrame(animationFrameId);
    window.removeEventListener("resize", handleResize);
    renderer?.dispose();
    chartInstance?.destroy();
  }
});

function initThree() {
  if (!canvasRef.value) return;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x202025);
  // Grid
  const gridHelper = new THREE.GridHelper(50, 50, 0x444444, 0x222222);
  scene.add(gridHelper);

  // Camera
  camera = new THREE.PerspectiveCamera(
    45,
    (window.innerWidth - sidebarWidth.value) / window.innerHeight,
    0.1,
    1000,
  );
  camera.position.set(10, 10, 20);
  camera.lookAt(0, 0, 0);

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.setSize(window.innerWidth - sidebarWidth.value, window.innerHeight);

  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  // Lighting
  const ambLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.add(ambLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(10, 20, 10);
  dirLight.castShadow = true;
  dirLight.shadow.mapSize.width = 2048;
  dirLight.shadow.mapSize.height = 2048;
  scene.add(dirLight);

  // Mesh 1 (Red)
  const geo1 = new THREE.SphereGeometry(settings.body1.radius, 32, 32);
  const mat1 = new THREE.MeshStandardMaterial({
    color: settings.body1.color,
    roughness: 0.1,
    metalness: 0.1,
  });
  mBody1 = new THREE.Mesh(geo1, mat1);
  mBody1.castShadow = true;
  mBody1.receiveShadow = true;
  scene.add(mBody1);

  // Mesh 2 (Blue)
  const geo2 = new THREE.SphereGeometry(settings.body2.radius, 32, 32);
  const mat2 = new THREE.MeshStandardMaterial({
    color: settings.body2.color,
    roughness: 0.1,
    metalness: 0.1,
  });
  mBody2 = new THREE.Mesh(geo2, mat2);
  mBody2.castShadow = true;
  mBody2.receiveShadow = true;
  scene.add(mBody2);
}

function resetSimulation() {
  isRunning.value = false;
  time.value = 0;
  dataLogs = [];

  // Initial Positions
  // Starting apart by 10 units, ensuring no overlap
  const startDist = 5;

  // Clone settings to physics state
  bodies.value[0] = {
    mass: settings.body1.mass,
    radius: settings.body1.radius,
    position: { x: -startDist, y: settings.body1.radius, z: 0 },
    velocity: { ...settings.body1.velocity },
  };

  bodies.value[1] = {
    mass: settings.body2.mass,
    radius: settings.body2.radius,
    position: { x: startDist, y: settings.body2.radius, z: 0 },
    velocity: { ...settings.body2.velocity },
  };

  updateVisuals();
  updateDisplayData();
}

// Custom Physics Step
function stepSimulation() {
  const dt = 1 / 60;
  const gravity = settings.useGravity ? -9.81 : 0;

  const b1 = bodies.value[0];
  const b2 = bodies.value[1];

  // 1. Update Velocities with Gravity
  if (settings.useGravity) {
    // Only apply gravity if above ground
    if (b1.position.y > b1.radius) b1.velocity.y += gravity * dt;
    if (b2.position.y > b2.radius) b2.velocity.y += gravity * dt;
  }

  // 2. Update Positions
  b1.position.x += b1.velocity.x * dt;
  b1.position.y += b1.velocity.y * dt;
  b1.position.z += b1.velocity.z * dt;

  b2.position.x += b2.velocity.x * dt;
  b2.position.y += b2.velocity.y * dt;
  b2.position.z += b2.velocity.z * dt;

  // 3. Ground Collision (Simple floor at y=0)
  if (b1.position.y < b1.radius) {
    b1.position.y = b1.radius;
    b1.velocity.y *= -1; // Elastic bounce
  }
  if (b2.position.y < b2.radius) {
    b2.position.y = b2.radius;
    b2.velocity.y *= -1; // Elastic bounce
  }

  // 4. Sphere-Sphere Collision Check
  const dx = b2.position.x - b1.position.x;
  const dy = b2.position.y - b1.position.y;
  const dz = b2.position.z - b1.position.z;
  const distSq = dx * dx + dy * dy + dz * dz;
  const minDist = b1.radius + b2.radius;

  if (distSq < minDist * minDist) {
    resolveCollision(b1, b2);
  }

  time.value += dt;
  updateVisuals();
  updateDisplayData();
  logData();
}

// Elastic Collision Formula Resolution
function resolveCollision(b1: PhysicsBody, b2: PhysicsBody) {
  // Vector math for generalized 3D elastic collision
  // Formula: v1' = v1 - (2*m2 / (m1+m2)) * <v1-v2, n> * n
  // Where n is the normalized collision normal vector (pos2 - pos1)

  const m1 = b1.mass;
  const m2 = b2.mass;

  // Normal vector n = (pos1 - pos2) / |pos1 - pos2|
  // Note: Using (x1 - x2) convention
  const n = {
    x: b1.position.x - b2.position.x,
    y: b1.position.y - b2.position.y,
    z: b1.position.z - b2.position.z,
  };

  const nMagSq = n.x * n.x + n.y * n.y + n.z * n.z;
  const nMag = Math.sqrt(nMagSq);

  if (nMag === 0) return; // Exact overlap edge case

  // Separate bodies to prevent sticking (move them out along normal)
  const overlap = b1.radius + b2.radius - nMag;
  const correctionRatio = overlap / nMag / 2;

  b1.position.x += n.x * correctionRatio;
  b1.position.y += n.y * correctionRatio;
  b1.position.z += n.z * correctionRatio;

  b2.position.x -= n.x * correctionRatio;
  b2.position.y -= n.y * correctionRatio;
  b2.position.z -= n.z * correctionRatio;

  // Relative velocity dot Product <v1-v2, x1-x2>
  const vDiff = {
    x: b1.velocity.x - b2.velocity.x,
    y: b1.velocity.y - b2.velocity.y,
    z: b1.velocity.z - b2.velocity.z,
  };

  const dotProd = vDiff.x * n.x + vDiff.y * n.y + vDiff.z * n.z;

  // Scalar factor for impulse scaling
  // Based on formula: v1_new = v1 - [2m2/(m1+m2)] * [dot(v1-v2, x1-x2) / |x1-x2|^2] * (x1-x2)

  const factor = (2 * dotProd) / ((m1 + m2) * nMagSq);

  // Update velocities
  b1.velocity.x -= factor * m2 * n.x;
  b1.velocity.y -= factor * m2 * n.y;
  b1.velocity.z -= factor * m2 * n.z;

  b2.velocity.x += factor * m1 * n.x;
  b2.velocity.y += factor * m1 * n.y;
  b2.velocity.z += factor * m1 * n.z;
}

function updateVisuals() {
  const b1 = bodies.value[0];
  const b2 = bodies.value[1];

  mBody1.position.set(b1.position.x, b1.position.y, b1.position.z);
  mBody2.position.set(b2.position.x, b2.position.y, b2.position.z);
}

function updateDisplayData() {
  const b1 = bodies.value[0];
  const b2 = bodies.value[1];

  const v1Mag = Math.sqrt(
    b1.velocity.x ** 2 + b1.velocity.y ** 2 + b1.velocity.z ** 2,
  );
  const v2Mag = Math.sqrt(
    b2.velocity.x ** 2 + b2.velocity.y ** 2 + b2.velocity.z ** 2,
  );

  const K1 = 0.5 * b1.mass * v1Mag ** 2;
  const K2 = 0.5 * b2.mass * v2Mag ** 2;

  // Momentum P = mv
  // P_tot magnitude requires vector addition
  const p1Vec = {
    x: b1.velocity.x * b1.mass,
    y: b1.velocity.y * b1.mass,
    z: b1.velocity.z * b1.mass,
  };
  const p2Vec = {
    x: b2.velocity.x * b2.mass,
    y: b2.velocity.y * b2.mass,
    z: b2.velocity.z * b2.mass,
  };

  const pTotVec = {
    x: p1Vec.x + p2Vec.x,
    y: p1Vec.y + p2Vec.y,
    z: p1Vec.z + p2Vec.z,
  };
  const PTotMag = Math.sqrt(pTotVec.x ** 2 + pTotVec.y ** 2 + pTotVec.z ** 2);

  displayData.v1 = v1Mag;
  displayData.v2 = v2Mag;
  displayData.p1 = v1Mag * b1.mass;
  displayData.p2 = v2Mag * b2.mass;
  displayData.totalEnergy = K1 + K2;
  displayData.totalMomentum = PTotMag;
}

function logData() {
  dataLogs.push({
    time: time.value,
    p_tot: displayData.totalMomentum,
    k_tot: displayData.totalEnergy,
    v1_mag: displayData.v1,
    v2_mag: displayData.v2,
    pos1_x: bodies.value[0].position.x,
    pos2_x: bodies.value[1].position.x,
  });
}

function renderLoop() {
  animationFrameId = requestAnimationFrame(renderLoop);

  if (isRunning.value) {
    stepSimulation();
  }

  if (controls) controls.update();
  if (renderer && scene && camera) renderer.render(scene, camera);
}

function toggleSimulation() {
  isRunning.value = !isRunning.value;
}

function handleResize() {
  if (!renderer || !camera) return;
  const w = window.innerWidth - sidebarWidth.value;
  const h = window.innerHeight;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
}

// --- Charting ---

function initChart() {
  if (!chartCanvasRef.value) return;

  chartInstance = new Chart(chartCanvasRef.value, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        {
          label: "Metric",
          data: [],
          borderColor: "#570df8",
          tension: 0.1,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: false,
      scales: {
        x: { title: { display: true, text: "Time (s)" } },
        y: { title: { display: true, text: "Value" } },
      },
    },
  });
}

function updateChart() {
  if (!chartInstance) return;

  const keyMap: Record<string, keyof LogEntry> = {
    momentum: "p_tot",
    energy: "k_tot",
    v1: "v1_mag",
    v2: "v2_mag",
    pos_x: "pos1_x",
  };

  const key = keyMap[chartConfig.variable] || "p_tot";
  const labelMap: Record<string, string> = {
    momentum: "Total Momentum (kg·m/s)",
    energy: "Total Kinetic Energy (J)",
    v1: "Velocity 1 (m/s)",
    v2: "Velocity 2 (m/s)",
    pos_x: "Position 1 X (m)",
  };

  chartInstance.data.labels = dataLogs.map((l) => l.time.toFixed(2));
  chartInstance.data.datasets[0].label = labelMap[chartConfig.variable];
  chartInstance.data.datasets[0].data = dataLogs.map((l) => l[key]);
  chartInstance.update();
}

// --- Exporting ---

function exportCSV() {
  if (dataLogs.length === 0) return;

  const headers = Object.keys(dataLogs[0]).join(",");
  const rows = dataLogs.map((obj) => Object.values(obj).join(","));
  const csvContent = [headers, ...rows].join("\n");

  const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.setAttribute("href", url);
  link.setAttribute("download", "collision_data.csv");
  link.style.visibility = "hidden";
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function exportChartCSV() {
  if (!chartInstance || dataLogs.length === 0) return;

  const label = chartInstance.data.datasets[0].label || "Value";
  const rows = chartInstance.data.labels!.map(
    (t, i) => `${t},${chartInstance!.data.datasets[0].data[i]}`,
  );
  const csvContent = `Time,${label}\n` + rows.join("\n");

  const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.download = "chart_data.csv";
  link.click();
}
</script>

<style scoped>
/* No extra CSS needed due to Tailwind/DaisyUI */
</style>
