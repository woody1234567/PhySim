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

      <!-- Help Button (Top Right Overlay) -->
      <button
        @click="showHelpModal = true"
        class="absolute top-4 right-4 btn btn-circle btn-ghost btn-md z-10 text-primary-focus bg-base-100/50 backdrop-blur hover:bg-base-200 transition-all pointer-events-auto"
        title="Physics Explanation"
      >
        <Icon name="heroicons:question-mark-circle" class="text-3xl" />
      </button>

      <!-- Physics Help Modal -->
      <dialog class="modal" :class="{ 'modal-open': showHelpModal }">
        <div class="modal-box max-w-2xl bg-base-100 border border-base-300 text-base-content">
          <h3 class="font-bold text-2xl mb-4 flex items-center gap-2">
            <Icon name="heroicons:academic-cap" class="text-primary text-3xl" />
            Physics Concepts: Stopping Distance
          </h3>

          <div class="space-y-4 text-sm leading-relaxed overflow-y-auto max-h-[70vh] pr-2">
            <section>
              <h4 class="font-bold text-lg text-secondary">1. Newton's Second Law</h4>
              <p>
                When braking, a constant force (<MathLatex formula="F" />) is applied opposite to the direction of motion. The resulting deceleration (<MathLatex formula="a" />) is given by:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="a = \frac{F}{m}" :inline="false" />
              </div>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">2. Work-Energy Theorem</h4>
              <p>
                The stopping distance (<MathLatex formula="d" />) can be derived by equating the work done by the braking force to the change in kinetic energy:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="F \cdot d = \frac{1}{2} m v_0^2" :inline="false" />
              </div>
              <p>
                Solving for <MathLatex formula="d" />:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="d = \frac{m v_0^2}{2 F}" :inline="false" />
              </div>
              <p class="text-xs italic opacity-70">Note: Notice that stopping distance is proportional to the square of the initial velocity!</p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">3. Kinematic Equations</h4>
              <p>
                Alternatively, using kinematics for constant acceleration:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="v_f^2 = v_0^2 + 2 a d" :inline="false" />
              </div>
              <p>
                Setting final velocity <MathLatex formula="v_f = 0" /> and deceleration <MathLatex formula="a = -F/m" /> yields the same distance formula.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">4. Controls Guide</h4>
              <ul class="list-disc ml-6 space-y-2">
                <li><strong>Mass (m):</strong> Increasing mass increases the inertia, requiring more distance to stop for the same force.</li>
                <li><strong>Initial Velocity (v₀):</strong> Doubling the velocity will quadruple the stopping distance.</li>
                <li><strong>Braking Force (F):</strong> A stronger force leads to a shorter stopping distance and time.</li>
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
      class="w-2 cursor-col-resize bg-base-300 hover:bg-primary transition-colors z-10 flex items-center justify-center"
      @mousedown="startResize"
    >
      <div class="h-8 w-1 bg-base-content/20 rounded-full"></div>
    </div>

    <!-- Right: Sidebar -->
    <div
      class="flex flex-col bg-base-200 border-l border-base-300 shadow-xl z-20"
      :style="{ width: sidebarWidth + 'px', minWidth: '260px' }"
    >
      <div class="p-4 border-b border-base-300 font-bold text-lg bg-base-100 flex justify-between items-center">
        <span>Simulation Controls</span>
        <UButton
          icon="i-lucide-book-open"
          color="neutral"
          variant="ghost"
          size="xs"
          @click.stop="showHelpModal = true"
        />
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
const showHelpModal = ref(true); // Auto-show on first load

// Logging
interface LogEntry { t: number; x: number; v: number; a: number; }
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
    initThree(); initPhysics(); initChart(); resetSimulation();
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
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);
  scene.fog = new THREE.Fog(0x111111, 20, 200);
  const aspect = canvasContainer.value.clientWidth / canvasContainer.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 1000);
  camera.position.set(0, 10, 100);
  renderer = new THREE.WebGLRenderer({ canvas: canvasRef.value, antialias: true, });
  renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight);
  renderer.shadowMap.enabled = true;
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.add(ambientLight);
  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(10, 20, 10);
  dirLight.castShadow = true;
  scene.add(dirLight);
  const floorGeo = new THREE.PlaneGeometry(200, 20);
  const floorMat = new THREE.MeshStandardMaterial({ color: 0x333333, roughness: 0.8, metalness: 0.2, });
  floorMesh = new THREE.Mesh(floorGeo, floorMat);
  floorMesh.rotation.x = -Math.PI / 2;
  floorMesh.receiveShadow = true;
  scene.add(floorMesh);
  const gridHelper = new THREE.GridHelper(200, 20, 0x444444, 0x222222);
  scene.add(gridHelper);
  const ballGeo = new THREE.SphereGeometry(1, 32, 32);
  const ballMat = new THREE.MeshStandardMaterial({ color: 0x00ff88, roughness: 0.4, metalness: 0.5, });
  ballMesh = new THREE.Mesh(ballGeo, ballMat);
  ballMesh.castShadow = true;
  scene.add(ballMesh);
}

function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, -9.81, 0);
  const groundMat = new CANNON.Material();
  const ballMat = new CANNON.Material();
  const contactMat = new CANNON.ContactMaterial(groundMat, ballMat, { friction: 0.0, restitution: 0.5, });
  world.addContactMaterial(contactMat);
  const groundBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane(), material: groundMat, });
  groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
  world.addBody(groundBody);
  ballBody = new CANNON.Body({ mass: params.mass, shape: new CANNON.Sphere(1), material: ballMat, });
  ballBody.linearDamping = 0; ballBody.angularDamping = 0;
  world.addBody(ballBody);
}

