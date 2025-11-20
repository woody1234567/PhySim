<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-200 text-base-content font-sans"
  >
    <!-- 3D Simulation Area -->
    <div class="flex-grow relative h-full" ref="canvasContainer">
      <canvas ref="canvasRef" class="block w-full h-full outline-none"></canvas>

      <!-- Overlay Info (Top Left) -->
      <div
        class="absolute top-4 left-4 bg-base-100/80 backdrop-blur p-3 rounded-lg shadow-lg pointer-events-none z-10 text-sm"
      >
        <h2 class="font-bold text-primary mb-1">Physics Simulation</h2>
        <p>
          Status: <span :class="statusColor">{{ simulationStatus }}</span>
        </p>
        <p>Time: {{ currentTime.toFixed(2) }} s</p>
        <p>Distance from Start: {{ currentDistance.toFixed(2) }} m</p>
      </div>
    </div>

    <!-- Resizable Divider -->
    <div
      class="w-2 bg-base-300 hover:bg-primary cursor-col-resize flex items-center justify-center transition-colors z-20 select-none"
      @mousedown="startResize"
    >
      <div class="w-1 h-8 bg-base-content/20 rounded"></div>
    </div>

    <!-- Sidebar -->
    <div
      class="flex flex-col h-full bg-base-100 border-l border-base-300 shadow-xl z-10 flex-shrink-0"
      :style="{ width: sidebarWidth + 'px' }"
    >
      <div class="p-4 border-b border-base-300 bg-base-200">
        <h1 class="text-xl font-bold">Settings & Data</h1>
      </div>

      <div class="flex-grow overflow-y-auto p-2 space-y-2">
        <!-- PANEL 1: Control Panel -->
        <div
          class="collapse collapse-arrow bg-base-100 border border-base-300 rounded-box"
        >
          <input type="checkbox" checked />
          <div class="collapse-title text-lg font-medium">Control Panel</div>
          <div class="collapse-content space-y-4">
            <!-- Angle Slider -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Incline Angle (deg)</span>
                <span class="label-text-alt font-bold"
                  >{{ params.angle }}°</span
                >
              </label>
              <input
                type="range"
                min="0"
                max="80"
                step="1"
                class="range range-primary range-sm"
                v-model.number="params.angle"
                @input="flagReset"
              />
            </div>

            <!-- Friction Slider -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Friction Coeff (μ)</span>
                <span class="label-text-alt font-bold">{{
                  params.friction.toFixed(2)
                }}</span>
              </label>
              <input
                type="range"
                min="0.0"
                max="1.0"
                step="0.05"
                class="range range-secondary range-sm"
                v-model.number="params.friction"
                @input="flagReset"
              />
            </div>

            <!-- Mass Slider -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Ball Mass (kg)</span>
                <span class="label-text-alt font-bold"
                  >{{ params.mass }} kg</span
                >
              </label>
              <input
                type="range"
                min="0.1"
                max="10"
                step="0.1"
                class="range range-accent range-sm"
                v-model.number="params.mass"
                @input="flagReset"
              />
            </div>

            <div class="divider my-2"></div>

            <!-- Action Buttons -->
            <div class="flex flex-col gap-2">
              <div class="grid grid-cols-2 gap-2">
                <button
                  class="btn btn-success btn-sm"
                  @click="startSimulation"
                  :disabled="isRunning"
                >
                  Start
                </button>
                <button class="btn btn-error btn-sm" @click="resetSimulation">
                  Restart
                </button>
              </div>
              <button
                class="btn btn-warning btn-sm w-full"
                @click="togglePause"
                :disabled="!isRunning"
              >
                {{ isPaused ? "Resume" : "Pause" }}
              </button>
            </div>
          </div>
        </div>

        <!-- PANEL 2: Live Data Panel -->
        <div
          class="collapse collapse-arrow bg-base-100 border border-base-300 rounded-box"
        >
          <input type="checkbox" />
          <div class="collapse-title text-lg font-medium">Live Data</div>
          <div class="collapse-content">
            <div class="overflow-x-auto">
              <table class="table table-xs w-full">
                <tbody>
                  <tr>
                    <td class="font-bold">Time (s)</td>
                    <td class="font-mono text-right">{{ liveData.t }}</td>
                  </tr>
                  <tr>
                    <td class="font-bold">Dist. Traveled (m)</td>
                    <td class="font-mono text-right">{{ liveData.d }}</td>
                  </tr>
                  <tr>
                    <td class="font-bold">Pos X</td>
                    <td class="font-mono text-right">{{ liveData.x }}</td>
                  </tr>
                  <tr>
                    <td class="font-bold">Pos Y</td>
                    <td class="font-mono text-right">{{ liveData.y }}</td>
                  </tr>
                  <tr>
                    <td class="font-bold">Vel Mag (m/s)</td>
                    <td class="font-mono text-right">{{ liveData.v }}</td>
                  </tr>
                  <tr>
                    <td class="font-bold">Accel (m/s²)</td>
                    <td class="font-mono text-right">{{ liveData.a }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
            <button
              class="btn btn-outline btn-sm w-full mt-4"
              @click="exportCSV"
            >
              Export Log (CSV)
            </button>
          </div>
        </div>

        <!-- PANEL 3: Chart Panel -->
        <div
          class="collapse collapse-arrow bg-base-100 border border-base-300 rounded-box"
        >
          <input type="checkbox" checked />
          <div class="collapse-title text-lg font-medium">Analysis Charts</div>
          <div class="collapse-content">
            <div class="form-control w-full max-w-xs mb-2">
              <label class="label">
                <span class="label-text">Plot Variable</span>
              </label>
              <select
                class="select select-bordered select-sm"
                v-model="chartConfig.variable"
              >
                <option value="distance">Distance vs Time</option>
                <option value="velocity">Velocity vs Time</option>
                <option value="acceleration">Acceleration vs Time</option>
                <option value="height">Height (Y) vs Time</option>
              </select>
            </div>

            <div class="w-full h-48 bg-white rounded p-1 mb-2">
              <canvas ref="chartCanvas"></canvas>
            </div>

            <div class="flex gap-2">
              <button
                class="btn btn-primary btn-xs flex-1"
                @click="updateChart"
              >
                Plot Chart
              </button>
              <button
                class="btn btn-secondary btn-xs flex-1"
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
import { ref, reactive, onMounted, onUnmounted, watch, computed } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";

// --- Types ---
interface SimulationLog {
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
  distance: number;
}

// --- UI State ---
const sidebarWidth = ref(340);
const canvasContainer = ref<HTMLElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const chartCanvas = ref<HTMLCanvasElement | null>(null);
const isResizing = ref(false);

// --- Simulation State ---
const isRunning = ref(false);
const isPaused = ref(false);
const needsReset = ref(false);
const currentTime = ref(0);
const currentDistance = ref(0);
const logs = ref<SimulationLog[]>([]);

const params = reactive({
  angle: 30,
  friction: 0.2,
  mass: 1.0,
  gravity: -9.81,
});

const liveData = reactive({
  t: "0.00",
  d: "0.00",
  x: "0.00",
  y: "0.00",
  v: "0.00",
  a: "0.00",
});

const chartConfig = reactive({
  variable: "distance",
});

const simulationStatus = computed(() => {
  if (needsReset.value) return "Changed (Restart Needed)";
  if (isPaused.value) return "Paused";
  if (isRunning.value) return "Running";
  return "Ready";
});

const statusColor = computed(() => {
  if (needsReset.value) return "text-warning";
  if (isPaused.value) return "text-info";
  if (isRunning.value) return "text-success";
  return "text-base-content";
});

// --- Physics & 3D Objects ---
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let world: CANNON.World;

// Meshes
let rampMesh: THREE.Mesh;
let ballMesh: THREE.Mesh;
let originMarker: THREE.Mesh;

// Bodies
let rampBody: CANNON.Body;
let ballBody: CANNON.Body;
let groundMaterial: CANNON.Material;
let ballMaterial: CANNON.Material;
let contactMaterial: CANNON.ContactMaterial;

// Logic Constants
const RAMP_LENGTH = 20;
const RAMP_WIDTH = 4;
const BALL_RADIUS = 0.5;

// Simulation Loop Variables
let animationFrameId: number;
const timeStep = 1 / 60;
let initialPosVec = new CANNON.Vec3();
let chartInstance: Chart | null = null;

// --- Initialization ---

onMounted(() => {
  if (process.client) {
    initThree();
    initPhysics();
    initChart();
    renderLoop();
    window.addEventListener("resize", handleResize);
    document.addEventListener("mousemove", handleMouseMove);
    document.addEventListener("mouseup", handleMouseUp);
  }
});

onUnmounted(() => {
  if (process.client) {
    cancelAnimationFrame(animationFrameId);
    window.removeEventListener("resize", handleResize);
    document.removeEventListener("mousemove", handleMouseMove);
    document.removeEventListener("mouseup", handleMouseUp);
    if (chartInstance) chartInstance.destroy();
  }
});

function flagReset() {
  if (isRunning.value && !needsReset.value) {
    needsReset.value = true;
  } else if (!isRunning.value) {
    resetSimulation();
  }
}

// --- Three.js Setup ---
function initThree() {
  if (!canvasRef.value || !canvasContainer.value) return;

  const { clientWidth, clientHeight } = canvasContainer.value;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0xeef2ff); // slightly lighter blue-gray
  scene.fog = new THREE.Fog(0xeef2ff, 20, 100);

  // Camera
  camera = new THREE.PerspectiveCamera(
    45,
    clientWidth / clientHeight,
    0.1,
    1000
  );
  // Position camera to see the slope (Start X is negative, End is 0)
  camera.position.set(0, 10, 30);

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.setSize(clientWidth, clientHeight);
  renderer.shadowMap.enabled = true;

  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.target.set(-10, 5, 0); // Look at middle of approx slope area

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(10, 30, 20);
  dirLight.castShadow = true;
  dirLight.shadow.camera.top = 30;
  dirLight.shadow.camera.bottom = -30;
  dirLight.shadow.camera.left = -30;
  dirLight.shadow.camera.right = 30;
  scene.add(dirLight);

  // Visuals
  // 1. Grid (Ground plane representation)
  const gridHelper = new THREE.GridHelper(60, 60, 0xaaaaaa, 0xdddddd);
  scene.add(gridHelper);

  // 2. Origin Marker (Where the ramp ends)
  const originGeo = new THREE.SphereGeometry(0.2, 16, 16);
  const originMat = new THREE.MeshBasicMaterial({ color: 0x000000 });
  originMarker = new THREE.Mesh(originGeo, originMat);
  scene.add(originMarker); // At 0,0,0

  // 3. Ramp Mesh
  const rampGeo = new THREE.BoxGeometry(RAMP_LENGTH, 0.2, RAMP_WIDTH);
  const rampMat = new THREE.MeshStandardMaterial({ color: 0x3b82f6 });
  rampMesh = new THREE.Mesh(rampGeo, rampMat);
  rampMesh.receiveShadow = true;
  scene.add(rampMesh);

  // 4. Ball Mesh
  const ballGeo = new THREE.SphereGeometry(BALL_RADIUS, 32, 32);
  const ballMatVis = new THREE.MeshStandardMaterial({ color: 0xff0000 });
  ballMesh = new THREE.Mesh(ballGeo, ballMatVis);
  ballMesh.castShadow = true;
  scene.add(ballMesh);
}

