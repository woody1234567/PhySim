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
            Physics Concepts: Doppler Effect
          </h3>

          <div class="space-y-4 text-sm leading-relaxed overflow-y-auto max-h-[70vh] pr-2">
            <section>
              <h4 class="font-bold text-lg text-secondary">1. Wave Propagation</h4>
              <p>
                The Doppler effect is the change in frequency of a wave in relation to an observer who is moving relative to the wave source. In this simulation:
              </p>
              <ul class="list-disc ml-6 mt-2 space-y-1">
                <li><strong>Red Sphere:</strong> The sound source emitting waves at frequency <MathLatex formula="f_s" />.</li>
                <li><strong>Blue Sphere:</strong> The stationary receiver detecting the wavefronts.</li>
                <li><strong>Yellow Rings:</strong> Representing the propagation of wavefronts at the speed of sound (<MathLatex formula="c" />).</li>
              </ul>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">2. The Doppler Formula</h4>
              <p>
                The observed frequency (<MathLatex formula="f_o" />) depends on the relative velocity component of the source towards the receiver (<MathLatex formula="v_s" />):
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="f_o = f_s \left( \frac{c}{c - v_s} \right)" :inline="false" />
              </div>
              <p>
                where <MathLatex formula="c \approx 343" /> m/s is the speed of sound.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">3. Approaching vs. Receding</h4>
              <ul class="list-disc ml-6 space-y-2">
                <li><strong>Approaching:</strong> When the source moves towards the receiver, <MathLatex formula="v_s" /> is positive, wavefronts are "compressed," and <MathLatex formula="f_o > f_s" />.</li>
                <li><strong>Receding:</strong> When the source moves away, <MathLatex formula="v_s" /> is negative, wavefronts are "stretched," and <MathLatex formula="f_o < f_s" />.</li>
                <li><strong>Sonic Boom:</strong> If <MathLatex formula="v_s \ge c" />, the source catches up with its own wavefronts, forming a shock wave.</li>
              </ul>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">4. Controls Guide</h4>
              <ul class="list-disc ml-6 space-y-2">
                <li><strong>Source Velocity:</strong> Adjust how fast the source travels. Positive values move towards the receiver.</li>
                <li><strong>Source Frequency:</strong> The rate at which new wavefronts are emitted.</li>
                <li><strong>Chart:</strong> Tracks the real-time frequency detected by the blue receiver.</li>
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
      class="w-2 bg-base-300 hover:bg-primary cursor-col-resize flex items-center justify-center transition-colors z-10"
      @mousedown="handleMouseDown"
    >
      <div class="w-1 h-8 bg-base-content/20 rounded-full"></div>
    </div>

    <!-- Sidebar -->
    <div
      class="flex flex-col bg-base-200 border-l border-base-300 shadow-xl z-10"
      :style="{ width: sidebarWidth + 'px', minWidth: '260px' }"
    >
      <div
        class="p-4 border-b border-base-300 font-bold text-lg flex items-center justify-between"
      >
        <div class="flex items-center gap-2">
          <i class="fa fa-cogs"></i> <span>Controls</span>
        </div>
        <UButton
          icon="i-lucide-book-open"
          color="neutral"
          variant="ghost"
          size="xs"
          @click.stop="showHelpModal = true"
        />
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
const showHelpModal = ref(true); // Auto-show on first load

// Physics Parameters
const params = reactive({
  velocity: 200, // m/s
  frequency: 8, // Hz
  speedOfSound: 343, // m/s
});

// Simulation State
const sourcePos = reactive({ x: -500, y: 0, z: 0 });
const receiverPos = reactive({ x: 0, y: 0, z: 0 }); 
const observedFreq = ref<number | null>(null);
const logs = ref<any[]>([]);
const waves = ref<{ id: number; emitTime: number; emitPos: THREE.Vector3 }[]>(
  [],
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
    if (renderer) renderer.dispose();
    if (chartInstance) chartInstance.destroy();
  }
});

