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
        <div class="modal-box max-w-2xl bg-base-100 border border-base-300">
          <h3 class="font-bold text-2xl mb-4 flex items-center gap-2">
            <Icon name="heroicons:academic-cap" class="text-primary text-3xl" />
            Physics Concepts: Bowl Simulation
          </h3>

          <div class="space-y-4 text-sm leading-relaxed overflow-y-auto max-h-[70vh] pr-2">
            <section>
              <h4 class="font-bold text-lg text-secondary">1. Surface Geometry</h4>
              <p>
                The simulation features a particle moving on a curved parabolic surface defined by the equation:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="z = a \cdot x^2 + b \cdot y^2" :inline="false" />
              </div>
              <p>
                where <MathLatex formula="a" /> and <MathLatex formula="b" /> determine the curvature along the X and Y axes respectively.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">2. Energy Conservation</h4>
              <p>
                In the absence of friction, the total mechanical energy is conserved:
              </p>
              <ul class="list-disc ml-6 mt-2 space-y-1">
                <li><strong>Potential Energy (PE):</strong> <MathLatex formula="U = m \cdot g \cdot z" :inline="false" /></li>
                <li><strong>Kinetic Energy (KE):</strong> <MathLatex formula="K = \frac{1}{2} m v^2" :inline="false" /></li>
                <li><strong>Total Energy:</strong> <MathLatex formula="E = K + U" :inline="false" /></li>
              </ul>
              <p class="mt-2 text-xs opacity-70 italic">
                Note: The ball starts from rest at the specified initial height, converting its initial potential energy into kinetic energy as it slides down.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">3. Lagrangian Dynamics</h4>
              <p>
                The motion is calculated using Lagrangian mechanics, specifically constrained motion on a manifold. The simulation enforces the constraint that the particle must remain on the surface <MathLatex formula="f(x,y,z) = z - ax^2 - by^2 = 0" />.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">4. Friction & Work</h4>
              <p>
                When friction is enabled, the system experiences non-conservative forces. The work done by friction is calculated as:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="W_{fric} = \int \vec{F}_{fric} \cdot d\vec{s}" :inline="false" />
              </div>
              <p>
                This results in a gradual decrease in total mechanical energy over time.
              </p>
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
        class="p-4 font-bold text-lg border-b border-base-300 bg-base-100 shadow-sm flex justify-between items-center"
      >
        <span>Physics Lab</span>
        <UButton
          icon="i-lucide-book-open"
          color="neutral"
          variant="ghost"
          @click="showHelpModal = true"
        />
      </div>

      <div class="overflow-y-auto flex-1 p-2 space-y-2">
        <!-- Panel 1: Controls -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="radio" name="accordion" checked="checked" />
          <div class="collapse-title text-md font-medium">Control Panel</div>
          <div class="collapse-content space-y-4">
            <!-- Ball Params -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Initial Height (Z)</span>
                <span class="label-text-alt">{{ params.startHeight }}m</span>
              </label>
              <input
                type="range"
                min="1"
                max="6"
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
                    <td>Work</td>
                    <td>{{ energies.workFriction.toFixed(3) }}</td>
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
import { ref, onMounted, onUnmounted, reactive, nextTick } from "vue";
import * as THREE from "three";
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
const showHelpModal = ref(true); // Auto-show on first load

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
  workFriction: 0,
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

// Physics State (Custom)
let simState = {
  x: 0,
  y: 0,
  vx: 0,
  vy: 0,
};

// Animation Loop
let animationFrameId: number;
let lastTime = 0;
const timeStep = 1 / 120; // Higher precision time step

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
    Math.min(newWidth, Math.min(600, maxSidebar)),
  );

  nextTick(() => {
    onWindowResize();
  });
};

// --- Initialization ---

