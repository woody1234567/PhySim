<template>
  <div
    class="flex flex-col h-screen w-full overflow-hidden bg-base-100 text-base-content"
  >
    <!-- Main Layout: Canvas + Resizable Sidebar -->
    <div
      class="flex flex-grow overflow-hidden relative"
      @mousemove="onResizerMove"
      @mouseup="stopResize"
    >
      <!-- Left: 3D Simulation Canvas -->
      <div
        ref="canvasContainer"
        class="flex-grow relative bg-base-300 overflow-hidden"
        :style="{ marginRight: sidebarWidth + 'px' }"
      >
        <canvas
          ref="canvasRef"
          class="block w-full h-full outline-none"
        ></canvas>

        <!-- Overlay UI -->
        <div class="absolute top-4 left-4 pointer-events-none">
          <div class="badge badge-lg badge-primary font-bold shadow-lg">
            x: {{ formattedX }} m | t: {{ formattedTime }} s
          </div>
        </div>
      </div>

      <!-- Resizer Handle -->
      <div
        class="absolute top-0 bottom-0 w-2 cursor-col-resize z-50 hover:bg-primary transition-colors flex items-center justify-center bg-base-300 border-l border-base-content/10"
        :style="{ right: sidebarWidth + 'px' }"
        @mousedown="startResize"
      >
        <div class="h-8 w-1 bg-base-content/20 rounded-full"></div>
      </div>

      <!-- Right: Sidebar -->
      <div
        class="absolute top-0 right-0 bottom-0 bg-base-100 flex flex-col border-l border-base-content/10 shadow-xl z-10"
        :style="{ width: sidebarWidth + 'px' }"
      >
        <div
          class="p-4 border-b border-base-content/10 bg-base-200/50 backdrop-blur"
        >
          <h2
            class="text-xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-primary to-secondary"
          >
            SHM Simulator
          </h2>
          <p class="text-xs opacity-60">Spring-Mass System</p>
        </div>

        <div class="flex-grow overflow-y-auto p-4 space-y-4 scrollbar-thin">
          <!-- Panel 1: Controls -->
          <div
            class="collapse collapse-arrow bg-base-200 border border-base-content/5"
          >
            <input type="radio" name="sidebar-accordion" checked="checked" />
            <div class="collapse-title text-lg font-medium">Controls</div>
            <div class="collapse-content space-y-4">
              <!-- Playback Controls -->
              <div class="grid grid-cols-2 gap-2">
                <button
                  class="btn btn-primary btn-sm"
                  @click="toggleSimulation"
                >
                  <span v-if="isRunning">Pause</span>
                  <span v-else>Resume</span>
                </button>
                <button
                  class="btn btn-ghost btn-sm border-base-content/20"
                  @click="resetSimulation"
                >
                  Reset
                </button>
              </div>

              <div class="divider my-0"></div>

              <!-- Parameters -->
              <div class="form-control w-full">
                <label class="label">
                  <span class="label-text">Mass (m)</span>
                  <span class="label-text-alt font-mono"
                    >{{ params.mass.toFixed(1) }} kg</span
                  >
                </label>
                <input
                  type="range"
                  min="0.1"
                  max="10.0"
                  step="0.1"
                  class="range range-xs range-primary"
                  v-model.number="params.mass"
                  @input="onParamsChange"
                />
              </div>

              <div class="form-control w-full">
                <label class="label">
                  <span class="label-text">Spring Constant (k)</span>
                  <span class="label-text-alt font-mono"
                    >{{ params.k.toFixed(1) }} N/m</span
                  >
                </label>
                <input
                  type="range"
                  min="0.1"
                  max="20.0"
                  step="0.1"
                  class="range range-xs range-secondary"
                  v-model.number="params.k"
                  @input="onParamsChange"
                />
              </div>

              <div class="form-control w-full">
                <label class="label">
                  <span class="label-text">Initial Displacement</span>
                  <span class="label-text-alt font-mono"
                    >{{ params.initialX.toFixed(1) }} m</span
                  >
                </label>
                <input
                  type="range"
                  min="-5.0"
                  max="5.0"
                  step="0.1"
                  class="range range-xs range-accent"
                  v-model.number="params.initialX"
                  @change="resetSimulation"
                />
              </div>
            </div>
          </div>

          <!-- Panel 2: Live Data -->
          <div
            class="collapse collapse-arrow bg-base-200 border border-base-content/5"
          >
            <input type="radio" name="sidebar-accordion" />
            <div class="collapse-title text-lg font-medium">Live Data</div>
            <div class="collapse-content space-y-2">
              <div class="stats stats-vertical shadow w-full bg-base-100/50">
                <div class="stat p-2">
                  <div class="stat-title text-xs">Position (x)</div>
                  <div class="stat-value text-lg font-mono">
                    {{ currentData.position.toFixed(3) }} m
                  </div>
                </div>
                <div class="stat p-2">
                  <div class="stat-title text-xs">Velocity (v)</div>
                  <div class="stat-value text-lg font-mono">
                    {{ currentData.velocity.toFixed(3) }} m/s
                  </div>
                </div>
                <div class="stat p-2">
                  <div class="stat-title text-xs">Acceleration (a)</div>
                  <div class="stat-value text-lg font-mono">
                    {{ currentData.acceleration.toFixed(3) }} m/s²
                  </div>
                </div>
                <div class="stat p-2">
                  <div class="stat-title text-xs">Force (F)</div>
                  <div class="stat-value text-lg font-mono">
                    {{ currentData.force.toFixed(3) }} N
                  </div>
                </div>
                <div class="stat p-2">
                  <div class="stat-title text-xs">Total Energy</div>
                  <div class="stat-value text-lg font-mono">
                    {{ currentData.energy.toFixed(3) }} J
                  </div>
                </div>
              </div>

              <button
                class="btn btn-outline btn-xs w-full mt-2"
                @click="exportLogsToCSV"
              >
                Export Simulation Data (CSV)
              </button>
            </div>
          </div>

          <!-- Panel 3: Charts -->
          <div
            class="collapse collapse-arrow bg-base-200 border border-base-content/5"
          >
            <input type="radio" name="sidebar-accordion" />
            <div class="collapse-title text-lg font-medium">Charts</div>
            <div class="collapse-content space-y-4">
              <div class="flex gap-2 justify-center">
                <button
                  class="btn btn-xs"
                  :class="
                    activeChart === 'x'
                      ? 'btn-active btn-primary'
                      : 'btn-outline'
                  "
                  @click="
                    activeChart = 'x';
                    updateChart();
                  "
                >
                  x-t
                </button>
                <button
                  class="btn btn-xs"
                  :class="
                    activeChart === 'v'
                      ? 'btn-active btn-secondary'
                      : 'btn-outline'
                  "
                  @click="
                    activeChart = 'v';
                    updateChart();
                  "
                >
                  v-t
                </button>
                <button
                  class="btn btn-xs"
                  :class="
                    activeChart === 'a'
                      ? 'btn-active btn-accent'
                      : 'btn-outline'
                  "
                  @click="
                    activeChart = 'a';
                    updateChart();
                  "
                >
                  a-t
                </button>
              </div>

              <div
                class="w-full bg-base-100 p-2 rounded-lg border border-base-content/10 h-48"
              >
                <canvas ref="chartCanvasRef"></canvas>
              </div>

              <div class="grid grid-cols-2 gap-2">
                <button
                  class="btn btn-sm btn-primary w-full"
                  @click="updateChart"
                >
                  Update Plot
                </button>
                <button
                  class="btn btn-sm btn-outline w-full"
                  @click="exportChartCSV"
                >
                  Export CSV
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- Footer info -->
        <div class="p-2 text-center text-[10px] opacity-40">
          Physics Engine: Cannon-es | Render: Three.js
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
/**
 * Simple Harmonic Motion Simulation
 *
 * Demonstrates F = -kx using Cannon.js and Three.js
 * Visualizes a block on a frictionless surface attached to a spring.
 */

