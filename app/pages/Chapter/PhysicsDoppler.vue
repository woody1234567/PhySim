<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-100"
    @mousemove="handleMouseMove"
    @mouseup="handleMouseUp"
  >
    <!-- 3D Canvas Area -->
    <div
      ref="canvasContainer"
      class="flex-grow relative bg-black overflow-hidden"
    >
      <canvas ref="canvasRef" class="block w-full h-full outline-none"></canvas>

      <!-- Overlay Info -->
      <div
        class="absolute top-4 left-4 pointer-events-none text-white/80 font-mono text-sm bg-black/50 p-2 rounded"
      >
        <div>Time: {{ time.toFixed(2) }} s</div>
        <div>Source X: {{ sourcePos.x.toFixed(2) }} m</div>
        <div>
          Observed Freq:
          {{ observedFreq ? observedFreq.toFixed(2) + " Hz" : "---" }}
        </div>
      </div>
    </div>

    <!-- Resizer Handle -->
    <div
      class="w-2 bg-base-300 hover:bg-primary cursor-col-resize flex items-center justify-center transition-colors z-50"
      @mousedown="handleMouseDown"
    >
      <div class="w-1 h-8 bg-base-content/20 rounded-full"></div>
    </div>

    <!-- Sidebar -->
    <div
      class="flex flex-col bg-base-200 border-l border-base-300 shadow-xl z-40"
      :style="{ width: sidebarWidth + 'px', minWidth: '260px' }"
    >
      <div
        class="p-4 border-b border-base-300 font-bold text-lg flex items-center gap-2"
      >
        <i class="fa fa-cogs"></i> Controls
      </div>

      <div class="overflow-y-auto flex-1 p-4 space-y-4">
        <!-- Panel 1: Control Panel -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="radio" name="my-accordion-1" checked="checked" />
          <div class="collapse-title text-md font-medium">
            Simulation Parameters
          </div>
          <div class="collapse-content space-y-4 pt-2">
            <!-- Velocity Control -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Source Velocity (v)</span>
                <span class="label-text-alt font-mono"
                  >{{ params.velocity }} m/s</span
                >
              </label>
              <input
                type="range"
                min="-300"
                max="300"
                step="10"
                class="range range-primary range-xs"
                v-model.number="params.velocity"
                :disabled="isPlaying"
              />
              <div
                class="w-full flex justify-between text-xs px-2 mt-1 text-base-content/50"
              >
                <span>-300</span>
                <span>0</span>
                <span>300</span>
              </div>
            </div>

            <!-- Frequency Control -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Source Frequency (f)</span>
                <span class="label-text-alt font-mono"
                  >{{ params.frequency }} Hz</span
                >
              </label>
              <input
                type="range"
                min="1"
                max="20"
                step="1"
                class="range range-secondary range-xs"
                v-model.number="params.frequency"
                :disabled="isPlaying"
              />
            </div>

            <!-- Buttons -->
            <div class="flex gap-2 pt-2">
              <button
                class="btn btn-primary flex-1 btn-sm"
                @click="toggleSimulation"
              >
                {{ isPlaying ? "Pause" : "Start" }}
              </button>
              <button
                class="btn btn-outline btn-error flex-1 btn-sm"
                @click="resetSimulation"
              >
                Reset
              </button>
            </div>
          </div>
        </div>

        <!-- Panel 2: Live Data Panel -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="radio" name="my-accordion-1" />
          <div class="collapse-title text-md font-medium">Live Data</div>
          <div class="collapse-content space-y-2 pt-2 font-mono text-xs">
            <div class="grid grid-cols-2 gap-x-2 gap-y-1">
              <span class="opacity-70">Time:</span>
              <span class="text-right">{{ time.toFixed(3) }} s</span>

              <span class="opacity-70">Pos (x):</span>
              <span class="text-right">{{ sourcePos.x.toFixed(2) }} m</span>

              <span class="opacity-70">Vel (vx):</span>
              <span class="text-right"
                >{{ params.velocity.toFixed(1) }} m/s</span
              >

              <span class="opacity-70">Obs. Freq:</span>
              <span class="text-right text-primary font-bold">
                {{ observedFreq ? observedFreq.toFixed(2) : "---" }} Hz
              </span>
            </div>

            <button
              class="btn btn-xs btn-outline w-full mt-2"
              @click="exportLogsToCSV"
            >
              <i class="fa fa-download"></i> Export CSV
            </button>
          </div>
        </div>

        <!-- Panel 3: Chart Panel -->
        <div class="collapse collapse-arrow bg-base-100 border border-base-300">
          <input type="radio" name="my-accordion-1" />
          <div class="collapse-title text-md font-medium">Charts</div>
          <div class="collapse-content pt-2">
            <div class="h-40 w-full relative">
              <canvas ref="chartCanvas"></canvas>
            </div>
            <div class="flex gap-2 mt-2">
              <button class="btn btn-xs btn-ghost flex-1" @click="resetChart">
                Clear
              </button>
              <button
                class="btn btn-xs btn-outline flex-1"
                @click="exportChartData"
              >
                Export
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
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";