onMounted(() => {
  initThree();
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

  const aspect = viewportRef.value.clientWidth / viewportRef.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 1000);
  camera.position.set(10, 10, 10);
  camera.up.set(0, 0, 1);

  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.setSize(
    viewportRef.value.clientWidth,
    viewportRef.value.clientHeight,
  );
  renderer.shadowMap.enabled = true;

  controls = new OrbitControls(camera, canvasRef.value);
  controls.enableDamping = true;

  const ambientLight = new THREE.AmbientLight(0x404040);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 5, 10);
  dirLight.castShadow = true;
  scene.add(dirLight);

  const geometry = new THREE.PlaneGeometry(24, 24, 128, 128);
  const material = new THREE.MeshStandardMaterial({
    color: 0x3b82f6,
    side: THREE.DoubleSide,
    wireframe: true,
  });
  surfaceMesh = new THREE.Mesh(geometry, material);
  rebuildSurfaceMesh(); // Apply initial curvature
  scene.add(surfaceMesh);

  const ballGeo = new THREE.SphereGeometry(params.radius, 32, 32);
  const ballMat = new THREE.MeshStandardMaterial({ color: 0xff0000 });
  ballMesh = new THREE.Mesh(ballGeo, ballMat);
  ballMesh.castShadow = true;
  scene.add(ballMesh);

  const gridHelper = new THREE.GridHelper(20, 20);
  gridHelper.rotation.x = Math.PI / 2;
  scene.add(gridHelper);
};

const rebuildSurfaceMesh = () => {
  if (!surfaceMesh) return;
  const positions = surfaceMesh.geometry.attributes.position.array;
  const count = positions.length / 3;

  for (let i = 0; i < count; i++) {
    const x = positions[i * 3];
    const y = positions[i * 3 + 1];
    const z = params.a * x * x + params.b * y * y;
    positions[i * 3 + 2] = z;
  }
  surfaceMesh.geometry.attributes.position.needsUpdate = true;
  surfaceMesh.geometry.computeVertexNormals();
};

// --- Physics Logic (Lagrangian Dynamics) ---
const getSurfaceZ = (x: number, y: number) => {
  return params.a * x * x + params.b * y * y;
};

const getSurfaceNormal = (x: number, y: number) => {
  // f(x,y,z) = z - ax^2 - by^2 = 0
  // Grad(f) = (-2ax, -2by, 1)
  const nx = -2 * params.a * x;
  const ny = -2 * params.b * y;
  const nz = 1;
  const len = Math.sqrt(nx * nx + ny * ny + nz * nz);
  return { x: nx / len, y: ny / len, z: nz / len };
};

const resetSimulation = () => {
  isRunning.value = false;
  simulationTime.value = 0;
  energies.workFriction = 0;
  dataLog = [];

  // Reset State: Place ball at pure energy height on Y-axis
  const yStart = Math.sqrt(params.startHeight / params.b);

  simState = {
    x: 0,
    y: yStart,
    vx: 0,
    vy: 0,
  };

  updateStateDisplay();

  if (chartInstance) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets.forEach((ds) => (ds.data = []));
    chartInstance.update();
  }
};

const toggleSimulation = () => {
  isRunning.value = !isRunning.value;
};

const stepPhysics = () => {
  if (!isRunning.value) return;

  // Run multiple sub-steps for stability
  const subSteps = 10;
  const h = timeStep / subSteps;
  const g = 9.8;

  for (let i = 0; i < subSteps; i++) {
    const { x, y, vx, vy } = simState;
    const m = params.mass;

    const zx = 2 * params.a * x;
    const zy = 2 * params.b * y;

    // Current Position & Velocity in 3D
    const z = getSurfaceZ(x, y);
    const vz = zx * vx + zy * vy;

    const V = new THREE.Vector3(vx, vy, vz);
    const Pos = new THREE.Vector3(x, y, z);

    // Normal Vector
    const n = getSurfaceNormal(x, y);
    const N = new THREE.Vector3(n.x, n.y, n.z);

    // Forces
    const F_g = new THREE.Vector3(0, 0, -m * g);
    const f_dot_n = F_g.dot(N);
    const F_n = N.clone().multiplyScalar(f_dot_n);

    V.addScaledVector(F_g, h / m);

    // Friction
    if (params.frictionEnabled) {
      const speed = V.length();
      if (speed > 0.0001) {
        const N_mag = Math.abs(F_n.length());
        const F_f_mag = params.mu * N_mag;

        const v_dir = V.clone().normalize();
        const F_f = v_dir.multiplyScalar(-F_f_mag);

        const dW = F_f_mag * speed * h;
        energies.workFriction -= dW;

        V.addScaledVector(F_f, h / m);
      }
    }

    Pos.addScaledVector(V, h);

    const newX = Pos.x;
    const newY = Pos.y;
    simState.x = newX;
    simState.y = newY;

    const n_new = getSurfaceNormal(newX, newY);
    const N_new = new THREE.Vector3(n_new.x, n_new.y, n_new.z);

    const v_dot_n = V.dot(N_new);
    V.sub(N_new.multiplyScalar(v_dot_n));

    simState.vx = V.x;
    simState.vy = V.y;
  }

  simulationTime.value += timeStep;
  updateStateDisplay();
  logData();
};