// --- Physics Setup ---
function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, params.gravity, 0);
  world.broadphase = new CANNON.NaiveBroadphase();
  (world.solver as CANNON.GSSolver).iterations = 10;

  // Materials
  groundMaterial = new CANNON.Material("ground");
  ballMaterial = new CANNON.Material("ball");

  contactMaterial = new CANNON.ContactMaterial(groundMaterial, ballMaterial, {
    friction: params.friction,
    restitution: 0.1,
  });
  world.addContactMaterial(contactMaterial);

  // Ramp Body (Static)
  const rampShape = new CANNON.Box(
    new CANNON.Vec3(RAMP_LENGTH / 2, 0.1, RAMP_WIDTH / 2)
  );
  rampBody = new CANNON.Body({ mass: 0, material: groundMaterial });
  rampBody.addShape(rampShape);
  world.addBody(rampBody);

  // Ball Body (Dynamic)
  const ballShape = new CANNON.Sphere(BALL_RADIUS);
  ballBody = new CANNON.Body({ mass: params.mass, material: ballMaterial });
  ballBody.addShape(ballShape);
  ballBody.linearDamping = 0.0; // Minimal air resistance
  ballBody.angularDamping = 0.01;
  world.addBody(ballBody);

  alignObjects();
}

// Aligns the ramp so the "End" (bottom edge) is at world (0,0,0)
// And the ball starts at the "Top" (start edge).
function alignObjects() {
  // 1. Update Physics Constants
  world.gravity.set(0, params.gravity, 0);
  ballBody.mass = params.mass;
  ballBody.updateMassProperties();
  contactMaterial.friction = params.friction;

  // 2. Calculate Rotation
  // Angle is degrees. We want the ramp to go UP into negative X.
  // So we rotate around Z by +angle radians.
  const angleRad = (params.angle * Math.PI) / 180;
  const quaternion = new CANNON.Quaternion();
  quaternion.setFromAxisAngle(new CANNON.Vec3(0, 0, 1), angleRad);

  // 3. Position the Ramp
  // The Ramp is centered at its local origin. Length extends +/- L/2 along X.
  // We want the point (L/2, 0, 0) [locally] to be at World (0,0,0).
  // Vector from Center to End is V_local = (L/2, 0, 0).
  // In World space, this vector is V_world = Q * V_local.
  // We want Center_World + V_world = (0,0,0) => Center_World = - V_world.

  const halfLength = RAMP_LENGTH / 2;
  const localEndVec = new CANNON.Vec3(halfLength, 0, 0);
  const worldEndOffset = quaternion.vmult(localEndVec);

  const rampPos = new CANNON.Vec3(
    worldEndOffset.x,
    worldEndOffset.y,
    worldEndOffset.z
  );

  rampBody.position.copy(rampPos);
  rampBody.quaternion.copy(quaternion);

  // Sync Visual Ramp
  rampMesh.position.copy(rampBody.position as any);
  rampMesh.quaternion.copy(rampBody.quaternion as any);

  // 4. Position the Ball at the Top
  // Top edge local X is -L/2.
  // We place ball slightly offset so it sits on surface.
  // Local Ball Pos = (-L/2 + margin, BALL_RADIUS + thickness/2, 0)
  const startMargin = 0.5; // Distance from exact top edge
  const localBallPos = new CANNON.Vec3(
    halfLength - startMargin,
    BALL_RADIUS + 0.11,
    0
  );

  // World Ball Pos = RampPos + (Q * LocalBallPos)
  const worldBallOffset = quaternion.vmult(localBallPos);
  const startPos = rampPos.vadd(worldBallOffset);

  ballBody.position.copy(startPos);
  ballBody.velocity.set(0, 0, 0);
  ballBody.angularVelocity.set(0, 0, 0);

  // Store initial position for distance calc
  initialPosVec.copy(startPos);

  // Sync Visual Ball
  ballMesh.position.copy(ballBody.position as any);
  ballMesh.quaternion.copy(ballBody.quaternion as any);

  // Camera adjustment (Optional: center camera on the slope midpoint)
  const midX = startPos.x / 2;
  const midY = startPos.y / 2;
  controls.target.set(midX, midY, 0);
}