// --- State ---
const isPlaying = ref(false);
const time = ref(0);
const sidebarWidth = ref(320);
const isDragging = ref(false);
const canvasContainer = ref<HTMLElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const chartCanvas = ref<HTMLCanvasElement | null>(null);

// Physics Parameters
const params = reactive({
  velocity: 200, // m/s
  frequency: 8, // Hz
  speedOfSound: 343, // m/s
});

// Simulation State
const sourcePos = reactive({ x: -500, y: 0, z: 0 });
const receiverPos = reactive({ x: 0, y: 0, z: 0 }); // Fixed receiver at x=100
const observedFreq = ref<number | null>(null);
const logs = ref<any[]>([]);
const waves = ref<{ id: number; emitTime: number; emitPos: THREE.Vector3 }[]>(
  []
);
let nextWaveId = 0;
let lastEmitTime = -1;

// Three.js Globals
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let sourceMesh: THREE.Mesh;
let receiverMesh: THREE.Mesh;
let waveMeshes: Map<number, THREE.Mesh> = new Map();

// Cannon.js Globals
let world: CANNON.World;
let sourceBody: CANNON.Body;

// Chart.js Globals
let chartInstance: Chart | null = null;

// Animation Loop
let animationFrameId: number;
const clock = new THREE.Clock();

// --- Lifecycle ---

onMounted(() => {
  if (process.client) {
    initThree();
    initPhysics();
    initChart();
    startAnimationLoop();
    window.addEventListener("resize", handleResize);
  }
});

onUnmounted(() => {
  if (process.client) {
    window.removeEventListener("resize", handleResize);
    cancelAnimationFrame(animationFrameId);
    renderer?.dispose();
    chartInstance?.destroy();
  }
});

// --- Sidebar Resizing ---

const handleMouseDown = () => {
  isDragging.value = true;
  document.body.style.cursor = "col-resize";
};

const handleMouseMove = (e: MouseEvent) => {
  if (!isDragging.value) return;

  const maxWidth = window.innerWidth - 300; // Keep at least 300px for canvas
  const newWidth = window.innerWidth - e.clientX;

  if (newWidth >= 260 && newWidth <= maxWidth) {
    sidebarWidth.value = newWidth;
    handleResize(); // Trigger resize for Three.js
  }
};

const handleMouseUp = () => {
  isDragging.value = false;
  document.body.style.cursor = "default";
};

// --- Initialization ---

function initThree() {
  if (!canvasContainer.value || !canvasRef.value) return;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);

  // Grid & Axes
  const gridHelper = new THREE.GridHelper(1000, 50, 0x444444, 0x222222);
  scene.add(gridHelper);
  const axesHelper = new THREE.AxesHelper(20);
  scene.add(axesHelper);

  // Camera
  const aspect =
    canvasContainer.value.clientWidth / canvasContainer.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 2000);
  camera.position.set(0, 150, 200);
  camera.lookAt(0, 0, 0);

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.setSize(
    canvasContainer.value.clientWidth,
    canvasContainer.value.clientHeight
  );
  renderer.setPixelRatio(window.devicePixelRatio);

  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  // Objects
  // Source
  const sourceGeo = new THREE.SphereGeometry(2, 16, 16);
  const sourceMat = new THREE.MeshBasicMaterial({ color: 0xff0000 });
  sourceMesh = new THREE.Mesh(sourceGeo, sourceMat);
  scene.add(sourceMesh);

  // Receiver
  const receiverGeo = new THREE.SphereGeometry(2, 16, 16);
  const receiverMat = new THREE.MeshBasicMaterial({ color: 0x0088ff });
  receiverMesh = new THREE.Mesh(receiverGeo, receiverMat);
  receiverMesh.position.set(receiverPos.x, receiverPos.y, receiverPos.z);
  scene.add(receiverMesh);
}

