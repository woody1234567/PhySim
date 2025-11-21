<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-100 text-base-content"
  >
    <!-- Left: 3D Simulation Canvas -->
    <div
      ref="canvasContainer"
      class="relative flex-grow overflow-hidden bg-black"
    >
      <canvas ref="canvasRef" class="block h-full w-full outline-none"></canvas>

      <!-- Overlay Info -->
      <div
        class="absolute top-4 left-4 pointer-events-none text-white/80 font-mono text-sm"
      >
        <div>Stopping Distance Simulation</div>
        <div>Time: {{ simulationTime.toFixed(2) }} s</div>
        <div>Velocity: {{ currentVelocity.toFixed(2) }} m/s</div>
        <div>Distance: {{ currentPosition.toFixed(2) }} m</div>
      </div>
    </div>

    <!-- Resizer Handle -->
    <div
      class="w-2 cursor-col-resize bg-base-300 hover:bg-primary transition-colors z-10 flex items-center justify-center"
      @mousedown="startResize"
    >
      <div class="h-8 w-1 bg-base-content/20 rounded"></div>
    </div>

    <!-- Right: Sidebar -->
    <div
      class="flex flex-col bg-base-200 border-l border-base-300 shadow-xl z-20"
      :style="{ width: sidebarWidth + 'px', minWidth: '260px' }"
    >
      <div class="p-4 border-b border-base-300 font-bold text-lg bg-base-100">
        Simulation Controls
      </div>

      <div class="flex-grow overflow-y-auto p-2 space-y-2">
        <!-- Panel 1: Control Panel -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="checkbox" checked />
          <div class="collapse-title text-md font-medium">⚙️ Parameters</div>
          <div class="collapse-content space-y-4">
            <!-- Mass -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Mass (kg)</span>
                <span class="label-text-alt">{{ params.mass }}</span>
              </label>
              <input
                type="range"
                min="1"
                max="100"
                step="1"
                v-model.number="params.mass"
                class="range range-xs range-primary"
                :disabled="isPlaying"
              />
            </div>

            <!-- Initial Velocity -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Initial Velocity (m/s)</span>
                <span class="label-text-alt">{{ params.initialVelocity }}</span>
              </label>
              <input
                type="range"
                min="1"
                max="50"
                step="1"
                v-model.number="params.initialVelocity"
                class="range range-xs range-secondary"
                :disabled="isPlaying"
              />
            </div>

            <!-- Braking Force -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Braking Force (N)</span>
                <span class="label-text-alt">{{ params.brakingForce }}</span>
              </label>
              <input
                type="range"
                min="1"
                max="200"
                step="1"
                v-model.number="params.brakingForce"
                class="range range-xs range-accent"
                :disabled="isPlaying"
              />
            </div>

            <!-- Time Step -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Time Step (s)</span>
                <span class="label-text-alt">{{ params.timeStep }}</span>
              </label>
              <select
                class="select select-bordered select-sm w-full"
                v-model.number="params.timeStep"
                :disabled="isPlaying"
              >
                <option :value="1 / 60">1/60 (60 Hz)</option>
                <option :value="1 / 120">1/120 (120 Hz)</option>
                <option :value="0.01">0.01 (10ms)</option>
              </select>
            </div>

            <!-- Action Buttons -->
            <div class="flex gap-2 pt-2">
              <button class="btn btn-primary flex-1" @click="toggleSimulation">
                {{ isPlaying ? "Pause" : isFinished ? "Restart" : "Start" }}
              </button>
              <button
                class="btn btn-outline btn-error"
                @click="resetSimulation"
              >
                Reset
              </button>
            </div>
          </div>
        </div>

        <!-- Panel 2: Live Data Panel -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="checkbox" />
          <div class="collapse-title text-md font-medium">📊 Live Data</div>
          <div class="collapse-content">
            <div class="overflow-x-auto">
              <table class="table table-xs w-full">
                <tbody>
                  <tr>
                    <th>Time</th>
                    <td class="font-mono">{{ simulationTime.toFixed(3) }} s</td>
                  </tr>
                  <tr>
                    <th>Position X</th>
                    <td class="font-mono">
                      {{ currentPosition.toFixed(3) }} m
                    </td>
                  </tr>
                  <tr>
                    <th>Velocity X</th>
                    <td class="font-mono">
                      {{ currentVelocity.toFixed(3) }} m/s
                    </td>
                  </tr>
                  <tr>
                    <th>Accel X</th>
                    <td class="font-mono">
                      {{ currentAcceleration.toFixed(3) }} m/s²
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
            <div class="mt-4">
              <button
                class="btn btn-sm btn-block btn-ghost border-base-300"
                @click="exportLogsToCSV"
              >
                📥 Export CSV
              </button>
            </div>
          </div>
        </div>

        <!-- Panel 3: Chart Panel -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="checkbox" />
          <div class="collapse-title text-md font-medium">📈 Charts</div>
          <div class="collapse-content">
            <div
              class="h-48 w-full bg-base-100 rounded border border-base-200 p-1 relative"
            >
              <canvas ref="chartCanvasRef"></canvas>
            </div>
            <div class="flex gap-2 mt-2">
              <button
                class="btn btn-xs btn-secondary flex-1"
                @click="updateChart"
              >
                Plot Chart
              </button>
              <button
                class="btn btn-xs btn-ghost border-base-300 flex-1"
                @click="exportChartCSV"
              >
                Export Data
              </button>
            </div>
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
  onBeforeUnmount,
  reactive,
  watch,
  nextTick,
} from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";