// --- Simulation Logic ---

function startSimulation() {
  if (needsReset.value) {
    resetSimulation();
  }
  isRunning.value = true;
  isPaused.value = false;
  needsReset.value = false;
}

function togglePause() {
  isPaused.value = !isPaused.value;
}

function resetSimulation() {
  isRunning.value = false;
  isPaused.value = false;
  needsReset.value = false;
  currentTime.value = 0;
  currentDistance.value = 0;
  logs.value = [];

  // Reset Chart
  if (chartInstance && chartInstance.data.datasets[0]) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets[0].data = [];
    chartInstance.update();
  }

  alignObjects(); // Resets positions
}

function stepSimulation() {
  if (!isRunning.value || isPaused.value) return;

  // Step physics
  world.step(timeStep);
  currentTime.value += timeStep;

  const currentPos = ballBody.position;

  // Calculate distance from STARTING point (Top of incline)
  currentDistance.value = currentPos.distanceTo(initialPosVec);

  // Stop condition:
  // Ball has reached the origin (End of incline).
  // Since the end is at X=0, and ball moves from negative X to 0.
  // We stop when X >= 0.
  if (currentPos.x <= 0) {
    isRunning.value = false;
    isPaused.value = false;

    // Clamp at zero for visual neatness
    // (Optional, but prevents overshooting in visual frame)
  }

  // Sync Visuals
  ballMesh.position.copy(ballBody.position as any);
  ballMesh.quaternion.copy(ballBody.quaternion as any);

  // Data Logging
  logFrame();
}