import {
  ref,
  onMounted,
  onUnmounted,
  reactive,
  computed,
  watch,
  nextTick,
} from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";

/* -------------------------------------------------------------------------- */
/*                                    STATE                                   */
/* -------------------------------------------------------------------------- */

// Physics Parameters
const params = reactive({
  mass: 2.0,
  k: 5.0,
  initialX: 3.0,
});

// Current System State (for Live Data)
const currentData = reactive({
  time: 0,
  position: 0,
  velocity: 0,
  acceleration: 0,
  force: 0,
  energy: 0,
});

const isRunning = ref(false);
const sidebarWidth = ref(320);
const minSidebarWidth = 260;
const minCanvasWidth = 300;
let isResizing = false;

// Data Logging
interface LogEntry {
  time: number;
  x: number;
  v: number;
  a: number;
  ke: number;
  pe: number;
}
const dataLog = ref<LogEntry[]>([]);
const MAX_LOG_ENTRIES = 1000; // Limit for memory safety

/* -------------------------------------------------------------------------- */
/*                                THREE.JS REFS                               */
/* -------------------------------------------------------------------------- */
const canvasContainer = ref<HTMLElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let gridHelper: THREE.GridHelper;
let animationFrameId: number;

// Meshes
let blockMesh: THREE.Mesh;
let springLine: THREE.Line;
let wallMesh: THREE.Mesh;