// --- State & Refs ---
const sidebarWidth = ref(320);
const canvasContainer = ref<HTMLElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const chartCanvasRef = ref<HTMLCanvasElement | null>(null);

// Simulation Parameters
const params = reactive({
  mass: 10,
  initialVelocity: 20,
  brakingForce: 50,
  timeStep: 1 / 60,
});

// Simulation State
const isPlaying = ref(false);
const isFinished = ref(false);
const simulationTime = ref(0);
const currentPosition = ref(0);
const currentVelocity = ref(0);
const currentAcceleration = ref(0);

// Logging
interface LogEntry {
  t: number;
  x: number;
  v: number;
  a: number;
}
const logs = ref<LogEntry[]>([]);

// Three.js Globals
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let ballMesh: THREE.Mesh;
let floorMesh: THREE.Mesh;
let animationId: number | null = null;

// Cannon.js Globals
let world: CANNON.World;
let ballBody: CANNON.Body;

// Chart.js Globals
let chartInstance: Chart | null = null;

// --- Initialization ---

onMounted(() => {
  if (process.client) {
    initThree();
    initPhysics();
    initChart();
    resetSimulation();

    window.addEventListener("resize", onWindowResize);
    animate();
  }
});

onBeforeUnmount(() => {
  if (process.client) {
    window.removeEventListener("resize", onWindowResize);
    if (animationId) cancelAnimationFrame(animationId);
    if (renderer) renderer.dispose();
    if (chartInstance) chartInstance.destroy();
  }
});

// --- Three.js Setup ---
function initThree() {
  if (!canvasRef.value || !canvasContainer.value) return;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);
  scene.fog = new THREE.Fog(0x111111, 20, 200);

  // Camera
  const aspect =
    canvasContainer.value.clientWidth / canvasContainer.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 1000);
  camera.position.set(0, 10, 100);

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.setSize(
    canvasContainer.value.clientWidth,
    canvasContainer.value.clientHeight
  );
  renderer.shadowMap.enabled = true;

  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  // Lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(10, 20, 10);
  dirLight.castShadow = true;
  dirLight.shadow.camera.top = 20;
  dirLight.shadow.camera.bottom = -20;
  dirLight.shadow.camera.left = -20;
  dirLight.shadow.camera.right = 20;
  scene.add(dirLight);

  // Objects
  // Floor
  const floorGeo = new THREE.PlaneGeometry(200, 20);
  const floorMat = new THREE.MeshStandardMaterial({
    color: 0x333333,
    roughness: 0.8,
    metalness: 0.2,
  });
  floorMesh = new THREE.Mesh(floorGeo, floorMat);
  floorMesh.rotation.x = -Math.PI / 2;
  floorMesh.receiveShadow = true;
  scene.add(floorMesh);

  // Grid Helper
  const gridHelper = new THREE.GridHelper(200, 20, 0x444444, 0x222222);
  scene.add(gridHelper);

  // Ball
  const ballGeo = new THREE.SphereGeometry(1, 32, 32);
  const ballMat = new THREE.MeshStandardMaterial({
    color: 0x00ff88,
    roughness: 0.4,
    metalness: 0.5,
  });
  ballMesh = new THREE.Mesh(ballGeo, ballMat);
  ballMesh.castShadow = true;
  scene.add(ballMesh);
}