// --- Sidebar Resizing ---
const handleMouseDown = () => { isDragging.value = true; document.body.style.cursor = "col-resize"; };
const handleMouseMove = (e: MouseEvent) => {
  if (!isDragging.value) return;
  const maxWidth = window.innerWidth - 300;
  const newWidth = window.innerWidth - e.clientX;
  if (newWidth >= 260 && newWidth <= maxWidth) { sidebarWidth.value = newWidth; handleResize(); }
};
const handleMouseUp = () => { isDragging.value = false; document.body.style.cursor = "default"; };

// --- Initialization ---
function initThree() {
  if (!canvasContainer.value || !canvasRef.value) return;
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);
  const gridHelper = new THREE.GridHelper(1000, 50, 0x444444, 0x222222);
  scene.add(gridHelper);
  const axesHelper = new THREE.AxesHelper(20);
  scene.add(axesHelper);
  const aspect = canvasContainer.value.clientWidth / canvasContainer.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 2000);
  camera.position.set(0, 150, 200);
  camera.lookAt(0, 0, 0);
  renderer = new THREE.WebGLRenderer({ canvas: canvasRef.value, antialias: true, });
  renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight,);
  renderer.setPixelRatio(window.devicePixelRatio);
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  const sourceGeo = new THREE.SphereGeometry(2, 16, 16);
  const sourceMat = new THREE.MeshBasicMaterial({ color: 0xff0000 });
  sourceMesh = new THREE.Mesh(sourceGeo, sourceMat);
  scene.add(sourceMesh);
  const receiverGeo = new THREE.SphereGeometry(2, 16, 16);
  const receiverMat = new THREE.MeshBasicMaterial({ color: 0x0088ff });
  receiverMesh = new THREE.Mesh(receiverGeo, receiverMat);
  receiverMesh.position.set(receiverPos.x, receiverPos.y, receiverPos.z);
  scene.add(receiverMesh);
}

function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, 0, 0);
  const shape = new CANNON.Sphere(2);
  sourceBody = new CANNON.Body({
    mass: 1,
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
    data: { labels: [], datasets: [{ label: "Observed Frequency (Hz)", data: [], borderColor: "#0088ff", tension: 0.1, pointRadius: 2, },], },
    options: {
      responsive: true, maintainAspectRatio: false, animation: false,
      scales: { x: { title: { display: true, text: "Time (s)" } }, y: { title: { display: true, text: "Freq (Hz)" } }, },
    },
  });
}

// --- Simulation Logic ---
function toggleSimulation() {
  isPlaying.value = !isPlaying.value;
  if (isPlaying.value) clock.start(); else clock.stop();
}

function resetSimulation() {
  isPlaying.value = false; time.value = 0; lastEmitTime = -1; observedFreq.value = null; logs.value = []; waves.value = [];
  sourceBody.position.set(-500, 0, 0);
  sourceBody.velocity.set(params.velocity, 0, 0);
  sourcePos.x = -500; sourcePos.y = 0; sourcePos.z = 0;
  sourceMesh.position.set(-500, 0, 0);
  waveMeshes.forEach((mesh) => scene.remove(mesh));
  waveMeshes.clear();
  resetChart();
}

function stepSimulation(dt: number) {
  sourceBody.velocity.set(params.velocity, 0, 0);
  world.step(1 / 60, dt, 3);
  sourceMesh.position.copy(sourceBody.position as any);
  sourcePos.x = sourceBody.position.x;
  sourcePos.y = sourceBody.position.y;
  sourcePos.z = sourceBody.position.z;
  const period = 1 / params.frequency;
  if (time.value - lastEmitTime >= period) { emitWave(); lastEmitTime = time.value; }
  updateWaves(dt);
  logFrame();
}

function emitWave() {
  const id = nextWaveId++;
  waves.value.push({ id, emitTime: time.value, emitPos: new THREE.Vector3(sourcePos.x, sourcePos.y, sourcePos.z), });
  const curve = new THREE.EllipseCurve(0, 0, 1, 1, 0, 2 * Math.PI, false, 0);
  const points = curve.getPoints(64);
  const geometry = new THREE.BufferGeometry().setFromPoints(points);
  geometry.rotateX(-Math.PI / 2);
  const material = new THREE.LineBasicMaterial({ color: 0xffff00 });
  const line = new THREE.LineLoop(geometry, material);
  line.position.copy(sourceMesh.position);
  scene.add(line);
  waveMeshes.set(id, line);
}