/* -------------------------------------------------------------------------- */
/*                                PHYSICS REFS                                */
/* -------------------------------------------------------------------------- */
let world: CANNON.World;
let blockBody: CANNON.Body;

/* -------------------------------------------------------------------------- */
/*                                CHART.JS REFS                               */
/* -------------------------------------------------------------------------- */
const chartCanvasRef = ref<HTMLCanvasElement | null>(null);
let chartInstance: Chart | null = null;
const activeChart = ref<"x" | "v" | "a">("x");

/* -------------------------------------------------------------------------- */
/*                               COMPUTED PROPS                               */
/* -------------------------------------------------------------------------- */
const formattedX = computed(() => currentData.position.toFixed(2));
const formattedTime = computed(() => currentData.time.toFixed(1));

/* -------------------------------------------------------------------------- */
/*                               LIFECYCLE HOOKS                              */
/* -------------------------------------------------------------------------- */
onMounted(() => {
  if (process.client) {
    initThree();
    initPhysics();
    initOrbitControls();
    initChart();
    resetSimulation();
    startAnimationLoop();

    window.addEventListener("resize", onWindowResize);
  }
});

onUnmounted(() => {
  if (process.client) {
    cancelAnimationFrame(animationFrameId);
    window.removeEventListener("resize", onWindowResize);

    if (renderer) renderer.dispose();
    if (chartInstance) chartInstance.destroy();
  }
});

