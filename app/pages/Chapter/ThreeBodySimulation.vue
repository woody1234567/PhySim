<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-300 font-sans text-base-content"
  >
    <!-- Left: 3D Simulation Canvas -->
    <div
      ref="canvasContainer"
      class="relative flex-grow overflow-hidden bg-black"
    >
      <canvas
        ref="canvasRef"
        class="block h-full w-full focus:outline-none"
      ></canvas>

      <!-- Overlay Info -->
      <div
        class="pointer-events-none absolute left-4 top-4 rounded bg-base-100/80 p-2 text-xs shadow backdrop-blur"
      >
        <div class="font-bold text-primary">
          Three-Body Gravitational Simulation
        </div>
        <div>Time: {{ simulationTime.toFixed(2) }} s</div>
        <div>Status: {{ isRunning ? "Running" : "Paused" }}</div>
      </div>

      <!-- Help Button (Top Left Overlay) -->
      <button
        @click="showHelpModal = true"
        class="absolute top-20 left-4 btn btn-circle btn-ghost btn-md z-10 text-primary-focus bg-base-100/50 backdrop-blur hover:bg-base-200 transition-all pointer-events-auto"
        title="Physics Explanation"
      >
        <Icon name="heroicons:question-mark-circle" class="text-3xl" />
      </button>

      <!-- Physics Help Modal -->
      <dialog class="modal" :class="{ 'modal-open': showHelpModal }">
        <div class="modal-box max-w-2xl bg-base-100 border border-base-300 text-base-content">
          <h3 class="font-bold text-2xl mb-4 flex items-center gap-2">
            <Icon name="heroicons:academic-cap" class="text-primary text-3xl" />
            Physics Concepts: The Three-Body Problem
          </h3>

          <div class="space-y-4 text-sm leading-relaxed overflow-y-auto max-h-[70vh] pr-2">
            <section>
              <h4 class="font-bold text-lg text-secondary">1. Newton's Law of Universal Gravitation</h4>
              <p>
                Each body in the system exerts a gravitational force on every other body. The force magnitude between body <MathLatex formula="i" /> and body <MathLatex formula="j" /> is:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="F_{ij} = G \frac{m_i m_j}{r^2}" :inline="false" />
              </div>
              <p>
                where <MathLatex formula="r" /> is the distance between the centers of mass of the two bodies.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">2. Vector Sum of Forces</h4>
              <p>
                For any single body, the total force acting on it is the vector sum of the forces exerted by all other bodies:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="\vec{F}_i = \sum_{j \neq i} \vec{F}_{ij}" :inline="false" />
              </div>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">3. Chaos and Non-integrability</h4>
              <p>
                Unlike the two-body problem, the general three-body problem has no closed-form analytical solution. It is known for its <strong>sensitivity to initial conditions</strong>, leading to chaotic orbital patterns that can only be predicted using numerical integration methods.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">4. Controls Guide</h4>
              <ul class="list-disc ml-6 space-y-2">
                <li><strong>Masses:</strong> Increasing a body's mass increases its gravitational pull and its inertia.</li>
                <li><strong>G Constant:</strong> Scales the overall strength of gravity in the simulation.</li>
                <li><strong>Trails:</strong> The glowing lines represent the historical trajectory, helping visualize the complex orbital patterns.</li>
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
      class="w-2 cursor-col-resize bg-base-content/20 hover:bg-primary transition-colors"
      @mousedown="startResize"
    ></div>

    <!-- Right: Sidebar -->
    <div
      class="flex flex-col overflow-y-auto bg-base-200 shadow-xl"
      :style="{
        width: sidebarWidth + 'px',
        minWidth: '260px',
        maxWidth: '600px',
      }"
    >
      <!-- Header -->
      <div class="bg-base-100 p-4 shadow flex justify-between items-center">
        <h2 class="text-xl font-bold text-primary">Control Center</h2>
        <UButton
          icon="i-lucide-book-open"
          color="neutral"
          variant="ghost"
          size="xs"
          @click.stop="showHelpModal = true"
        />
      </div>

      <!-- Panel 1: Controls -->
      <div class="collapse collapse-arrow border-b border-base-300 bg-base-100">
        <input type="checkbox" checked />
        <div class="collapse-title text-lg font-medium">
          Simulation Controls
        </div>
        <div class="collapse-content space-y-4">
          <!-- Global Controls -->
          <div class="flex flex-wrap gap-2">
            <button
              class="btn btn-primary btn-sm flex-1"
              @click="toggleSimulation"
            >
              {{ isRunning ? "Pause" : "Start" }}
            </button>
            <button
              class="btn btn-error btn-sm flex-1"
              @click="resetSimulation"
            >
              Restart
            </button>
          </div>

          <div class="divider my-2">Mass Configuration</div>

          <!-- Body 1 Control -->
          <div class="form-control">
            <label class="label">
              <span class="label-text text-red-500 font-bold"
                >Red Body Mass</span
              >
              <span class="label-text-alt">{{ masses[0] }} kg</span>
            </label>
            <input
              type="range"
              min="1"
              max="100"
              step="1"
              v-model.number="masses[0]"
              class="range range-error range-xs"
              :disabled="isRunning"
            />
          </div>

          <!-- Body 2 Control -->
          <div class="form-control">
            <label class="label">
              <span class="label-text text-green-500 font-bold"
                >Green Body Mass</span
              >
              <span class="label-text-alt">{{ masses[1] }} kg</span>
            </label>
            <input
              type="range"
              min="1"
              max="100"
              step="1"
              v-model.number="masses[1]"
              class="range range-success range-xs"
              :disabled="isRunning"
            />
          </div>

          <!-- Body 3 Control -->
          <div class="form-control">
            <label class="label">
              <span class="label-text text-blue-500 font-bold"
                >Blue Body Mass</span
              >
              <span class="label-text-alt">{{ masses[2] }} kg</span>
            </label>
            <input
              type="range"
              min="1"
              max="100"
              step="1"
              v-model.number="masses[2]"
              class="range range-info range-xs"
              :disabled="isRunning"
            />
          </div>

          <div class="form-control">
            <label class="label cursor-pointer">
              <span class="label-text">Gravitational Constant (G)</span>
              <span class="label-text-alt font-mono">{{ G_CONSTANT }}</span>
            </label>
            <input
              type="range"
              min="1"
              max="20"
              step="0.5"
              v-model.number="G_CONSTANT"
              class="range range-xs"
            />
          </div>
        </div>
      </div>

      <!-- Panel 2: Live Data -->
      <div class="collapse collapse-arrow border-b border-base-300 bg-base-100">
        <input type="checkbox" />
        <div class="collapse-title text-lg font-medium">Live Data</div>
        <div class="collapse-content">
          <div class="mb-4 flex justify-end">
            <button class="btn btn-outline btn-xs" @click="exportLogsToCSV">
              Export CSV
            </button>
          </div>

          <div
            class="h-64 overflow-y-auto rounded bg-base-300 p-2 font-mono text-xs"
          >
            <div
              v-for="(body, idx) in liveData"
              :key="idx"
              class="mb-2 border-b border-base-content/10 pb-2"
            >
              <div :class="['font-bold', bodyColors[idx]]">
                Body {{ idx + 1 }}
              </div>
              <div class="grid grid-cols-2 gap-x-2">
                <span>Pos:</span>
                <span>{{ formatVec(body.position) }}</span>
                <span>Vel:</span>
                <span>{{ formatVec(body.velocity) }}</span>
                <span>Acc:</span>
                <span>{{ formatVec(body.acceleration) }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Panel 3: Charts -->
      <div class="collapse collapse-arrow border-b border-base-300 bg-base-100">
        <input type="checkbox" />
        <div class="collapse-title text-lg font-medium">Charts</div>
        <div class="collapse-content">
          <div class="flex flex-col gap-2">
            <div class="flex gap-2">
              <button
                class="btn btn-sm btn-secondary flex-1"
                @click="updateCharts"
              >
                Plot Chart
              </button>
              <button
                class="btn btn-sm btn-outline flex-1"
                @click="exportChartData"
              >
                Export Data
              </button>
            </div>
            <div class="relative h-64 w-full">
              <canvas ref="chartCanvas"></canvas>
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
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";

// --- State & Config ---
const canvasRef = ref<HTMLCanvasElement | null>(null);
const canvasContainer = ref<HTMLElement | null>(null);
const chartCanvas = ref<HTMLCanvasElement | null>(null);

const isRunning = ref(false);
const simulationTime = ref(0);
const sidebarWidth = ref(320);
const masses = reactive([10, 10, 10]); // Red, Green, Blue
const G_CONSTANT = ref(5);
const TIME_STEP = 1 / 60;
const showHelpModal = ref(true); // Auto-show on first load

// Colors for UI and Three.js
const bodyColors = ["text-red-500", "text-green-500", "text-blue-500"];
const threeColors = [0xff0000, 0x00ff00, 0x0000ff];

// --- Three.js Globals ---
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let meshes: THREE.Mesh[] = [];
let trailLines: THREE.Line[] = [];
const trailPoints: THREE.Vector3[][] = [[], [], []];
const MAX_TRAIL_POINTS = 500;

// --- Cannon.js Globals ---
let world: CANNON.World;
let bodies: CANNON.Body[] = [];

// --- Data Logging ---
interface LogEntry {
  t: number;
  bodies: {
    x: number;
    y: number;
    z: number;
    vx: number;
    vy: number;
    vz: number;
    ax: number;
    ay: number;
    az: number;
  }[];
}
const dataLog = ref<LogEntry[]>([]);
const liveData = ref<any[]>([{}, {}, {}]);

// --- Chart.js ---
let chartInstance: Chart | null = null;

// --- Initialization ---

onMounted(() => {
  if (process.client) {
    initThree();
    initPhysics();
    initOrbitControls();
    initChart();

    requestAnimationFrame(animate);

    handleResize();
    window.addEventListener("resize", handleResize);
    window.addEventListener("mouseup", stopResize);
    window.addEventListener("mousemove", resizeSidebar);
  }
});

onBeforeUnmount(() => {
  if (process.client) {
    window.removeEventListener("resize", handleResize);
    window.removeEventListener("mouseup", stopResize);
    window.removeEventListener("mousemove", resizeSidebar);
    if (chartInstance) chartInstance.destroy();
    if (renderer) renderer.dispose();
  }
});

// --- Three.js Setup ---
function initThree() {
  if (!canvasRef.value) return;

  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x050510);

  const ambientLight = new THREE.AmbientLight(0x404040, 2);
  scene.add(ambientLight);
  const pointLight = new THREE.PointLight(0xffffff, 2, 100);
  pointLight.position.set(0, 20, 20);
  scene.add(pointLight);

  camera = new THREE.PerspectiveCamera(
    45,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  );
  camera.position.set(0, 30, 50);
  camera.lookAt(0, 0, 0);

  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.setSize(window.innerWidth - sidebarWidth.value, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);

  const gridHelper = new THREE.GridHelper(100, 50, 0x444444, 0x222222);
  scene.add(gridHelper);

  const geometry = new THREE.SphereGeometry(1, 32, 32);

  meshes = [];
  trailLines = [];
  threeColors.forEach((color, i) => {
    const material = new THREE.MeshStandardMaterial({
      color: color,
      roughness: 0.4,
      metalness: 0.3,
      emissive: color,
      emissiveIntensity: 0.2,
    });
    const mesh = new THREE.Mesh(geometry, material);
    scene.add(mesh);
    meshes.push(mesh);

    const trailGeo = new THREE.BufferGeometry();
    const trailMat = new THREE.LineBasicMaterial({
      color: color,
      opacity: 0.5,
      transparent: true,
    });
    const trail = new THREE.Line(trailGeo, trailMat);
    scene.add(trail);
    trailLines.push(trail);
  });
}