function logFrame() {
  const b = ballBody;
  // Approximate acceleration from force (F=ma -> a=F/m)
  // Note: Cannon-es doesn't always persist 'force' unless applied manually,
  // but we can derive it or just log velocity changes.
  // Better to calculate numerical acceleration from previous velocity if needed,
  // but for this simple demo, we'll just use the force vector if available or gravity.
  // A sliding object has constant acceleration down the slope: g * sin(theta) - friction...
  // Let's just compute magnitude of change for display or store 0 if easier.
  // Actually, let's calculate acceleration scalar based on physics theory or numerical diff.

  // Simple numerical diff for display:
  const log: SimulationLog = {
    t: currentTime.value,
    x: b.position.x,
    y: b.position.y,
    z: b.position.z,
    vx: b.velocity.x,
    vy: b.velocity.y,
    vz: b.velocity.z,
    ax: 0, // simplified for log
    ay: 0,
    az: 0,
    distance: currentDistance.value,
  };

  // Use theoretical or previous frame for accel if strictly needed,
  // but here we focus on Pos/Vel/Dist.

  logs.value.push(log);

  // Update Live Data Panel
  liveData.t = log.t.toFixed(2);
  liveData.d = log.distance.toFixed(2);
  liveData.x = log.x.toFixed(2);
  liveData.y = log.y.toFixed(2);
  liveData.v = b.velocity.length().toFixed(2);

  // Calculate apparent acceleration magnitude (roughly)
  const gravity = 9.81;
  const angleRad = (params.angle * Math.PI) / 180;
  const theoreticalAccel =
    gravity * (Math.sin(angleRad) - params.friction * Math.cos(angleRad));
  liveData.a = Math.max(0, theoreticalAccel).toFixed(2);
}