/* -------------------------------------------------------------------------- */
/*                               INITIALIZATION                               */
/* -------------------------------------------------------------------------- */
function initThree() {
  if (!canvasRef.value || !canvasContainer.value) return;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);
  scene.fog = new THREE.Fog(0x111111, 10, 50);

  // Camera
  const aspect =
    canvasContainer.value.clientWidth / canvasContainer.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 100);
  camera.position.set(0, 5, 10);
  camera.lookAt(0, 0, 0);

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.shadowMap.enabled = true;
  renderer.setSize(
    canvasContainer.value.clientWidth,
    canvasContainer.value.clientHeight,
  );
  renderer.setPixelRatio(window.devicePixelRatio);

  // Lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(5, 10, 7);
  dirLight.castShadow = true;
  dirLight.shadow.mapSize.width = 1024;
  dirLight.shadow.mapSize.height = 1024;
  scene.add(dirLight);

  // Grid
  gridHelper = new THREE.GridHelper(20, 20, 0x444444, 0x222222);
  scene.add(gridHelper);

  // Wall (Visual Anchor)
  const wallGeom = new THREE.BoxGeometry(0.2, 2, 2);
  const wallMat = new THREE.MeshStandardMaterial({ color: 0x555555 });
  wallMesh = new THREE.Mesh(wallGeom, wallMat);
  wallMesh.position.set(-6, 1, 0);
  wallMesh.castShadow = true;
  wallMesh.receiveShadow = true;
  scene.add(wallMesh);

  // Block
  const blockGeom = new THREE.BoxGeometry(1, 1, 1);
  const blockMat = new THREE.MeshStandardMaterial({
    color: 0x3b82f6,
    roughness: 0.2,
    metalness: 0.1,
  });
  blockMesh = new THREE.Mesh(blockGeom, blockMat);
  blockMesh.castShadow = true;
  blockMesh.receiveShadow = true;
  scene.add(blockMesh);

  // Spring Visual (ZigZag Line)
  // We'll update geometry in render loop
  const springPoints = [];
  for (let i = 0; i < 20; i++) {
    springPoints.push(new THREE.Vector3(0, 0, 0));
  }
  const springGeom = new THREE.BufferGeometry().setFromPoints(springPoints);
  const springMat = new THREE.LineBasicMaterial({
    color: 0xef4444,
    linewidth: 2,
  });
  springLine = new THREE.Line(springGeom, springMat);
  scene.add(springLine);
}

function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, 0, 0); // No gravity for horizontal SHM
  world.broadphase = new CANNON.NaiveBroadphase();

  // Physics Material
  const material = new CANNON.Material("frictionless");

  // Block Body
  const shape = new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5));
  blockBody = new CANNON.Body({
    mass: params.mass,
    material: material,
  });
  blockBody.addShape(shape);
  blockBody.linearDamping = 0.0; // No damping for ideal SHM
  world.addBody(blockBody);

  // Setup Force Application (The "Spring")
  world.addEventListener("postStep", () => {
    // F = -kx
    // Force is applied to the center of mass
    const x = blockBody.position.x;
    const f = -params.k * x;
    blockBody.force.x += f;

    // Derived values for chart/log
    const a = f / params.mass;
    currentData.acceleration = a;
    currentData.force = f;
  });
}

function initOrbitControls() {
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
}

function initChart() {
  if (!chartCanvasRef.value) return;

  chartInstance = new Chart(chartCanvasRef.value, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        {
          label: "Position (m)",
          data: [],
          borderColor: "#3b82f6",
          tension: 0.1,
          pointRadius: 0,
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
          grid: { color: "rgba(255, 255, 255, 0.1)" },
          ticks: { color: "#888" },
        },
        y: {
          title: { display: true, text: "Value" },
          grid: { color: "rgba(255, 255, 255, 0.1)" },
          ticks: { color: "#888" },
        },
      },
      plugins: {
        legend: { labels: { color: "#ccc" } },
      },
    },
  });
}

/* -------------------------------------------------------------------------- */
/*                             SIMULATION LOGIC                               */
/* -------------------------------------------------------------------------- */

function resetSimulation() {
  isRunning.value = false;

  // Reset Physics
  blockBody.position.set(params.initialX, 0.5, 0);
  blockBody.velocity.set(0, 0, 0);
  blockBody.angularVelocity.set(0, 0, 0);
  blockBody.force.set(0, 0, 0);

  // Update mass if changed
  blockBody.mass = params.mass;
  blockBody.updateMassProperties();

  // Reset Time
  currentData.time = 0;

  // Clear Logs
  dataLog.value = [];

  // Reset Charts
  if (chartInstance) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets[0].data = [];
    chartInstance.update();
  }

  // Initial Sync
  syncVisuals();
}

function toggleSimulation() {
  isRunning.value = !isRunning.value;
}

function stepSimulation() {
  if (!isRunning.value) return;

  const dt = 1 / 60;

  // Manual force application is handled in 'postStep' hook logic
  // essentially, but we can also do it pre-step here to be explicit
  // However, we used postStep event or preStep.
  // Let's stick to doing it right before step for clarity:
  blockBody.force.set(0, 0, 0); // Clear previous forces
  const x = blockBody.position.x;
  const f = -params.k * x;
  blockBody.force.x = f;

  world.step(dt);
  currentData.time += dt;

  logFrame(dt);
}

