<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-100 font-sans"
    ref="mainContainer"
  >
    <!-- Left: 3D Simulation Canvas -->
    <div
      class="relative flex-grow h-full overflow-hidden"
      ref="canvasContainer"
    >
      <canvas ref="canvasRef" class="block w-full h-full outline-none"></canvas>

      <!-- Overlay Instructions -->
      <div
        class="absolute top-4 left-4 pointer-events-none bg-base-300/80 backdrop-blur text-base-content p-2 rounded shadow-lg text-sm"
      >
        <p class="font-bold">Rutherford Scattering</p>
        <p>Left Click: Rotate</p>
        <p>Right Click: Pan</p>
        <p>Scroll: Zoom</p>
      </div>
    </div>

    <!-- Resizer Handle -->
    <div
      class="w-2 hover:w-3 bg-base-300 hover:bg-primary cursor-col-resize transition-all z-50 flex items-center justify-center"
      @mousedown="startResize"
    >
      <div class="w-1 h-8 bg-base-content/20 rounded"></div>
    </div>

    <!-- Right: Sidebar -->
    <div
      class="flex flex-col h-full bg-base-200 border-l border-base-300 text-base-content shadow-xl overflow-hidden"
      :style="{ width: sidebarWidth + 'px', minWidth: '260px' }"
    >
      <!-- Header -->
      <div class="p-4 border-b border-base-300 bg-base-100">
        <h2 class="text-xl font-bold flex items-center gap-2">
          <span class="text-primary">⚛️</span> Controls
        </h2>
      </div>

      <!-- Scrollable Content -->
      <div class="flex-1 overflow-y-auto p-4 space-y-4">
        <!-- Panel 1: Controls -->
        <div
          class="collapse collapse-arrow bg-base-100 hover:shadow-md transition-shadow duration-200 border border-base-200 rounded-box"
        >
          <input type="checkbox" checked />
          <div class="collapse-title text-lg font-bold flex items-center gap-2">
            ⚙️ Parameters
          </div>
          <div class="collapse-content space-y-4 pt-4">
            <!-- Alpha Speed Control -->
            <div class="form-control">
              <label class="label">
                <span class="label-text flex items-center gap-2">
                  Initial Velocity (v₀)
                  <span class="badge badge-sm badge-ghost">{{
                    alphaVelocity.toFixed(1)
                  }}</span>
                </span>
                <span class="label-text-alt text-xs text-info">Units/s</span>
              </label>
              <input
                type="range"
                min="1"
                max="50"
                step="0.5"
                v-model.number="alphaVelocity"
                class="range range-primary range-xs"
                :disabled="isSimulating"
              />
            </div>

            <!-- Impact Parameter Control -->
            <div class="form-control">
              <label class="label">
                <span class="label-text flex items-center gap-2">
                  Impact Parameter (b)
                  <span class="badge badge-sm badge-ghost">{{
                    impactParameter.toFixed(1)
                  }}</span>
                </span>
                <span class="label-text-alt text-xs text-info"
                  >Vertical Dist</span
                >
              </label>
              <input
                type="range"
                min="0.1"
                max="20"
                step="0.1"
                v-model.number="impactParameter"
                class="range range-accent range-xs"
                :disabled="isSimulating"
              />
            </div>

            <!-- Nucleus Charge Strength -->
            <div class="form-control">
              <label class="label">
                <span class="label-text flex items-center gap-2">
                  Coulomb Constant (kQq)
                  <span class="badge badge-sm badge-ghost">{{
                    coulombConstant
                  }}</span>
                </span>
              </label>
              <input
                type="range"
                min="100"
                max="5000"
                step="100"
                v-model.number="coulombConstant"
                class="range range-warning range-xs"
                :disabled="isSimulating"
              />
            </div>

            <!-- Action Buttons -->
            <div class="flex gap-2 pt-2">
              <button
                class="btn btn-primary flex-1 btn-sm"
                @click="startSimulation"
                v-if="!isSimulating"
              >
                Start
              </button>
              <button
                class="btn btn-error flex-1 btn-sm"
                @click="resetSimulation"
                v-else
              >
                Reset
              </button>

              <button
                class="btn btn-neutral btn-square btn-sm"
                @click="togglePause"
                :disabled="!isSimulating"
              >
                {{ isPaused ? "▶" : "⏸" }}
              </button>
            </div>

            <div class="text-xs text-base-content/60 px-1">
              *Adjust 'b' and Fire multiple times to build the chart!
            </div>
          </div>
        </div>

        <!-- Panel 2: Live Data -->
        <div
          class="collapse collapse-arrow bg-base-100 hover:shadow-md transition-shadow duration-200 border border-base-200 rounded-box"
        >
          <input type="checkbox" checked />
          <div class="collapse-title text-lg font-bold flex items-center gap-2">
            📊 Live Data
          </div>
          <div class="collapse-content space-y-2 pt-4">
            <div
              class="stats stats-vertical w-full shadow-inner bg-base-200/50 text-xs"
            >
              <div class="stat p-2 min-h-0">
                <div class="stat-title">Time</div>
                <div class="stat-value text-lg font-mono">
                  {{ simulationTime.toFixed(2) }} s
                </div>
              </div>

              <div class="stat p-2 min-h-0">
                <div class="stat-title">Separation (r)</div>
                <div class="stat-value text-lg font-mono">
                  {{ currentDistance.toFixed(2) }}
                </div>
                <div class="stat-desc">Distance to Nucleus</div>
              </div>

              <div class="stat p-2 min-h-0">
                <div class="stat-title">Velocity</div>
                <div class="stat-value text-lg font-mono">
                  {{ currentSpeed.toFixed(2) }}
                </div>
              </div>

              <div class="stat p-2 min-h-0" v-if="scatteringAngle !== null">
                <div class="stat-title text-success">Scattering Angle</div>
                <div class="stat-value text-lg font-mono text-success">
                  {{ scatteringAngle.toFixed(2) }}°
                </div>
              </div>
            </div>

            <button
              class="btn btn-outline btn-xs w-full mt-2"
              @click="exportLogsToCSV"
            >
              📥 Export Data (CSV)
            </button>
          </div>
        </div>

        <!-- Panel 3: Charts -->
        <div
          class="collapse collapse-arrow bg-base-100 hover:shadow-md transition-shadow duration-200 border border-base-200 rounded-box"
        >
          <input type="checkbox" checked />
          <div class="collapse-title text-lg font-bold flex items-center gap-2">
            📈 Scattering Analysis
          </div>
          <div class="collapse-content pt-4">
            <div
              class="w-full h-48 bg-base-100 rounded border border-base-200 p-2"
            >
              <canvas ref="chartCanvas"></canvas>
            </div>
            <div class="mt-2 text-xs text-center text-base-content/70">
              Scattering Angle (θ) vs Impact Parameter (b)
            </div>
            <button
              class="btn btn-ghost btn-xs w-full mt-2"
              @click="clearChart"
            >
              🗑️ Clear Chart
            </button>
            <button
              class="btn btn-outline btn-xs w-full mt-1"
              @click="exportChartCSV"
            >
              📥 Export Chart (CSV)
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, reactive, computed, watch } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";