function initOrbitControls() {
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
}

// --- Physics Setup ---
function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, 0, 0);
  world.broadphase = new CANNON.NaiveBroadphase();
  world.solver.iterations = 10;

  const initialPositions = [
    new CANNON.Vec3(10, 0, 0),
    new CANNON.Vec3(-5, 0, 8.66),
    new CANNON.Vec3(-5, 0, -8.66),
  ];

  const initialVelocities = [
    new CANNON.Vec3(0, 0, 2),
    new CANNON.Vec3(-1.73, 0, -1),
    new CANNON.Vec3(1.73, 0, -1),
  ];

  bodies = [];
  masses.forEach((mass, i) => {
    const shape = new CANNON.Sphere(1);
    const body = new CANNON.Body({
      mass: mass,
      position: initialPositions[i],
      velocity: initialVelocities[i],
      shape: shape,
      linearDamping: 0,
      angularDamping: 0,
    });
    world.addBody(body);
    bodies.push(body);
  });
}

// --- Simulation Logic ---
function applyGravity() {
  for (let i = 0; i < bodies.length; i++) {
    for (let j = i + 1; j < bodies.length; j++) {
      const bi = bodies[i];
      const bj = bodies[j];

      const distVec = new CANNON.Vec3();
      bj.position.vsub(bi.position, distVec);

      const r = distVec.length();
      if (r < 0.1) continue;

      const forceMagnitude = (G_CONSTANT.value * bi.mass * bj.mass) / (r * r);

      distVec.normalize();
      const force = distVec.scale(forceMagnitude);

      bi.force.vadd(force, bi.force);
      bj.force.vsub(force, bj.force);
    }
  }
}