function logFrame(dt: number) {
  // Update Current Data
  currentData.position = blockBody.position.x;
  currentData.velocity = blockBody.velocity.x;
  currentData.acceleration = blockBody.force.x / blockBody.mass; // F=ma
  currentData.force = blockBody.force.x;

  // Energy
  const ke = 0.5 * params.mass * blockBody.velocity.x ** 2;
  const pe = 0.5 * params.k * blockBody.position.x ** 2;
  currentData.energy = ke + pe;

  // Add to Log (throttle to every 5 frames to save memory if needed, or keeping it tight for now)
  // For charts, simpler to log every frame but limit total history?
  // Let's log every 10th frame roughly (0.16s resolution) for UI performance
  // Actually, let's just log every frame but verify size

  if (dataLog.value.length < MAX_LOG_ENTRIES) {
    dataLog.value.push({
      time: currentData.time,
      x: currentData.position,
      v: currentData.velocity,
      a: currentData.acceleration,
      ke,
      pe,
    });
  } else {
    // Rolling buffer maybe? Or just stop logging to keep memory safe
    // For this assignment, stopping is safer or shifting
    // dataLog.value.shift()
    // dataLog.value.push(...)
    // Let's shift to keep latest
    dataLog.value.shift();
    dataLog.value.push({
      time: currentData.time,
      x: currentData.position,
      v: currentData.velocity,
      a: currentData.acceleration,
      ke,
      pe,
    });
  }
}

function startAnimationLoop() {
  const animate = () => {
    animationFrameId = requestAnimationFrame(animate);

    // Physics
    stepSimulation();

    // Render
    syncVisuals();
    controls.update();
    renderer.render(scene, camera);

    // Note: Charts are updated manually via "Update Plot" button to maintain performance
  };
  animate();
}

function syncVisuals() {
  // Block
  blockMesh.position.copy(blockBody.position as any);
  blockMesh.quaternion.copy(blockBody.quaternion as any);

  // Spring Visual
  // Wall is at x = -6 (visual anchor)
  // Block attach point is at blockBody.position.x - 0.5 (left face of block)
  const startX = -6 + 0.1; // Just slightly off the wall center
  const endX = blockBody.position.x - 0.5;

  if (springLine) {
    const points = [];
    const coils = 20;
    const radius = 0.3;
    const distance = endX - startX;

    // Generate zigzag or helix vertices
    for (let i = 0; i <= coils; i++) {
      const t = i / coils;
      const x = startX + distance * t;
      const y = Math.sin(t * Math.PI * 10) * radius + 1; // +1 for height offset (center of block is y=0.5, so height 1 is wrong? Block py=0.5. Spring should be at y=0.5)
      // Wait, block y is 0.5 (resting on ground). So center is y=0.5.
      const y_spring = 0.5 + Math.sin(t * Math.PI * 2 * 5) * radius; // 5 turns
      const z_spring = Math.cos(t * Math.PI * 2 * 5) * radius;
      points.push(new THREE.Vector3(x, y_spring, z_spring));
    }
    springLine.geometry.setFromPoints(points);
  }
}

/* -------------------------------------------------------------------------- */
/*                               UI INTERACTION                               */
/* -------------------------------------------------------------------------- */

function onParamsChange() {
  // If running, we might want to apply parameters dynamically
  blockBody.mass = params.mass;
  blockBody.updateMassProperties();
}