const updateStateDisplay = () => {
  const { x, y } = simState;
  const z_surf = getSurfaceZ(x, y);
  const n = getSurfaceNormal(x, y);

  const bx = x + params.radius * n.x;
  const by = y + params.radius * n.y;
  const bz = z_surf + params.radius * n.z;

  ballMesh.position.set(bx, by, bz);

  const zx = 2 * params.a * x;
  const zy = 2 * params.b * y;
  const vz = zx * simState.vx + zy * simState.vy;

  const v_sq = simState.vx ** 2 + simState.vy ** 2 + vz ** 2;

  ballState.pos = { x: bx, y: by, z: bz };
  ballState.vel = { x: simState.vx, y: simState.vy, z: vz };

  const g = 9.8;
  energies.pe = params.mass * g * bz;
  energies.ke = 0.5 * params.mass * v_sq;
  energies.total = energies.pe + energies.ke;
};

const logData = () => {
  if (simulationTime.value % 0.1 < timeStep) {
    dataLog.push({
      t: simulationTime.value.toFixed(3),
      pe: energies.pe.toFixed(3),
      ke: energies.ke.toFixed(3),
      work: energies.workFriction.toFixed(3),
      total: energies.total.toFixed(3),
      mech: (energies.total + energies.workFriction).toFixed(3),
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
          label: "Total E",
          data: [],
          borderColor: "purple",
          borderWidth: 2,
          pointRadius: 0,
        },
        {
          label: "Work (Frict)",
          data: [],
          borderColor: "red",
          borderWidth: 1,
          pointRadius: 0,
          hidden: true,
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

  const sampleRate = 5;
  const labels = dataLog.filter((_, i) => i % sampleRate === 0).map((d) => d.t);
  const peData = dataLog
    .filter((_, i) => i % sampleRate === 0)
    .map((d) => d.pe);
  const keData = dataLog
    .filter((_, i) => i % sampleRate === 0)
    .map((d) => d.ke);
  const totalData = dataLog
    .filter((_, i) => i % sampleRate === 0)
    .map((d) => d.total);
  const workData = dataLog
    .filter((_, i) => i % sampleRate === 0)
    .map((d) => d.work);

  chartInstance.data.labels = labels;
  chartInstance.data.datasets[0].data = peData;
  chartInstance.data.datasets[1].data = keData;
  chartInstance.data.datasets[2].data = totalData;
  chartInstance.data.datasets[3].data = workData;
  chartInstance.data.datasets[3].hidden = !params.frictionEnabled;

  chartInstance.update();
};

const exportChartCSV = () => {
  const headers = Object.keys(dataLog[0] || {}).join(",");
  const rows = dataLog.map((row) => Object.values(row).join(","));
  const csvContent =
    "data:text/csv;charset=utf-8," + [headers, ...rows].join("\n");
  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", "energy_data.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
};

// --- Export CSV ---
const exportCSV = () => {
  const headers = "t,pe,ke,work,total,mech";
  const rows = dataLog.map(d => `${d.t},${d.pe},${d.ke},${d.work},${d.total},${d.mech}`);
  const csvContent = "data:text/csv;charset=utf-8," + [headers, ...rows].join("\n");
  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", "bowl_sim_data.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
};

// --- Animation Loop ---
const animate = (time: number) => {
  animationFrameId = requestAnimationFrame(animate);

  const delta = time - lastTime;
  lastTime = time;
  fps.value = delta > 0 ? Math.round(1000 / delta) : 0;

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