function stepSimulation() {
  applyGravity();
  world.step(TIME_STEP);
  simulationTime.value += TIME_STEP;

  bodies.forEach((body, i) => {
    meshes[i].position.copy(body.position as any);
    meshes[i].quaternion.copy(body.quaternion as any);

    trailPoints[i].push(
      new THREE.Vector3(body.position.x, body.position.y, body.position.z)
    );
    if (trailPoints[i].length > MAX_TRAIL_POINTS) trailPoints[i].shift();
    trailLines[i].geometry.setFromPoints(trailPoints[i]);
  });

  logData();
}

function logData() {
  const entry: LogEntry = {
    t: simulationTime.value,
    bodies: bodies.map((b) => ({
      x: b.position.x,
      y: b.position.y,
      z: b.position.z,
      vx: b.velocity.x,
      vy: b.velocity.y,
      vz: b.velocity.z,
      ax: b.force.x / b.mass,
      ay: b.force.y / b.mass,
      az: b.force.z / b.mass,
    })),
  };

  liveData.value = entry.bodies.map((b) => ({
    position: { x: b.x, y: b.y, z: b.z },
    velocity: { x: b.vx, y: b.vy, z: b.vz },
    acceleration: { x: b.ax, y: b.ay, z: b.az },
  }));

  if (Math.floor(simulationTime.value / TIME_STEP) % 10 === 0) {
    dataLog.value.push(entry);
  }
}