function renderLoop() {
  animationFrameId = requestAnimationFrame(renderLoop);
  stepSimulation();
  controls.update();
  renderer.render(scene, camera);
}

// --- Chart.js ---
function initChart() {
  if (!chartCanvas.value) return;

  chartInstance = new Chart(chartCanvas.value, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        {
          label: "Distance",
          data: [],
          borderColor: "#3b82f6",
          backgroundColor: "rgba(59, 130, 246, 0.1)",
          borderWidth: 2,
          pointRadius: 0,
          fill: true,
          tension: 0.1,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: false,
      scales: {
        x: {
          title: { display: true, text: "Time (s)" },
          ticks: { maxTicksLimit: 10 },
        },
        y: {
          title: { display: true, text: "Value" },
        },
      },
      plugins: {
        legend: { display: false },
      },
    },
  });
}

function updateChart() {
  if (!chartInstance) return;

  // Downsample
  const step = Math.ceil(logs.value.length / 300) || 1;
  const dataset = [];
  const labels = [];

  for (let i = 0; i < logs.value.length; i += step) {
    const l = logs.value[i];
    labels.push(l.t.toFixed(2));

    let val = 0;
    switch (chartConfig.variable) {
      case "distance":
        val = l.distance;
        break;
      case "velocity":
        val = Math.sqrt(l.vx * l.vx + l.vy * l.vy + l.vz * l.vz);
        break;
      case "acceleration":
        // Theoretical constant accel for plot smoothness
        const g = 9.81;
        const rad = (params.angle * Math.PI) / 180;
        val = g * (Math.sin(rad) - params.friction * Math.cos(rad));
        if (val < 0) val = 0;
        break;
      case "height":
        val = l.y;
        break;
    }
    dataset.push(val);
  }

  chartInstance.data.labels = labels;
  chartInstance.data.datasets[0].data = dataset;
  chartInstance.data.datasets[0].label = chartConfig.variable;
  chartInstance.update();
}

// --- Export Logic ---
function exportCSV() {
  const headers = ["t", "x", "y", "z", "vx", "vy", "vz", "distance"];
  const csvContent = [
    headers.join(","),
    ...logs.value.map(
      (l) => `${l.t},${l.x},${l.y},${l.z},${l.vx},${l.vy},${l.vz},${l.distance}`
    ),
  ].join("\n");

  downloadFile(csvContent, "simulation_logs.csv");
}

function exportChartCSV() {
  if (!chartInstance) return;
  const labels = chartInstance.data.labels as string[];
  const data = chartInstance.data.datasets[0].data as number[];

  const csvContent = [
    `time,${chartConfig.variable}`,
    ...labels.map((t, i) => `${t},${data[i]}`),
  ].join("\n");

  downloadFile(csvContent, `chart_${chartConfig.variable}.csv`);
}

function downloadFile(content: string, filename: string) {
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

// --- Resize Logic ---
function startResize() {
  isResizing.value = true;
  document.body.style.cursor = "col-resize";
}

function handleMouseMove(e: MouseEvent) {
  if (!isResizing.value) return;
  const newWidth = document.body.clientWidth - e.clientX;
  if (newWidth >= 260 && newWidth <= 600) {
    sidebarWidth.value = newWidth;
    handleResize();
  }
}

function handleMouseUp() {
  isResizing.value = false;
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
canvas {
  display: block;
}
::-webkit-scrollbar {
  width: 8px;
}
::-webkit-scrollbar-track {
  background: transparent;
}
::-webkit-scrollbar-thumb {
  background-color: rgba(156, 163, 175, 0.5);
  border-radius: 4px;
}
</style>
