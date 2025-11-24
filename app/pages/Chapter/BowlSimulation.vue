<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-100 text-base-content select-none"
    @mouseup="stopResize"
    @mousemove="handleResize"
    @mouseleave="stopResize"
  >
    <!-- 3D Simulation Viewport -->
    <div ref="viewportRef" class="flex-grow relative bg-black overflow-hidden">
      <canvas ref="canvasRef" class="block w-full h-full outline-none"></canvas>

      <!-- Overlay Info -->
      <div
        class="absolute top-4 left-4 pointer-events-none bg-base-300/80 p-2 rounded text-xs font-mono"
      >
        <div>Time: {{ simulationTime.toFixed(2) }} s</div>
        <div>FPS: {{ fps }}</div>
      </div>
    </div>

    <!-- Resizer Handle -->
    <div
      class="w-2 hover:bg-primary cursor-col-resize flex-none transition-colors bg-base-300 z-10 flex items-center justify-center"
      @mousedown="startResize"
    >
      <div class="w-0.5 h-8 bg-base-content/20 rounded"></div>
    </div>

    <!-- Sidebar -->
    <div
      class="flex-none flex flex-col bg-base-200 border-l border-base-300 h-full overflow-hidden"
      :style="{ width: sidebarWidth + 'px' }"
    >
      <div
        class="p-4 font-bold text-lg border-b border-base-300 bg-base-100 shadow-sm"
      >
        Physics Lab
      </div>

      <div class="overflow-y-auto flex-1 p-2 space-y-2">
        <!-- Panel 1: Controls -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="radio" name="accordion" checked="checked" />
          <div class="collapse-title text-md font-medium">Control Panel</div>
          <div class="collapse-content space-y-4">
            <!-- Surface Params -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Curvature A (x²)</span>
                <span class="label-text-alt">{{ params.a.toFixed(3) }}</span>
              </label>
              <input
                type="range"
                min="0.01"
                max="0.2"
                step="0.01"
                v-model.number="params.a"
                class="range range-xs range-primary"
                @change="rebuildSurface"
              />
            </div>

            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Curvature B (y²)</span>
                <span class="label-text-alt">{{ params.b.toFixed(3) }}</span>
              </label>
              <input
                type="range"
                min="0.01"
                max="0.2"
                step="0.01"
                v-model.number="params.b"
                class="range range-xs range-primary"
                @change="rebuildSurface"
              />
            </div>

            <!-- Ball Params -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Initial Height (Z)</span>
                <span class="label-text-alt">{{ params.startHeight }}m</span>
              </label>
              <input
                type="range"
                min="1"
                max="10"
                step="0.5"
                v-model.number="params.startHeight"
                class="range range-xs range-secondary"
              />
            </div>

            <!-- Friction -->
            <div class="form-control">
              <label class="label cursor-pointer">
                <span class="label-text">Enable Friction</span>
                <input
                  type="checkbox"
                  v-model="params.frictionEnabled"
                  class="toggle toggle-primary"
                />
              </label>
            </div>

            <div class="form-control w-full" v-if="params.frictionEnabled">
              <label class="label">
                <span class="label-text">Friction Coeff (μ)</span>
                <span class="label-text-alt">{{ params.mu.toFixed(2) }}</span>
              </label>
              <input
                type="range"
                min="0.0"
                max="1.0"
                step="0.05"
                v-model.number="params.mu"
                class="range range-xs range-accent"
              />
            </div>

            <!-- Actions -->
            <div class="flex gap-2 pt-2">
              <button
                class="btn btn-sm btn-primary flex-1"
                @click="toggleSimulation"
              >
                {{ isRunning ? "Pause" : "Start" }}
              </button>
              <button
                class="btn btn-sm btn-outline flex-1"
                @click="resetSimulation"
              >
                Reset
              </button>
            </div>
            <button class="btn btn-sm btn-ghost w-full" @click="rebuildSurface">
              Rebuild Bowl
            </button>
          </div>
        </div>

        <!-- Panel 2: Live Data -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="radio" name="accordion" />
          <div class="collapse-title text-md font-medium">Live Data</div>
          <div class="collapse-content">
            <div class="overflow-x-auto">
              <table class="table table-xs w-full font-mono">
                <tbody>
                  <tr>
                    <td>t</td>
                    <td>{{ simulationTime.toFixed(2) }} s</td>
                  </tr>
                  <tr>
                    <td>pos</td>
                    <td>{{ formatVec(ballState.pos) }}</td>
                  </tr>
                  <tr>
                    <td>vel</td>
                    <td>{{ formatVec(ballState.vel) }}</td>
                  </tr>
                  <tr>
                    <td>acc</td>
                    <td>{{ formatVec(ballState.acc) }}</td>
                  </tr>
                  <tr>
                    <td colspan="2" class="font-bold text-center bg-base-200">
                      Energy (J)
                    </td>
                  </tr>
                  <tr>
                    <td>PE</td>
                    <td>{{ energies.pe.toFixed(3) }}</td>
                  </tr>
                  <tr>
                    <td>KE</td>
                    <td>{{ energies.ke.toFixed(3) }}</td>
                  </tr>
                  <tr v-if="params.frictionEnabled">
                    <td>TE</td>
                    <td>{{ energies.te.toFixed(3) }}</td>
                  </tr>
                  <tr>
                    <td>Total</td>
                    <td>{{ energies.total.toFixed(3) }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
            <div class="mt-4">
              <button class="btn btn-xs btn-outline w-full" @click="exportCSV">
                Export CSV
              </button>
            </div>
          </div>
        </div>

        <!-- Panel 3: Charts -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="radio" name="accordion" />
          <div class="collapse-title text-md font-medium">Charts</div>
          <div class="collapse-content">
            <div class="h-48 w-full relative">
              <canvas ref="chartCanvasRef"></canvas>
            </div>
            <div class="flex gap-2 mt-2">
              <button
                class="btn btn-xs btn-primary flex-1"
                @click="updateChart"
              >
                Plot
              </button>
              <button
                class="btn btn-xs btn-outline flex-1"
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
import { ref, onMounted, onUnmounted, reactive, watch, nextTick } from "vue";
import * as THREE from "three";
import * as CANNON from "cannon-es";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import Chart from "chart.js/auto";

// --- Types ---
type Vec3 = { x: number; y: number; z: number };

// --- State ---
const sidebarWidth = ref(320);
const isResizing = ref(false);
const viewportRef = ref<HTMLElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const chartCanvasRef = ref<HTMLCanvasElement | null>(null);

const isRunning = ref(false);
const simulationTime = ref(0);
const fps = ref(0);

// Physics Parameters
const params = reactive({
  a: 0.05,
  b: 0.05,
  startHeight: 5,
  frictionEnabled: false,
  mu: 0.1,
  mass: 1.0,
  radius: 0.5,
});

// Real-time State
const ballState = reactive({
  pos: { x: 0, y: 0, z: 0 } as Vec3,
  vel: { x: 0, y: 0, z: 0 } as Vec3,
  acc: { x: 0, y: 0, z: 0 } as Vec3,
});

const energies = reactive({
  pe: 0,
  ke: 0,
  te: 0,
  total: 0,
});

// Data Logging
let dataLog: any[] = [];
let chartInstance: Chart | null = null;

// Three.js Globals
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let ballMesh: THREE.Mesh;
let surfaceMesh: THREE.Mesh;

// Cannon.js Globals
let world: CANNON.World;
let ballBody: CANNON.Body;
let surfaceBody: CANNON.Body;

// Animation Loop
let animationFrameId: number;
let lastTime = 0;
const timeStep = 1 / 60;

// --- Sidebar Resizing ---
const startResize = () => {
  isResizing.value = true;
};
const stopResize = () => {
  isResizing.value = false;
};
const handleResize = (e: MouseEvent) => {
  if (!isResizing.value) return;
  const newWidth = window.innerWidth - e.clientX;
  const minCanvasWidth = 300;
  const maxSidebar = window.innerWidth - minCanvasWidth;

  sidebarWidth.value = Math.max(
    260,
    Math.min(newWidth, Math.min(600, maxSidebar))
  );

  // Trigger resize for Three.js and Chart.js
  nextTick(() => {
    onWindowResize();
  });
};

// --- Initialization ---

onMounted(() => {
  initThree();
  initPhysics();
  initChart();
  resetSimulation();
  animate(0);

  window.addEventListener("resize", onWindowResize);
});

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId);
  window.removeEventListener("resize", onWindowResize);
  if (renderer) renderer.dispose();
  if (chartInstance) chartInstance.destroy();
});