function animate() {
  requestAnimationFrame(animate);
  if (isRunning.value) stepSimulation();
  controls.update();
  renderer.render(scene, camera);
}

// --- Controls ---
function toggleSimulation() { isRunning.value = !isRunning.value; }

function resetSimulation() {
  isRunning.value = false;
  simulationTime.value = 0;
  dataLog.value = [];
  bodies.forEach((b) => world.removeBody(b));
  initPhysics();
  trailPoints.forEach((arr) => (arr.length = 0));
  trailLines.forEach((line) => line.geometry.setFromPoints([]));
  bodies.forEach((body, i) => meshes[i].position.copy(body.position as any));
  if (chartInstance) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets.forEach((ds) => (ds.data = []));
    chartInstance.update();
  }
}

// --- Charting ---
function initChart() {
  if (!chartCanvas.value) return;
  chartInstance = new Chart(chartCanvas.value, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        { label: "R-Vel", data: [], borderColor: "red", borderWidth: 1, pointRadius: 0, },
        { label: "G-Vel", data: [], borderColor: "green", borderWidth: 1, pointRadius: 0, },
        { label: "B-Vel", data: [], borderColor: "blue", borderWidth: 1, pointRadius: 0, },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: false,
      interaction: { mode: "index", intersect: false },
      scales: {
        x: { title: { display: true, text: "Time (s)" } },
        y: { title: { display: true, text: "Velocity Magnitude" } },
      },
    },
  });
}