// --- State ---
const sidebarWidth = ref(320);
const isSimulating = ref(false);
const isPaused = ref(false);
const simulationTime = ref(0);
const currentDistance = ref(0);
const currentSpeed = ref(0);
const scatteringAngle = ref<number | null>(null);

// Parameters
const alphaVelocity = ref(10);
const impactParameter = ref(2);
const coulombConstant = ref(1000); // kQq
const particleMass = 1;

// Refs
const canvasRef = ref<HTMLCanvasElement | null>(null);
const canvasContainer = ref<HTMLElement | null>(null);
const mainContainer = ref<HTMLElement | null>(null);
const chartCanvas = ref<HTMLCanvasElement | null>(null);
let chartInstance: Chart | null = null;

// Three.js Globals
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let animationId: number;
let nucleusMesh: THREE.Mesh;
let alphaMesh: THREE.Mesh;
let trailPoints: THREE.Vector3[] = [];
let trailLine: THREE.Line;

// Physics Globals
let world: CANNON.World;
let alphaBody: CANNON.Body;
const timeStep = 1 / 60;

// Logging
const dataLog = ref<any[]>([]);
const chartDataPoints = ref<{ x: number; y: number }[]>([]);

// --- Lifecycle ---
onMounted(() => {
  if (import.meta.client) {
    initThree();
    initPhysics();
    initOrbitControls();
    initChart();

    window.addEventListener("resize", handleResize);
    handleResize(); // Initial sizing

    // Start loop
    animate();
  }
});