// --- Three.js Setup ---
const initThree = () => {
  if (!canvasRef.value || !viewportRef.value) return;

  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);

  // Camera (Z-up world)
  const aspect = viewportRef.value.clientWidth / viewportRef.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 1000);
  camera.position.set(10, 10, 10);
  camera.up.set(0, 0, 1); // Z-up

  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.setSize(
    viewportRef.value.clientWidth,
    viewportRef.value.clientHeight
  );
  renderer.shadowMap.enabled = true;

  controls = new OrbitControls(camera, canvasRef.value);
  controls.enableDamping = true;

  // Lighting
  const ambientLight = new THREE.AmbientLight(0x404040);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 5, 10);
  dirLight.castShadow = true;
  scene.add(dirLight);

  // Surface Mesh
  const geometry = new THREE.PlaneGeometry(12, 12, 64, 64);
  const material = new THREE.MeshStandardMaterial({
    color: 0x3b82f6,
    side: THREE.DoubleSide,
    wireframe: true,
  });
  surfaceMesh = new THREE.Mesh(geometry, material);
  scene.add(surfaceMesh);

  // Ball Mesh
  const ballGeo = new THREE.SphereGeometry(params.radius, 32, 32);
  const ballMat = new THREE.MeshStandardMaterial({ color: 0xff0000 });
  ballMesh = new THREE.Mesh(ballGeo, ballMat);
  ballMesh.castShadow = true;
  scene.add(ballMesh);

  // Grid Helper (XY plane)
  const gridHelper = new THREE.GridHelper(20, 20);
  gridHelper.rotation.x = Math.PI / 2;
  scene.add(gridHelper);
};