function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, 0, 0); // No gravity needed for 1D Doppler

  // Source Body (Kinematic)
  const shape = new CANNON.Sphere(2);
  sourceBody = new CANNON.Body({
    mass: 1, // Dynamic but we control velocity
    position: new CANNON.Vec3(-500, 0, 0),
    shape: shape,
    type: CANNON.Body.KINEMATIC,
  });
  sourceBody.velocity.set(params.velocity, 0, 0);
  world.addBody(sourceBody);
}

function initChart() {
  if (!chartCanvas.value) return;

  chartInstance = new Chart(chartCanvas.value, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        {
          label: "Observed Frequency (Hz)",
          data: [],
          borderColor: "#0088ff",
          tension: 0.1,
          pointRadius: 2,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: false,
      scales: {
        x: { title: { display: true, text: "Time (s)" } },
        y: { title: { display: true, text: "Freq (Hz)" } },
      },
    },
  });
}

// --- Simulation Logic ---

function toggleSimulation() {
  isPlaying.value = !isPlaying.value;
  if (isPlaying.value) {
    clock.start();
  } else {
    clock.stop();
  }
}

function resetSimulation() {
  isPlaying.value = false;
  time.value = 0;
  lastEmitTime = -1;
  observedFreq.value = null;
  logs.value = [];
  waves.value = [];

  // Reset Physics
  sourceBody.position.set(0, 0, 0);
  sourceBody.velocity.set(params.velocity, 0, 0);
  sourcePos.x = 0;
  sourcePos.y = 0;
  sourcePos.z = 0;

  // Reset Visuals
  sourceMesh.position.set(0, 0, 0);
  waveMeshes.forEach((mesh) => scene.remove(mesh));
  waveMeshes.clear();

  resetChart();
}

function stepSimulation(dt: number) {
  // Update Physics
  sourceBody.velocity.set(params.velocity, 0, 0);
  world.step(1 / 60, dt, 3);

  // Sync Visuals
  sourceMesh.position.copy(sourceBody.position as any);
  sourcePos.x = sourceBody.position.x;
  sourcePos.y = sourceBody.position.y;
  sourcePos.z = sourceBody.position.z;

  // Wave Emission
  const period = 1 / params.frequency;
  if (time.value - lastEmitTime >= period) {
    emitWave();
    lastEmitTime = time.value;
  }

  // Wave Propagation & Detection
  updateWaves(dt);

  // Logging
  logFrame();
}

function emitWave() {
  const id = nextWaveId++;
  waves.value.push({
    id,
    emitTime: time.value,
    emitPos: new THREE.Vector3(sourcePos.x, sourcePos.y, sourcePos.z),
  });

  // Create Visual Mesh for Wave
  const geometry = new THREE.RingGeometry(0.1, 0.2, 64); // Start small
  // Rotate to lie flat on XZ plane
  geometry.rotateX(-Math.PI / 2);
  const material = new THREE.MeshBasicMaterial({
    color: 0xffff00,
    side: THREE.DoubleSide,
    transparent: true,
    opacity: 0.8,
  });
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.copy(sourceMesh.position);
  scene.add(mesh);
  waveMeshes.set(id, mesh);
}

function updateWaves(dt: number) {
  const c = params.speedOfSound;
  const receiverVec = new THREE.Vector3(
    receiverPos.x,
    receiverPos.y,
    receiverPos.z
  );

  for (let i = waves.value.length - 1; i >= 0; i--) {
    const wave = waves.value[i];
    const age = time.value - wave.emitTime;
    const radius = c * age;

    // Update Visuals
    const mesh = waveMeshes.get(wave.id);
    if (mesh) {
      const scale = radius;
      // RingGeometry innerRadius=0.1, outer=0.2. We want radius R.
      // Scale logic: if base is ~0, we need to scale geometry or recreate.
      // Better: Use RingGeometry(radius - width, radius, ...)
      // For performance, let's just scale a unit ring? No, thickness changes.
      // Re-generating geometry is slow.
      // Let's use a simple LineLoop for the wavefront circle.

      // Replacing the mesh logic with LineLoop for better performance/look
      if (mesh.geometry.type === "RingGeometry") {
        scene.remove(mesh);
        const curve = new THREE.EllipseCurve(
          0,
          0, // ax, aY
          1,
          1, // xRadius, yRadius
          0,
          2 * Math.PI, // aStartAngle, aEndAngle
          false, // aClockwise
          0 // aRotation
        );
        const points = curve.getPoints(64);
        const geometry = new THREE.BufferGeometry().setFromPoints(points);
        geometry.rotateX(-Math.PI / 2);
        const material = new THREE.LineBasicMaterial({ color: 0xffff00 });
        const line = new THREE.LineLoop(geometry, material);
        line.position.copy(wave.emitPos);
        scene.add(line);
        waveMeshes.set(wave.id, line); // Replace in map
      } else {
        // It's already a line, just scale it
        mesh.scale.set(radius, radius, radius);
      }
    }

    // Detection Logic
    // Check if wave radius ~= distance to receiver
    const distToReceiver = wave.emitPos.distanceTo(receiverVec);

    // If the wavefront crosses the receiver in this frame
    // Previous radius = c * (age - dt)
    // Current radius = c * age
    // We check if distToReceiver is between prevRadius and currRadius
    const prevRadius = c * (age - dt);

    if (distToReceiver >= prevRadius && distToReceiver <= radius) {
      // Hit detected!
      calculateDoppler(wave);
    }

    // Cleanup old waves (e.g., > 500m)
    if (radius > 500) {
      const m = waveMeshes.get(wave.id);
      if (m) scene.remove(m);
      waveMeshes.delete(wave.id);
      waves.value.splice(i, 1);
    }
  }
}