// --- Physics Setup ---
function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, -9.81, 0);
  world.broadphase = new CANNON.NaiveBroadphase();

  // Physics Materials
  const groundMat = new CANNON.Material();
  const ballMat = new CANNON.Material();
  const contactMat = new CANNON.ContactMaterial(groundMat, ballMat, {
    friction: 0.0, // We simulate braking manually
    restitution: 0.5,
  });
  world.addContactMaterial(contactMat);

  // Ground Body
  const groundBody = new CANNON.Body({
    mass: 0, // static
    shape: new CANNON.Plane(),
    material: groundMat,
  });
  groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
  world.addBody(groundBody);

  // Ball Body
  ballBody = new CANNON.Body({
    mass: params.mass,
    shape: new CANNON.Sphere(1),
    material: ballMat,
  });
  ballBody.linearDamping = 0;
  ballBody.angularDamping = 0;
  world.addBody(ballBody);
}

// --- Chart.js Setup ---
function initChart() {
  if (!chartCanvasRef.value) return;

  chartInstance = new Chart(chartCanvasRef.value, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        {
          label: "Velocity (m/s)",
          data: [],
          borderColor: "rgb(75, 192, 192)",
          tension: 0.1,
          pointRadius: 0,
        },
        {
          label: "Position (m)",
          data: [],
          borderColor: "rgb(255, 99, 132)",
          tension: 0.1,
          pointRadius: 0,
          yAxisID: "y1",
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: false,
      interaction: {
        mode: "index",
        intersect: false,
      },
      scales: {
        x: {
          title: { display: true, text: "Time (s)" },
        },
        y: {
          type: "linear",
          display: true,
          position: "left",
          title: { display: true, text: "Velocity" },
        },
        y1: {
          type: "linear",
          display: true,
          position: "right",
          title: { display: true, text: "Position" },
          grid: {
            drawOnChartArea: false,
          },
        },
      },
    },
  });
}

function updateChart() {
  if (!chartInstance) return;

  // Downsample for performance if needed, but for this simple sim, all points are fine
  const labels = logs.value.map((l) => l.t.toFixed(2));
  const velData = logs.value.map((l) => l.v);
  const posData = logs.value.map((l) => l.x);

  chartInstance.data.labels = labels;
  chartInstance.data.datasets[0].data = velData;
  chartInstance.data.datasets[1].data = posData;
  chartInstance.update();
}

// --- Simulation Logic ---

function resetSimulation() {
  isPlaying.value = false;
  isFinished.value = false;
  simulationTime.value = 0;
  logs.value = [];

  // Reset Physics Body
  ballBody.mass = params.mass;
  ballBody.updateMassProperties();
  ballBody.position.set(-50, 1, 0); // Start on the left
  ballBody.velocity.set(0, 0, 0);
  ballBody.angularVelocity.set(0, 0, 0);
  ballBody.force.set(0, 0, 0);

  // Reset Visuals
  ballMesh.position.copy(ballBody.position as any);
  ballMesh.quaternion.copy(ballBody.quaternion as any);

  // Reset State
  currentPosition.value = ballBody.position.x;
  currentVelocity.value = 0;
  currentAcceleration.value = 0;

  // Clear Chart
  if (chartInstance) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets.forEach((ds) => (ds.data = []));
    chartInstance.update();
  }

  // Initial Log
  logFrame();
}

function toggleSimulation() {
  if (isFinished.value) {
    resetSimulation();
    // Small delay to ensure reset applies before starting
    setTimeout(() => {
      isPlaying.value = true;
      // Apply initial velocity
      ballBody.velocity.set(params.initialVelocity, 0, 0);
    }, 100);
  } else {
    isPlaying.value = !isPlaying.value;
    if (isPlaying.value && simulationTime.value === 0) {
      // First start
      ballBody.velocity.set(params.initialVelocity, 0, 0);
    }
  }
}