// --- Physics Setup ---
const initPhysics = () => {
  world = new CANNON.World();
  world.gravity.set(0, 0, -9.8);

  // Ball Body
  const shape = new CANNON.Sphere(params.radius);
  ballBody = new CANNON.Body({
    mass: params.mass,
    position: new CANNON.Vec3(0, 0, 5),
    shape: shape,
  });
  world.addBody(ballBody);

  // Surface Body (Heightfield)
  rebuildSurface();
};

const rebuildSurface = () => {
  if (!surfaceMesh || !world) return;

  // 1. Update Visual Mesh
  const positions = surfaceMesh.geometry.attributes.position.array;
  const count = positions.length / 3;
  // PlaneGeometry is created in XY plane by default.
  // We need to map it to our Z = ax^2 + by^2 function.
  // PlaneGeometry(12, 12, 64, 64) -> x from -6 to 6, y from -6 to 6 (if centered)

  // However, PlaneGeometry vertices are indexed row by row.
  // We'll iterate and update Z based on X and Y.
  // Note: PlaneGeometry is usually Z=0. We will modify Z.

  for (let i = 0; i < count; i++) {
    const x = positions[i * 3];
    const y = positions[i * 3 + 1];
    const z = params.a * x * x + params.b * y * y;
    positions[i * 3 + 2] = z;
  }
  surfaceMesh.geometry.attributes.position.needsUpdate = true;
  surfaceMesh.geometry.computeVertexNormals();

  // 2. Update Physics Body
  if (surfaceBody) {
    world.removeBody(surfaceBody);
  }

  // Create Heightfield
  // Cannon Heightfield expects data[x][y] = z
  // We need to match the resolution of the mesh (64 segments -> 65 vertices)
  const size = 64;
  const matrix: number[][] = [];
  const range = 12;
  const step = range / size; // 12 / 64

  // Cannon Heightfield origin is at (0,0,0) of local frame, extending +x, +y
  // We need to offset it to center at (0,0) world.

  for (let i = 0; i <= size; i++) {
    matrix.push([]);
    for (let j = 0; j <= size; j++) {
      // Calculate world x, y for this grid point
      // Note: Cannon's i corresponds to x, j to y
      // But we need to be careful with orientation.
      // Let's assume standard mapping.
      const x = -6 + i * step;
      const y = 6 - j * step; // Three.js PlaneGeometry UVs might be flipped, but let's stick to math
      // Actually, let's just match the grid.
      // x from -6 to 6
      // y from -6 to 6
      const wx = -6 + i * step;
      const wy = -6 + j * step;
      const height = params.a * wx * wx + params.b * wy * wy;
      matrix[i].push(height);
    }
  }

  const hfShape = new CANNON.Heightfield(matrix, {
    elementSize: step,
  });

  surfaceBody = new CANNON.Body({ mass: 0 }); // Static
  surfaceBody.addShape(hfShape);

  // Center the heightfield
  // It starts at 0,0. We want center at 0,0.
  // So we move it by -6, -6.
  surfaceBody.position.set(-6, -6, 0);

  world.addBody(surfaceBody);
};

