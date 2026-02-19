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
            Physics Concepts: Elastic Collisions
          </h3>

          <div class="space-y-4 text-sm leading-relaxed overflow-y-auto max-h-[70vh] pr-2">
            <section>
              <h4 class="font-bold text-lg text-secondary">1. Momentum Conservation</h4>
              <p>
                In an isolated system, the total momentum before collision equals the total momentum after collision:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="\vec{P}_{total} = m_1 \vec{v}_1 + m_2 \vec{v}_2 = constant" :inline="false" />
              </div>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">2. Kinetic Energy Conservation</h4>
              <p>
                This simulation models <strong>perfectly elastic collisions</strong>, where the total kinetic energy is preserved:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="\sum K = \frac{1}{2} m_1 v_1^2 + \frac{1}{2} m_2 v_2^2 = constant" :inline="false" />
              </div>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">3. 3D Collision Resolution</h4>
              <p>
                The velocities after impact are calculated using the 3D elastic collision formula along the normal vector (<MathLatex formula="\vec{n}" />) connecting the centers:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="\vec{v}_1' = \vec{v}_1 - \frac{2 m_2}{m_1 + m_2} \frac{\langle \vec{v}_1 - \vec{v}_2, \vec{x}_1 - \vec{x}_2 \rangle}{|\vec{x}_1 - \vec{x}_2|^2} (\vec{x}_1 - \vec{x}_2)" :inline="false" />
              </div>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">4. Controls & Features</h4>
              <ul class="list-disc ml-6 space-y-2">
                <li><strong>Mass (m):</strong> Adjusting mass changes the inertia and the resulting velocities after impact.</li>
                <li><strong>Initial Velocity:</strong> Define the 3D velocity vector (Vx, Vy, Vz) for each sphere.</li>
                <li><strong>Gravity:</strong> Toggle Earth's gravity to see how parabolic motion interacts with collisions.</li>
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
import { Chart, registerables } from "chart.js";

// Register Chart.js components
if (import.meta.client) {
  Chart.register(...registerables);
}

// --- Types & Interfaces ---
interface Vector3 { x: number; y: number; z: number; }
interface PhysicsBody { mass: number; radius: number; position: Vector3; velocity: Vector3; }

// --- State Management ---
const canvasRef = ref<HTMLCanvasElement | null>(null);
const chartCanvasRef = ref<HTMLCanvasElement | null>(null);
const sidebarWidth = ref(320);
const isRunning = ref(false);
const time = ref(0);
const showHelpModal = ref(true); // Auto-show on first load

// Drag Resizing Logic
const isDragging = ref(false);
function startResize() { isDragging.value = true; }
function stopResize() { isDragging.value = false; }
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
  body1: { mass: 1.0, velocity: { x: 5, y: 0, z: 0 }, radius: 1, color: 0xff4444, },
  body2: { mass: 1.0, velocity: { x: -5, y: 0, z: 0 }, radius: 1, color: 0x4444ff, },
});

// Physics State
const bodies = shallowRef<PhysicsBody[]>([
  { mass: 1, radius: 1, position: { x: 0, y: 0, z: 0 }, velocity: { x: 0, y: 0, z: 0 }, },
  { mass: 1, radius: 1, position: { x: 0, y: 0, z: 0 }, velocity: { x: 0, y: 0, z: 0 }, },
]);

// Live Data Display
const displayData = reactive({ totalMomentum: 0, totalEnergy: 0, v1: 0, v2: 0, p1: 0, p2: 0, });

// Logging System
interface LogEntry { time: number; p_tot: number; k_tot: number; v1_mag: number; v2_mag: number; pos1_x: number; pos2_x: number; }
let dataLogs: LogEntry[] = [];

// Chart State
let chartInstance: Chart | null = null;
const chartConfig = reactive({ variable: "momentum", });