function updateCharts() {
  if (!chartInstance) return;
  const step = Math.max(1, Math.floor(dataLog.value.length / 100));
  const sampled = dataLog.value.filter((_, i) => i % step === 0);
  chartInstance.data.labels = sampled.map((d) => d.t.toFixed(1));
  chartInstance.data.datasets[0].data = sampled.map((d) => Math.sqrt(d.bodies[0].vx ** 2 + d.bodies[0].vy ** 2 + d.bodies[0].vz ** 2));
  chartInstance.data.datasets[1].data = sampled.map((d) => Math.sqrt(d.bodies[1].vx ** 2 + d.bodies[1].vy ** 2 + d.bodies[1].vz ** 2));
  chartInstance.data.datasets[2].data = sampled.map((d) => Math.sqrt(d.bodies[2].vx ** 2 + d.bodies[2].vy ** 2 + d.bodies[2].vz ** 2));
  chartInstance.update();
}

function exportLogsToCSV() {
  if (dataLog.value.length === 0) return;
  let csv = "t,r_x,r_y,r_z,r_vx,r_vy,r_vz,g_x,g_y,g_z,g_vx,g_vy,g_vz,b_x,b_y,b_z,b_vx,b_vy,b_vz\n";
  dataLog.value.forEach((row) => {
    const r = row.bodies[0], g = row.bodies[1], b = row.bodies[2];
    csv += `${row.t.toFixed(3)},${r.x},${r.y},${r.z},${r.vx},${r.vy},${r.vz},${g.x},${g.y},${g.z},${g.vx},${g.vy},${g.vz},${b.x},${b.y},${b.z},${b.vx},${b.vy},${b.vz}\n`;
  });
  downloadCSV(csv, "simulation_data.csv");
}
function exportChartData() { exportLogsToCSV(); }

function downloadCSV(content: string, filename: string) {
  const blob = new Blob([content], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.setAttribute("href", url);
  link.setAttribute("download", filename);
  link.style.visibility = "hidden";
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

// --- Resizing ---
const isResizing = ref(false);
function startResize() { isResizing.value = true; document.body.style.cursor = "col-resize"; }
function stopResize() { isResizing.value = false; document.body.style.cursor = ""; }
function resizeSidebar(e: MouseEvent) {
  if (!isResizing.value) return;
  const newWidth = window.innerWidth - e.clientX;
  if (newWidth >= 260 && newWidth <= 600 && (window.innerWidth - newWidth >= 300)) {
    sidebarWidth.value = newWidth;
    handleResize();
  }
}
function handleResize() {
  if (!canvasContainer.value || !renderer || !camera) return;
  const w = window.innerWidth - sidebarWidth.value - 8;
  const h = window.innerHeight;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
}
function formatVec(v: { x: number; y: number; z: number } | undefined) {
  if (!v) return "0.00, 0.00, 0.00";
  return `${v.x.toFixed(2)}, ${v.y.toFixed(2)}, ${v.z.toFixed(2)}`;
}
watch(masses, (newMasses) => {
  if (!world) return;
  bodies.forEach((b, i) => { b.mass = newMasses[i]; b.updateMassProperties(); });
}, { deep: true });
</script>

<style scoped>
::-webkit-scrollbar { width: 8px; }
::-webkit-scrollbar-track { background: #1f2937; }
::-webkit-scrollbar-thumb { background: #4b5563; border-radius: 4px; }
::-webkit-scrollbar-thumb:hover { background: #6b7280; }
</style>