// --- Simulation Logic ---

const resetSimulation = () => {
  isRunning.value = false;
  simulationTime.value = 0;
  energies.te = 0;
  dataLog = [];

  // Reset Ball
  ballBody.position.set(0, 5, params.startHeight); // Start slightly offset in Y to induce roll
  ballBody.velocity.set(0, 0, 0);
  ballBody.angularVelocity.set(0, 0, 0);

  // Clear charts
  if (chartInstance) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets.forEach((ds) => (ds.data = []));
    chartInstance.update();
  }

  updateStateDisplay();
};

const toggleSimulation = () => {
  isRunning.value = !isRunning.value;
};

const stepPhysics = () => {
  if (!isRunning.value) return;

  // Custom Friction Step
  if (params.frictionEnabled) {
    applyFriction();
  }

  world.step(timeStep);
  simulationTime.value += timeStep;

  updateStateDisplay();
  logData();
};

const applyFriction = () => {
  const v = ballBody.velocity;
  const speed = v.length();

  if (speed < 0.001) return;

  // Calculate Normal Vector at current position
  const x = ballBody.position.x;
  const y = ballBody.position.y;

  // Surface normal n = (-2ax, -2by, 1) normalized
  const nx = -2 * params.a * x;
  const ny = -2 * params.b * y;
  const nz = 1;
  const len = Math.sqrt(nx * nx + ny * ny + nz * nz);

  // cos(theta) = n . k = nz / len
  const cosTheta = nz / len;

  // Friction Force Magnitude: F = mu * m * g * cos(theta)
  const F_mag = params.mu * params.mass * 9.8 * cosTheta;

  // Direction: Opposite to velocity
  const fx = (-v.x / speed) * F_mag;
  const fy = (-v.y / speed) * F_mag;
  const fz = (-v.z / speed) * F_mag;

  // Apply Force
  ballBody.applyForce(new CANNON.Vec3(fx, fy, fz), ballBody.position);

  // Thermal Energy Accumulation: Work = Force * distance = Force * speed * dt
  energies.te += F_mag * speed * timeStep;
};

const updateStateDisplay = () => {
  // Sync Mesh
  ballMesh.position.copy(ballBody.position as any);
  ballMesh.quaternion.copy(ballBody.quaternion as any);

  // Update Reactive State
  ballState.pos = {
    x: ballBody.position.x,
    y: ballBody.position.y,
    z: ballBody.position.z,
  };
  ballState.vel = {
    x: ballBody.velocity.x,
    y: ballBody.velocity.y,
    z: ballBody.velocity.z,
  };
  // Approximation of acceleration (force / mass)
  ballState.acc = {
    x: ballBody.force.x / params.mass,
    y: ballBody.force.y / params.mass,
    z: ballBody.force.z / params.mass,
  };

  // Energies
  const z = ballBody.position.z;
  const v = ballBody.velocity.length();

  energies.pe = params.mass * 9.8 * z;
  energies.ke = 0.5 * params.mass * v * v;
  energies.total = energies.pe + energies.ke + energies.te;
};