// --- Three.js Variables ---
const scene = shallowRef<THREE.Scene | null>(null);
const camera = shallowRef<THREE.PerspectiveCamera | null>(null);
const renderer = shallowRef<THREE.WebGLRenderer | null>(null);
const controls = shallowRef<OrbitControls | null>(null);
const mBody1 = shallowRef<THREE.Mesh | null>(null);
const mBody2 = shallowRef<THREE.Mesh | null>(null);
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
  scene.value = new THREE.Scene();
  scene.value.background = new THREE.Color(0x202025);
  const gridHelper = new THREE.GridHelper(50, 50, 0x444444, 0x222222);
  scene.value.add(gridHelper);

  camera.value = new THREE.PerspectiveCamera(45, (window.innerWidth - sidebarWidth.value) / window.innerHeight, 0.1, 1000,);
  camera.value.position.set(10, 10, 20);
  camera.value.lookAt(0, 0, 0);

  renderer.value = new THREE.WebGLRenderer({ canvas: canvasRef.value, antialias: true, });
  renderer.value.shadowMap.enabled = true;
  renderer.value.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.value.setSize(window.innerWidth - sidebarWidth.value, window.innerHeight,);

  controls.value = new OrbitControls(camera.value, renderer.value.domElement);
  controls.value.enableDamping = true;

  const ambLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.value.add(ambLight);
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(10, 20, 10);
  dirLight.castShadow = true;
  dirLight.shadow.mapSize.width = 2048;
  dirLight.shadow.mapSize.height = 2048;
  scene.value.add(dirLight);

  const geo1 = new THREE.SphereGeometry(settings.body1.radius, 32, 32);
  const mat1 = new THREE.MeshStandardMaterial({ color: settings.body1.color, roughness: 0.1, metalness: 0.1, });
  mBody1.value = new THREE.Mesh(geo1, mat1);
  mBody1.value.castShadow = true;
  mBody1.value.receiveShadow = true;
  scene.value.add(mBody1.value);

  const geo2 = new THREE.SphereGeometry(settings.body2.radius, 32, 32);
  const mat2 = new THREE.MeshStandardMaterial({ color: settings.body2.color, roughness: 0.1, metalness: 0.1, });
  mBody2.value = new THREE.Mesh(geo2, mat2);
  mBody2.value.castShadow = true;
  mBody2.value.receiveShadow = true;
  scene.value.add(mBody2.value);
}

function resetSimulation() {
  isRunning.value = false;
  time.value = 0;
  dataLogs = [];
  const startDist = 5;
  bodies.value[0] = { mass: settings.body1.mass, radius: settings.body1.radius, position: { x: -startDist, y: settings.body1.radius, z: 0 }, velocity: { ...settings.body1.velocity }, };
  bodies.value[1] = { mass: settings.body2.mass, radius: settings.body2.radius, position: { x: startDist, y: settings.body2.radius, z: 0 }, velocity: { ...settings.body2.velocity }, };
  updateVisuals();
  updateDisplayData();
}

function stepSimulation() {
  const dt = 1 / 60;
  const gravity = settings.useGravity ? -9.81 : 0;
  const b1 = bodies.value[0];
  const b2 = bodies.value[1];

  if (settings.useGravity) {
    if (b1.position.y > b1.radius) b1.velocity.y += gravity * dt;
    if (b2.position.y > b2.radius) b2.velocity.y += gravity * dt;
  }
  b1.position.x += b1.velocity.x * dt; b1.position.y += b1.velocity.y * dt; b1.position.z += b1.velocity.z * dt;
  b2.position.x += b2.velocity.x * dt; b2.position.y += b2.velocity.y * dt; b2.position.z += b2.velocity.z * dt;

  if (b1.position.y < b1.radius) { b1.position.y = b1.radius; b1.velocity.y *= -1; }
  if (b2.position.y < b2.radius) { b2.position.y = b2.radius; b2.velocity.y *= -1; }

  const dx = b2.position.x - b1.position.x;
  const dy = b2.position.y - b1.position.y;
  const dz = b2.position.z - b1.position.z;
  const distSq = dx * dx + dy * dy + dz * dz;
  const minDist = b1.radius + b2.radius;
  if (distSq < minDist * minDist) resolveCollision(b1, b2);

  time.value += dt;
  updateVisuals();
  updateDisplayData();
  logData();
}

function resolveCollision(b1: PhysicsBody, b2: PhysicsBody) {
  const m1 = b1.mass; const m2 = b2.mass;
  const n = { x: b1.position.x - b2.position.x, y: b1.position.y - b2.position.y, z: b1.position.z - b2.position.z, };
  const nMagSq = n.x * n.x + n.y * n.y + n.z * n.z;
  const nMag = Math.sqrt(nMagSq);
  if (nMag === 0) return;
  const overlap = b1.radius + b2.radius - nMag;
  const correctionRatio = overlap / nMag / 2;
  b1.position.x += n.x * correctionRatio; b1.position.y += n.y * correctionRatio; b1.position.z += n.z * correctionRatio;
  b2.position.x -= n.x * correctionRatio; b2.position.y -= n.y * correctionRatio; b2.position.z -= n.z * correctionRatio;
  const vDiff = { x: b1.velocity.x - b2.velocity.x, y: b1.velocity.y - b2.velocity.y, z: b1.velocity.z - b2.velocity.z, };
  const dotProd = vDiff.x * n.x + vDiff.y * n.y + vDiff.z * n.z;
  const factor = (2 * dotProd) / ((m1 + m2) * nMagSq);
  b1.velocity.x -= factor * m2 * n.x; b1.velocity.y -= factor * m2 * n.y; b1.velocity.z -= factor * m2 * n.z;
  b2.velocity.x += factor * m1 * n.x; b2.velocity.y += factor * m1 * n.y; b2.velocity.z += factor * m1 * n.z;
}