onUnmounted(() => {
  if (import.meta.client) {
    window.removeEventListener("resize", handleResize);
    cancelAnimationFrame(animationId);
    if (renderer) renderer.dispose();
    if (chartInstance) chartInstance.destroy();
  }
});

// --- Initialization ---
function initThree() {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x1a1a1a); // Dark background for contrast
  scene.fog = new THREE.FogExp2(0x1a1a1a, 0.005);

  // Camera
  camera = new THREE.PerspectiveCamera(
    45,
    window.innerWidth / window.innerHeight,
    0.1,
    1000,
  );
  camera.position.set(0, 40, 40);
  camera.lookAt(0, 0, 0);

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value!,
    antialias: true,
    alpha: true,
  });
  renderer.shadowMap.enabled = true;

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(10, 20, 10);
  dirLight.castShadow = true;
  scene.add(dirLight);

  // Grid
  const gridHelper = new THREE.GridHelper(100, 20, 0x444444, 0x222222);
  scene.add(gridHelper);

  // Nucleus (Gold?) - Fixed at origin
  const nucleusGeo = new THREE.SphereGeometry(1.5, 32, 32);
  const nucleusMat = new THREE.MeshStandardMaterial({
    color: 0xffd700,
    emissive: 0xaa6600,
    emissiveIntensity: 0.2,
    roughness: 0.2,
    metalness: 0.8,
  });
  nucleusMesh = new THREE.Mesh(nucleusGeo, nucleusMat);
  nucleusMesh.position.set(0, 0, 0);
  nucleusMesh.castShadow = true;
  scene.add(nucleusMesh);

  // Nucleus Label or Glow could be added, but keep simple

  // Alpha Particle
  const alphaGeo = new THREE.SphereGeometry(0.5, 16, 16);
  const alphaMat = new THREE.MeshStandardMaterial({
    color: 0xff3333,
    emissive: 0xff0000,
    emissiveIntensity: 0.5,
  });
  alphaMesh = new THREE.Mesh(alphaGeo, alphaMat);
  alphaMesh.castShadow = true;
  scene.add(alphaMesh);

  // Trail
  const trailGeo = new THREE.BufferGeometry().setFromPoints(trailPoints);
  const trailMat = new THREE.LineBasicMaterial({ color: 0xffaaaa });
  trailLine = new THREE.Line(trailGeo, trailMat);
  scene.add(trailLine);

  updateAlphaVisualsFromParams();
}

function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, 0, 0); // No gravity, only electrostatic force
  world.broadphase = new CANNON.NaiveBroadphase();

  // Alpha Body
  const shape = new CANNON.Sphere(0.5);
  alphaBody = new CANNON.Body({ mass: particleMass });
  alphaBody.addShape(shape);
  alphaBody.linearDamping = 0; // No air resistance in space
  world.addBody(alphaBody);

  // Init position off-screen or based on params
  resetSimulationState();
}

function initOrbitControls() {
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.minDistance = 5;
  controls.maxDistance = 200;
}