function updateWaves(dt: number) {
  const c = params.speedOfSound;
  const receiverVec = new THREE.Vector3(receiverPos.x, receiverPos.y, receiverPos.z,);
  for (let i = waves.value.length - 1; i >= 0; i--) {
    const wave = waves.value[i];
    const age = time.value - wave.emitTime;
    const radius = c * age;
    const mesh = waveMeshes.get(wave.id);
    if (mesh) mesh.scale.set(radius, radius, radius);
    const distToReceiver = wave.emitPos.distanceTo(receiverVec);
    const prevRadius = c * (age - dt);
    if (distToReceiver >= prevRadius && distToReceiver <= radius) calculateDoppler(wave);
    if (radius > 1000) {
      const m = waveMeshes.get(wave.id);
      if (m) scene.remove(m);
      waveMeshes.delete(wave.id);
      waves.value.splice(i, 1);
    }
  }
}

function calculateDoppler(wave: any) {
  const vecToReceiver = new THREE.Vector3().subVectors(new THREE.Vector3(receiverPos.x, 0, 0), wave.emitPos).normalize();
  const velVec = new THREE.Vector3(params.velocity, 0, 0);
  const v_source_component = velVec.dot(vecToReceiver);
  const c = params.speedOfSound;
  const f_src = params.frequency;
  const f_obs = f_src * (c / (c - v_source_component));
  observedFreq.value = f_obs;
  if (chartInstance) {
    chartInstance.data.labels?.push(time.value.toFixed(2));
    chartInstance.data.datasets[0].data.push(f_obs);
    if (chartInstance.data.labels && chartInstance.data.labels.length > 50) { chartInstance.data.labels.shift(); chartInstance.data.datasets[0].data.shift(); }
    chartInstance.update("none");
  }
}

function startAnimationLoop() {
  const animate = () => {
    animationFrameId = requestAnimationFrame(animate);
    if (isPlaying.value) { const dt = clock.getDelta(); time.value += dt; stepSimulation(dt); }
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

function logFrame() {
  logs.value.push({ time: time.value.toFixed(3), x: sourcePos.x.toFixed(3), v: params.velocity.toFixed(3), f_obs: observedFreq.value ? observedFreq.value.toFixed(3) : "", });
}

function exportLogsToCSV() {
  const headers = ["Time (s)", "Source X (m)", "Velocity (m/s)", "Observed Freq (Hz)"];
  const rows = logs.value.map((l) => [l.time, l.x, l.v, l.f_obs]);
  let csvContent = "data:text/csv;charset=utf-8," + headers.join(",") + "\n" + rows.map((e) => e.join(",")).join("\n");
  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a"); link.setAttribute("href", encodedUri); link.setAttribute("download", "doppler_simulation_data.csv"); document.body.appendChild(link); link.click(); document.body.removeChild(link);
}

function resetChart() { if (chartInstance) { chartInstance.data.labels = []; chartInstance.data.datasets[0].data = []; chartInstance.update(); } }

function exportChartData() {
  if (!chartInstance) return;
  const labels = chartInstance.data.labels as string[];
  const data = chartInstance.data.datasets[0].data as number[];
  const rows = labels.map((label, i) => [label, data[i]]);
  const headers = ["Time (s)", "Observed Frequency (Hz)"];
  let csvContent = "data:text/csv;charset=utf-8," + headers.join(",") + "\n" + rows.map((e) => e.join(",")).join("\n");
  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a"); link.setAttribute("href", encodedUri); link.setAttribute("download", "doppler_chart_data.csv"); document.body.appendChild(link); link.click(); document.body.removeChild(link);
}
</script>

<style scoped>
::-webkit-scrollbar { width: 8px; }
::-webkit-scrollbar-track { background: #f1f1f1; }
::-webkit-scrollbar-thumb { background: #888; border-radius: 4px; }
::-webkit-scrollbar-thumb:hover { background: #555; }
</style>