function calculateDoppler(wave: any) {
  // f_obs = f_src * (c / (c - v_src_towards_receiver))
  // For 1D: v_src is params.velocity.
  // If moving towards receiver (x < 100, v > 0), v is positive in formula?
  // Formula: f' = f * (c / (c + vr)) / (c + vs) ?
  // Standard: f' = f * (c + vo) / (c - vs)
  // vo = 0.
  // vs is component of velocity towards receiver.

  const vecToReceiver = new THREE.Vector3()
    .subVectors(new THREE.Vector3(receiverPos.x, 0, 0), wave.emitPos)
    .normalize();
  const velVec = new THREE.Vector3(params.velocity, 0, 0);
  const v_source_component = velVec.dot(vecToReceiver);

  const c = params.speedOfSound;
  const f_src = params.frequency;

  // Doppler Formula
  const f_obs = f_src * (c / (c - v_source_component));

  observedFreq.value = f_obs;

  // Update Chart
  if (chartInstance) {
    chartInstance.data.labels?.push(time.value.toFixed(2));
    chartInstance.data.datasets[0].data.push(f_obs);
    if (chartInstance.data.labels && chartInstance.data.labels.length > 50) {
      chartInstance.data.labels.shift();
      chartInstance.data.datasets[0].data.shift();
    }
    chartInstance.update("none"); // efficient update
  }
}

// --- Render Loop ---

function startAnimationLoop() {
  const animate = () => {
    animationFrameId = requestAnimationFrame(animate);

    if (isPlaying.value) {
      const dt = clock.getDelta();
      time.value += dt;
      stepSimulation(dt);
    }

    controls.update();
    renderer.render(scene, camera);
  };
  animate();
}

function handleResize() {
  if (!canvasContainer.value || !camera || !renderer) return;

  const width = canvasContainer.value.clientWidth;
  const height = canvasContainer.value.clientHeight;

  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
}

// --- Data Logging ---

function logFrame() {
  logs.value.push({
    time: time.value.toFixed(3),
    x: sourcePos.x.toFixed(3),
    v: params.velocity.toFixed(3),
    f_obs: observedFreq.value ? observedFreq.value.toFixed(3) : "",
  });
}

function exportLogsToCSV() {
  const headers = [
    "Time (s)",
    "Source X (m)",
    "Velocity (m/s)",
    "Observed Freq (Hz)",
  ];
  const rows = logs.value.map((l) => [l.time, l.x, l.v, l.f_obs]);

  let csvContent =
    "data:text/csv;charset=utf-8," +
    headers.join(",") +
    "\n" +
    rows.map((e) => e.join(",")).join("\n");

  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", "doppler_simulation_data.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function resetChart() {
  if (chartInstance) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets[0].data = [];
    chartInstance.update();
  }
}

function exportChartData() {
  if (!chartInstance) return;

  const labels = chartInstance.data.labels as string[];
  const data = chartInstance.data.datasets[0].data as number[];

  const rows = labels.map((label, i) => [label, data[i]]);
  const headers = ["Time (s)", "Observed Frequency (Hz)"];

  let csvContent =
    "data:text/csv;charset=utf-8," +
    headers.join(",") +
    "\n" +
    rows.map((e) => e.join(",")).join("\n");

  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", "doppler_chart_data.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
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