function updateVisuals() {
  const b1 = bodies.value[0]; const b2 = bodies.value[1];
  if (mBody1.value) mBody1.value.position.set(b1.position.x, b1.position.y, b1.position.z);
  if (mBody2.value) mBody2.value.position.set(b2.position.x, b2.position.y, b2.position.z);
}

function updateDisplayData() {
  const b1 = bodies.value[0]; const b2 = bodies.value[1];
  const v1Mag = Math.sqrt(b1.velocity.x ** 2 + b1.velocity.y ** 2 + b1.velocity.z ** 2,);
  const v2Mag = Math.sqrt(b2.velocity.x ** 2 + b2.velocity.y ** 2 + b2.velocity.z ** 2,);
  const K1 = 0.5 * b1.mass * v1Mag ** 2; const K2 = 0.5 * b2.mass * v2Mag ** 2;
  const p1Vec = { x: b1.velocity.x * b1.mass, y: b1.velocity.y * b1.mass, z: b1.velocity.z * b1.mass, };
  const p2Vec = { x: b2.velocity.x * b2.mass, y: b2.velocity.y * b2.mass, z: b2.velocity.z * b2.mass, };
  const pTotVec = { x: p1Vec.x + p2Vec.x, y: p1Vec.y + p2Vec.y, z: p1Vec.z + p2Vec.z, };
  const PTotMag = Math.sqrt(pTotVec.x ** 2 + pTotVec.y ** 2 + pTotVec.z ** 2);
  displayData.v1 = v1Mag; displayData.v2 = v2Mag; displayData.p1 = v1Mag * b1.mass; displayData.p2 = v2Mag * b2.mass;
  displayData.totalEnergy = K1 + K2; displayData.totalMomentum = PTotMag;
}

function logData() {
  dataLogs.push({ time: time.value, p_tot: displayData.totalMomentum, k_tot: displayData.totalEnergy, v1_mag: displayData.v1, v2_mag: displayData.v2, pos1_x: bodies.value[0].position.x, pos2_x: bodies.value[1].position.x, });
}

function renderLoop() {
  animationFrameId = requestAnimationFrame(renderLoop);
  if (isRunning.value) stepSimulation();
  if (controls.value) controls.value.update();
  if (renderer.value && scene.value && camera.value) renderer.value.render(scene.value, camera.value);
}

function toggleSimulation() { isRunning.value = !isRunning.value; }

function handleResize() {
  if (!renderer.value || !camera.value) return;
  const w = window.innerWidth - sidebarWidth.value; const h = window.innerHeight;
  camera.value.aspect = w / h; camera.value.updateProjectionMatrix(); renderer.value.setSize(w, h);
}

function initChart() {
  if (!chartCanvasRef.value) return;
  const ctx = chartCanvasRef.value.getContext("2d"); if (!ctx) return;
  chartInstance = new Chart(ctx, { type: "line", data: { labels: [], datasets: [{ label: "Metric", data: [], borderColor: "#570df8", tension: 0.1, },], }, options: { responsive: true, maintainAspectRatio: false, animation: false, scales: { x: { title: { display: true, text: "Time (s)" } }, y: { title: { display: true, text: "Value" } }, }, }, });
}

function updateChart() {
  if (!chartInstance) return;
  const keyMap: Record<string, keyof LogEntry> = { momentum: "p_tot", energy: "k_tot", v1: "v1_mag", v2: "v2_mag", pos_x: "pos1_x", };
  const key = keyMap[chartConfig.variable] || "p_tot";
  const labelMap: Record<string, string> = { momentum: "Total Momentum (kg·m/s)", energy: "Total Kinetic Energy (J)", v1: "Velocity 1 (m/s)", v2: "Velocity 2 (m/s)", pos_x: "Position 1 X (m)", };
  chartInstance.data.labels = dataLogs.map((l) => l.time.toFixed(2));
  chartInstance.data.datasets[0].label = labelMap[chartConfig.variable];
  chartInstance.data.datasets[0].data = dataLogs.map((l) => l[key]);
  chartInstance.update();
}

function exportCSV() {
  if (dataLogs.length === 0) return;
  const headers = Object.keys(dataLogs[0]).join(",");
  const rows = dataLogs.map((obj) => Object.values(obj).join(","));
  const csvContent = [headers, ...rows].join("\n");
  const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a"); link.setAttribute("href", url); link.setAttribute("download", "collision_data.csv"); link.style.visibility = "hidden"; document.body.appendChild(link); link.click(); document.body.removeChild(link);
}

function exportChartCSV() {
  if (!chartInstance || dataLogs.length === 0) return;
  const label = chartInstance.data.datasets[0].label || "Value";
  const rows = chartInstance.data.labels!.map((t, i) => `${t},${chartInstance!.data.datasets[0].data[i]}`,);
  const csvContent = `Time,${label}\n` + rows.join("\n");
  const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
  const link = document.createElement("a"); link.href = URL.createObjectURL(blob); link.download = "chart_data.csv"; link.click();
}
</script>

<style scoped></style>