function updateChart() {
  if (!chartInstance) return;

  const labels = dataLog.value.map((d) => d.time.toFixed(2));
  let data: number[] = [];
  let label = "";
  let color = "";

  switch (activeChart.value) {
    case "x":
      data = dataLog.value.map((d) => d.x);
      label = "Position (m)";
      color = "#3b82f6"; // Primary
      break;
    case "v":
      data = dataLog.value.map((d) => d.v);
      label = "Velocity (m/s)";
      color = "#ec4899"; // Secondary
      break;
    case "a":
      data = dataLog.value.map((d) => d.a);
      label = "Acceleration (m/s²)";
      color = "#10b981"; // Accent
      break;
  }

  chartInstance.data.labels = labels;
  chartInstance.data.datasets[0].data = data;
  chartInstance.data.datasets[0].label = label;
  chartInstance.data.datasets[0].borderColor = color;
  chartInstance.update();
}

function exportLogsToCSV() {
  const headers = [
    "Time (s)",
    "Position (m)",
    "Velocity (m/s)",
    "Acceleration (m/s²)",
    "Kinetic Energy (J)",
    "Potential Energy (J)",
  ];
  const rows = dataLog.value.map((d) => [
    d.time.toFixed(3),
    d.x.toFixed(4),
    d.v.toFixed(4),
    d.a.toFixed(4),
    d.ke.toFixed(4),
    d.pe.toFixed(4),
  ]);

  downloadCSV(headers, rows, "shm_simulation_data.csv");
}

function exportChartCSV() {
  const headers = ["Time (s)", activeChart.value];
  let rows: any[] = [];

  if (activeChart.value === "x") {
    rows = dataLog.value.map((d) => [d.time.toFixed(3), d.x.toFixed(4)]);
  } else if (activeChart.value === "v") {
    rows = dataLog.value.map((d) => [d.time.toFixed(3), d.v.toFixed(4)]);
  } else {
    rows = dataLog.value.map((d) => [d.time.toFixed(3), d.a.toFixed(4)]);
  }

  downloadCSV(headers, rows, `shm_chart_${activeChart.value}.csv`);
}

function downloadCSV(headers: string[], rows: any[][], filename: string) {
  const csvContent = [
    headers.join(","),
    ...rows.map((row) => row.join(",")),
  ].join("\n");

  const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.setAttribute("download", filename);
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

/* -------------------------------------------------------------------------- */
/*                                WINDOW RESIZE                               */
/* -------------------------------------------------------------------------- */

function onWindowResize() {
  if (canvasContainer.value && canvasRef.value) {
    // Clamp sidebar if window gets too small
    const maxSidebar = window.innerWidth - minCanvasWidth;
    if (sidebarWidth.value > maxSidebar) {
      sidebarWidth.value = maxSidebar;
    }

    const w = canvasContainer.value.clientWidth;
    const h = canvasContainer.value.clientHeight;

    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  }
}

function startResize() {
  isResizing = true;
  document.body.style.cursor = "col-resize";
  document.body.style.userSelect = "none";
  window.addEventListener("mouseup", stopResize);
  window.addEventListener("mousemove", onResizerMove);
}

function stopResize() {
  isResizing = false;
  document.body.style.cursor = "";
  document.body.style.userSelect = "";
  window.removeEventListener("mouseup", stopResize);
  window.removeEventListener("mousemove", onResizerMove);
  onWindowResize(); // Final adjustment
}

function onResizerMove(e: MouseEvent) {
  if (!isResizing) return;

  const newWidth = window.innerWidth - e.clientX;

  // Constraints
  const maxAllowed = window.innerWidth - minCanvasWidth;

  if (newWidth >= minSidebarWidth && newWidth <= Math.min(600, maxAllowed)) {
    sidebarWidth.value = newWidth;
    onWindowResize();
  }
}
</script>

<style scoped>
/* Scrollbar styling for sidebar */
.scrollbar-thin::-webkit-scrollbar {
  width: 6px;
}
.scrollbar-thin::-webkit-scrollbar-track {
  background: transparent;
}
.scrollbar-thin::-webkit-scrollbar-thumb {
  background-color: rgba(156, 163, 175, 0.3);
  border-radius: 3px;
}
.scrollbar-thin::-webkit-scrollbar-thumb:hover {
  background-color: rgba(156, 163, 175, 0.5);
}
</style>