function stepSimulation() {
  if (!isPlaying.value || isFinished.value) return;

  // 1. Apply Braking Force
  // Force is opposite to velocity
  // F = ma -> a = F/m
  // We want constant braking force magnitude

  const velX = ballBody.velocity.x;

  if (velX > 0.01) {
    // Apply force in -X direction
    // Cannon applies force for one step. We need to set it before step.
    // Force vector: (-brakingForce, 0, 0)
    ballBody.force.set(-params.brakingForce, 0, 0);
    currentAcceleration.value = -params.brakingForce / params.mass;
  } else {
    // Stop condition
    ballBody.velocity.set(0, 0, 0);
    ballBody.force.set(0, 0, 0);
    currentAcceleration.value = 0;
    isPlaying.value = false;
    isFinished.value = true;
    // Final log
    logFrame();
    return;
  }

  // 2. Step Physics World
  world.step(params.timeStep);
  simulationTime.value += params.timeStep;

  // 3. Sync Visuals
  ballMesh.position.copy(ballBody.position as any);
  ballMesh.quaternion.copy(ballBody.quaternion as any);

  // Update Camera to follow ball (optional, but nice)
  // camera.position.x = ballMesh.position.x + 20
  // controls.target.x = ballMesh.position.x

  // 4. Update State
  currentPosition.value = ballBody.position.x;
  currentVelocity.value = ballBody.velocity.x;

  // 5. Log Data
  logFrame();
}

function logFrame() {
  logs.value.push({
    t: simulationTime.value,
    x: currentPosition.value,
    v: currentVelocity.value,
    a: currentAcceleration.value,
  });
}

function animate() {
  animationId = requestAnimationFrame(animate);

  stepSimulation();
  controls.update();
  renderer.render(scene, camera);
}

// --- Resize Logic ---
const isResizing = ref(false);

function startResize(e: MouseEvent) {
  isResizing.value = true;
  document.addEventListener("mousemove", handleResize);
  document.addEventListener("mouseup", stopResize);
  document.body.style.cursor = "col-resize";
}

function handleResize(e: MouseEvent) {
  if (!isResizing.value) return;

  // Calculate new width from right edge
  const newWidth = window.innerWidth - e.clientX;

  // Constraints
  const minCanvasWidth = 300;
  const maxSidebarWidth = window.innerWidth - minCanvasWidth;
  const minSidebarWidth = 260;

  sidebarWidth.value = Math.max(
    minSidebarWidth,
    Math.min(newWidth, maxSidebarWidth)
  );

  // Update Three.js
  onWindowResize();
}

function stopResize() {
  isResizing.value = false;
  document.removeEventListener("mousemove", handleResize);
  document.removeEventListener("mouseup", stopResize);
  document.body.style.cursor = "";
}

function onWindowResize() {
  if (!canvasContainer.value || !renderer || !camera) return;

  const width = canvasContainer.value.clientWidth;
  const height = canvasContainer.value.clientHeight;

  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
}

// --- Export Logic ---
function exportLogsToCSV() {
  const headers = [
    "Time (s)",
    "Position (m)",
    "Velocity (m/s)",
    "Acceleration (m/s^2)",
  ];
  const rows = logs.value.map((l) => [
    l.t.toFixed(4),
    l.x.toFixed(4),
    l.v.toFixed(4),
    l.a.toFixed(4),
  ]);

  downloadCSV(headers, rows, "simulation_data.csv");
}

function exportChartCSV() {
  // Same data essentially, but triggered from chart panel
  exportLogsToCSV();
}

function downloadCSV(headers: string[], rows: string[][], filename: string) {
  const csvContent = [
    headers.join(","),
    ...rows.map((row) => row.join(",")),
  ].join("\n");

  const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
  const link = document.createElement("a");
  if (link.download !== undefined) {
    const url = URL.createObjectURL(blob);
    link.setAttribute("href", url);
    link.setAttribute("download", filename);
    link.style.visibility = "hidden";
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  }
}
</script>

<style scoped>
/* Custom scrollbar for sidebar */
::-webkit-scrollbar {
  width: 8px;
}
::-webkit-scrollbar-track {
  background: #f1f1f1;
}
::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 4px;
}
::-webkit-scrollbar-thumb:hover {
  background: #555;
}
</style>