const logData = () => {
  if (simulationTime.value % 0.1 < timeStep) {
    // Log every ~0.1s
    dataLog.push({
      t: simulationTime.value.toFixed(3),
      x: ballState.pos.x.toFixed(3),
      y: ballState.pos.y.toFixed(3),
      z: ballState.pos.z.toFixed(3),
      vx: ballState.vel.x.toFixed(3),
      vy: ballState.vel.y.toFixed(3),
      vz: ballState.vel.z.toFixed(3),
      pe: energies.pe.toFixed(3),
      ke: energies.ke.toFixed(3),
      te: energies.te.toFixed(3),
      total: energies.total.toFixed(3),
    });
  }
};

// --- Chart.js ---
const initChart = () => {
  if (!chartCanvasRef.value) return;

  chartInstance = new Chart(chartCanvasRef.value, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        {
          label: "PE",
          data: [],
          borderColor: "blue",
          borderWidth: 1,
          pointRadius: 0,
        },
        {
          label: "KE",
          data: [],
          borderColor: "green",
          borderWidth: 1,
          pointRadius: 0,
        },
        {
          label: "Total",
          data: [],
          borderColor: "purple",
          borderWidth: 1,
          pointRadius: 0,
        },
        {
          label: "TE",
          data: [],
          borderColor: "red",
          borderWidth: 1,
          pointRadius: 0,
          hidden: !params.frictionEnabled,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: false,
      interaction: { mode: "index", intersect: false },
      scales: {
        x: { title: { display: true, text: "Time (s)" } },
        y: { title: { display: true, text: "Energy (J)" } },
      },
    },
  });
};

const updateChart = () => {
  if (!chartInstance) return;

  // Resample data for chart to avoid performance issues
  const sampleRate = 5; // Take every 5th log
  const labels = dataLog.filter((_, i) => i % sampleRate === 0).map((d) => d.t);
  const peData = dataLog
    .filter((_, i) => i % sampleRate === 0)
    .map((d) => d.pe);
  const keData = dataLog
    .filter((_, i) => i % sampleRate === 0)
    .map((d) => d.ke);
  const teData = dataLog
    .filter((_, i) => i % sampleRate === 0)
    .map((d) => d.te);
  const totalData = dataLog
    .filter((_, i) => i % sampleRate === 0)
    .map((d) => d.total);

  chartInstance.data.labels = labels;
  chartInstance.data.datasets[0].data = peData;
  chartInstance.data.datasets[1].data = keData;
  chartInstance.data.datasets[2].data = totalData;
  chartInstance.data.datasets[3].data = teData;
  chartInstance.data.datasets[3].hidden = !params.frictionEnabled;

  chartInstance.update();
};

// --- Export ---
const exportCSV = () => {
  const headers = Object.keys(dataLog[0] || {}).join(",");
  const rows = dataLog.map((row) => Object.values(row).join(","));
  const csvContent =
    "data:text/csv;charset=utf-8," + [headers, ...rows].join("\n");
  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", "simulation_data.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
};

const exportChartCSV = () => {
  exportCSV(); // Same data
};

// --- Animation Loop ---
const animate = (time: number) => {
  requestAnimationFrame(animate);

  const delta = time - lastTime;
  lastTime = time;
  fps.value = Math.round(1000 / delta);

  stepPhysics();
  controls.update();
  renderer.render(scene, camera);
};

const onWindowResize = () => {
  if (!viewportRef.value || !renderer || !camera) return;

  const width = viewportRef.value.clientWidth;
  const height = viewportRef.value.clientHeight;

  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
};

// --- Helpers ---
const formatVec = (v: Vec3) =>
  `(${v.x.toFixed(2)}, ${v.y.toFixed(2)}, ${v.z.toFixed(2)})`;
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