function initChart() {
  if (!chartCanvas.value) return;

  chartInstance = new Chart(chartCanvas.value, {
    type: "scatter",
    data: {
      datasets: [
        {
          label: "Experimental Data",
          data: [],
          backgroundColor: "rgb(255, 99, 132)",
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      scales: {
        x: {
          title: { display: true, text: "Impact Parameter (b)" },
          type: "linear",
          min: 0,
        },
        y: {
          title: { display: true, text: "Scattering Angle (deg)" },
          min: 0,
          max: 180,
        },
      },
      plugins: {
        tooltip: {
          callbacks: {
            label: (context) => {
              const x = context.parsed.x as number;
              const y = context.parsed.y as number;
              return `b: ${x.toFixed(2)}, θ: ${y.toFixed(1)}°`;
            },
          },
        },
      },
    },
  });
}

// --- Simulation Logic ---

function updateAlphaVisualsFromParams() {
  // Start position: far left (-x), offset by b (y or z - let's use z for impact parameter to look nice from side?)
  // Actually, standard 2D view is usually X-Y plane. Let's start at x = -50, y = b, z = 0.
  // Wait, if nucleus is at origin, and we want to see it deflect.
  // Start: x = -50, y = impactParameter. Velocity: +x.

  if (!isSimulating.value) {
    alphaMesh.position.set(-50, impactParameter.value, 0);
    // Clear trail
    trailPoints = [];
    if (trailLine) trailLine.geometry.setFromPoints(trailPoints);
  }
}

// Watch params to update preview when not simulating
watch([impactParameter, alphaVelocity], () => {
  if (!isSimulating.value) {
    updateAlphaVisualsFromParams();
  }
});

function resetSimulationState() {
  // Reset physics body
  alphaBody.velocity.set(0, 0, 0);
  alphaBody.angularVelocity.set(0, 0, 0);
  alphaBody.position.set(-50, impactParameter.value, 0);

  // Set initial velocity
  alphaBody.velocity.set(alphaVelocity.value, 0, 0);

  // Reset visuals
  alphaMesh.position.copy(alphaBody.position as any);

  // Clear trace
  trailPoints = [new THREE.Vector3().copy(alphaMesh.position)];
  if (trailLine) trailLine.geometry.setFromPoints(trailPoints);

  simulationTime.value = 0;
  currentDistance.value = alphaMesh.position.length();
  scatteringAngle.value = null;
  dataLog.value = [];
}

function startSimulation() {
  if (isSimulating.value) return;
  resetSimulationState();
  isSimulating.value = true;
  isPaused.value = false;
}

function resetSimulation() {
  isSimulating.value = false;
  isPaused.value = false;
  resetSimulationState();
}

function togglePause() {
  isPaused.value = !isPaused.value;
}

function stepSimulation() {
  if (!isSimulating.value || isPaused.value) return;

  // 1. Calculate Electrostatic Force (Coulomb Law)
  // F = kQq / r^2 * (direction vector)
  // direction = pos / r
  // Vector F = kQq * pos / r^3

  const pos = alphaBody.position;
  const rSq = pos.lengthSquared();
  const r = Math.sqrt(rSq);

  // Avoid singularity
  if (r > 0.1) {
    const forceMag = coulombConstant.value / rSq; // F = C / r^2
    // F_vec = F_mag * (pos / r)
    // F_vec = C/r^2 * pos/r = C * pos / r^3

    // We can also just use scalar magnitude and normalize pos
    const forceVec = pos.clone().scale(forceMag / r); // Repulsive force (away from origin)

    // Apply force
    alphaBody.force.copy(forceVec);
  }

  // 2. Step Physics
  // Using fixed time step for consistency, but accumulated time for rendering could be better.
  // For this simple simulation, stepping once per frame is usually fine if fps is stable.
  world.step(timeStep);

  // 3. Update State
  simulationTime.value += timeStep;
  currentDistance.value = r;
  currentSpeed.value = alphaBody.velocity.length();

  // 4. Update Visuals
  alphaMesh.position.copy(alphaBody.position as any);

  // Trail
  if (simulationTime.value % 0.05 < 0.01) {
    // Optimize trail points
    trailPoints.push(new THREE.Vector3().copy(alphaMesh.position));
    if (trailLine) trailLine.geometry.setFromPoints(trailPoints);
  }

  // 5. Log Data
  logFrame();

  // 6. Check Completion
  // Stop if particle is far away AND moving away
  if (r > 60 && pos.x > 0) {
    // Passed the nucleus and far enough
    calculateResults();
  }
}

function calculateResults() {
  isSimulating.value = false; // Stop sim

  // Calculate final angle
  // Initial velocity vector: (v0, 0, 0)
  // Final velocity vector: alphaBody.velocity
  // cos(theta) = (v_in . v_out) / (|v_in| |v_out|)
  const vFinal = alphaBody.velocity;
  const vInitial = new CANNON.Vec3(alphaVelocity.value, 0, 0);

  // Dot product
  const dot = vInitial.dot(vFinal);
  const magIn = vInitial.length();
  const magOut = vFinal.length(); // Should be same as magIn by conservation of energy (if integration is accurate)

  // Safety
  let cosTheta = dot / (magIn * magOut);
  cosTheta = Math.min(1, Math.max(-1, cosTheta));

  const angleRad = Math.acos(cosTheta);
  const angleDeg = angleRad * (180 / Math.PI);

  scatteringAngle.value = angleDeg;

  // Add to chart
  addToChart(impactParameter.value, angleDeg);
}

// --- Charting ---
function addToChart(b: number, theta: number) {
  // Check if point exists strictly? No, let user add multiples.
  // Sort logic maybe?
  chartDataPoints.value.push({ x: b, y: theta });

  // Sort for nicer Line connection? Scatter doesn't need sort, but if we want to see a trend...
  // Let's keep it unsorted or sort by X
  chartDataPoints.value.sort((a, b) => a.x - b.x);

  updateChart();
}

function updateChart() {
  if (!chartInstance) return;
  chartInstance.data.datasets[0].data = chartDataPoints.value;
  chartInstance.update();
}

function clearChart() {
  chartDataPoints.value = [];
  scatteringAngle.value = null;
  updateChart();
}

// --- Logging ---
function logFrame() {
  dataLog.value.push({
    time: simulationTime.value.toFixed(3),
    x: alphaBody.position.x.toFixed(3),
    y: alphaBody.position.y.toFixed(3),
    z: alphaBody.position.z.toFixed(3),
    distance: currentDistance.value.toFixed(3),
    speed: currentSpeed.value.toFixed(3),
    kineticEnergy: (
      0.5 *
      particleMass *
      Math.pow(currentSpeed.value, 2)
    ).toFixed(3),
    potentialEnergy: (coulombConstant.value / currentDistance.value).toFixed(3),
  });
}

function exportLogsToCSV() {
  if (dataLog.value.length === 0) return;

  const headers = Object.keys(dataLog.value[0]).join(",");
  const rows = dataLog.value.map((row) => Object.values(row).join(","));
  const csvContent =
    "data:text/csv;charset=utf-8," + [headers, ...rows].join("\n");

  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", "rutherford_simulation_data.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function exportChartCSV() {
  if (chartDataPoints.value.length === 0) return;
  const csvContent =
    "data:text/csv;charset=utf-8," +
    "Impact Parameter (b),Scattering Angle (deg)\n" +
    chartDataPoints.value.map((p) => `${p.x},${p.y}`).join("\n");

  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", "scattering_results.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

// --- Loop ---
function animate() {
  animationId = requestAnimationFrame(animate);

  stepSimulation();
  controls.update(); // Required for damping
  renderer.render(scene, camera);
}

// --- Layout / Resize ---
let isResizing = false;
function startResize(e: MouseEvent) {
  isResizing = true;
  document.addEventListener("mousemove", handleMouseMove);
  document.addEventListener("mouseup", stopResize);
  document.body.style.cursor = "col-resize";
}

function handleMouseMove(e: MouseEvent) {
  if (!isResizing) return;
  const containerRect = mainContainer.value!.getBoundingClientRect();
  const rawNewWidth = containerRect.right - e.clientX;

  // Constraints
  const minCanvasWidth = 300;
  const maxSidebar = containerRect.width - minCanvasWidth;

  sidebarWidth.value = Math.max(260, Math.min(rawNewWidth, maxSidebar, 600));
  handleResize(); // Force camera update
}

function stopResize() {
  isResizing = false;
  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopResize);
  document.body.style.cursor = "";
}

function handleResize() {
  if (!canvasContainer.value || !renderer || !camera) return;

  const width = canvasContainer.value.clientWidth;
  const height = canvasContainer.value.clientHeight;

  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
}
</script>

<style scoped>
/* Scrollbar Styling for Webkit */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}
::-webkit-scrollbar-track {
  background: transparent;
}
::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 4px;
}
::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}

.range-xs {
  height: 1.25rem;
}
</style>