function initChart() {
  if (!chartCanvasRef.value) return;
  chartInstance = new Chart(chartCanvasRef.value, {
    type: "line",
    data: { labels: [], datasets: [{ label: "Velocity (m/s)", data: [], borderColor: "rgb(75, 192, 192)", tension: 0.1, pointRadius: 0, }, { label: "Position (m)", data: [], borderColor: "rgb(255, 99, 132)", tension: 0.1, pointRadius: 0, yAxisID: "y1", },], },
    options: { responsive: true, maintainAspectRatio: false, animation: false, interaction: { mode: "index", intersect: false, }, scales: { x: { title: { display: true, text: "Time (s)" } }, y: { type: "linear", display: true, position: "left", title: { display: true, text: "Velocity" } }, y1: { type: "linear", display: true, position: "right", title: { display: true, text: "Position" }, grid: { drawOnChartArea: false, }, }, }, },
  });
}

function updateChart() {
  if (!chartInstance) return;
  chartInstance.data.labels = logs.value.map((l) => l.t.toFixed(2));
  chartInstance.data.datasets[0].data = logs.value.map((l) => l.v);
  chartInstance.data.datasets[1].data = logs.value.map((l) => l.x);
  chartInstance.update();
}

function resetSimulation() {
  isPlaying.value = false; isFinished.value = false; simulationTime.value = 0; logs.value = [];
  ballBody.mass = params.mass; ballBody.updateMassProperties();
  ballBody.position.set(-50, 1, 0); ballBody.velocity.set(0, 0, 0); ballBody.angularVelocity.set(0, 0, 0); ballBody.force.set(0, 0, 0);
  ballMesh.position.copy(ballBody.position as any); ballMesh.quaternion.copy(ballBody.quaternion as any);
  currentPosition.value = ballBody.position.x; currentVelocity.value = 0; currentAcceleration.value = 0;
  if (chartInstance) { chartInstance.data.labels = []; chartInstance.data.datasets.forEach((ds) => (ds.data = [])); chartInstance.update(); }
  logFrame();
}

function toggleSimulation() {
  if (isFinished.value) { resetSimulation(); setTimeout(() => { isPlaying.value = true; ballBody.velocity.set(params.initialVelocity, 0, 0); }, 100); }
  else { isPlaying.value = !isPlaying.value; if (isPlaying.value && simulationTime.value === 0) ballBody.velocity.set(params.initialVelocity, 0, 0); }
}

function stepSimulation() {
  if (!isPlaying.value || isFinished.value) return;
  const velX = ballBody.velocity.x;
  if (velX > 0.01) { ballBody.force.set(-params.brakingForce, 0, 0); currentAcceleration.value = -params.brakingForce / params.mass; }
  else { ballBody.velocity.set(0, 0, 0); ballBody.force.set(0, 0, 0); currentAcceleration.value = 0; isPlaying.value = false; isFinished.value = true; logFrame(); return; }
  world.step(params.timeStep);
  simulationTime.value += params.timeStep;
  ballMesh.position.copy(ballBody.position as any); ballMesh.quaternion.copy(ballBody.quaternion as any);
  currentPosition.value = ballBody.position.x; currentVelocity.value = ballBody.velocity.x;
  logFrame();
}

function logFrame() { logs.value.push({ t: simulationTime.value, x: currentPosition.value, v: currentVelocity.value, a: currentAcceleration.value, }); }
function animate() { animationId = requestAnimationFrame(animate); stepSimulation(); controls.update(); renderer.render(scene, camera); }

let isResizing = false;
function startResize(e: MouseEvent) { isResizing = true; document.addEventListener("mousemove", handleResize); document.addEventListener("mouseup", stopResize); document.body.style.cursor = "col-resize"; }
function handleResize(e: MouseEvent) { if (!isResizing.value) return; const newWidth = window.innerWidth - e.clientX; sidebarWidth.value = Math.max(260, Math.min(newWidth, window.innerWidth - 300)); onWindowResize(); }
function stopResize() { isResizing.value = false; document.removeEventListener("mousemove", handleResize); document.removeEventListener("mouseup", stopResize); document.body.style.cursor = ""; }

function onWindowResize() {
  if (!canvasContainer.value || !renderer || !camera) return;
  const width = canvasContainer.value.clientWidth; const height = canvasContainer.value.clientHeight;
  camera.aspect = width / height; camera.updateProjectionMatrix(); renderer.setSize(width, height);
}

function exportLogsToCSV() {
  const headers = ["Time (s)", "Position (m)", "Velocity (m/s)", "Acceleration (m/s^2)"];
  const rows = logs.value.map((l) => [l.t.toFixed(4), l.x.toFixed(4), l.v.toFixed(4), l.a.toFixed(4)]);
  downloadCSV(headers, rows, "simulation_data.csv");
}
function exportChartCSV() { exportLogsToCSV(); }

function downloadCSV(headers: string[], rows: string[][], filename: string) {
  const csvContent = [headers.join(","), ...rows.map((row) => row.join(",")), ].join("\n");
  const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
  const link = document.createElement("a"); if (link.download !== undefined) { const url = URL.createObjectURL(blob); link.setAttribute("href", url); link.setAttribute("download", filename); link.style.visibility = "hidden"; document.body.appendChild(link); link.click(); document.body.removeChild(link); }
}
</script>

<style scoped>
::-webkit-scrollbar { width: 8px; }
::-webkit-scrollbar-track { background: #f1f1f1; }
::-webkit-scrollbar-thumb { background: #888; border-radius: 4px; }
::-webkit-scrollbar-thumb:hover { background: #555; }
</style>
