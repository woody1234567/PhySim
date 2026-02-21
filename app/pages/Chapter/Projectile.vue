<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-100 font-sans text-base-content"
  >
    <!-- LEFT: 3D Simulation Canvas -->
    <div
      ref="canvasContainer"
      class="relative flex-grow h-full overflow-hidden bg-black"
    >
      <div
        v-if="!isClient"
        class="absolute inset-0 flex items-center justify-center text-white"
      >
        Loading 3D Environment...
      </div>

      <!-- Overlay: Legend -->
      <div
        class="absolute top-4 right-4 pointer-events-none bg-base-100/80 p-2 rounded border border-base-300 text-xs shadow-lg backdrop-blur"
      >
        <div class="font-bold mb-1">Vectors</div>
        <div class="flex items-center gap-2">
          <span class="w-3 h-3 rounded-full bg-green-500"></span> Velocity
        </div>
        <div class="flex items-center gap-2">
          <span class="w-3 h-3 rounded-full bg-red-500"></span> Acceleration
        </div>
      </div>

      <!-- Help Button (Top Left Overlay) -->
      <button
        @click="showHelpModal = true"
        class="absolute top-4 left-4 btn btn-circle btn-ghost btn-md z-10 text-primary-focus bg-base-100/50 backdrop-blur hover:bg-base-200 transition-all pointer-events-auto"
        title="Physics Explanation"
      >
        <Icon name="heroicons:question-mark-circle" class="text-3xl" />
      </button>

      <!-- Physics Help Modal -->
      <dialog class="modal" :class="{ 'modal-open': showHelpModal }">
        <div class="modal-box max-w-2xl bg-base-100 border border-base-300 text-base-content">
          <h3 class="font-bold text-2xl mb-4 flex items-center gap-2">
            <Icon name="heroicons:academic-cap" class="text-primary text-3xl" />
            Physics Concepts: Projectile Motion
          </h3>

          <div class="space-y-4 text-sm leading-relaxed overflow-y-auto max-h-[70vh] pr-2">
            <section>
              <h4 class="font-bold text-lg text-secondary">1. Independence of Motion</h4>
              <p>
                Projectile motion is a form of motion experienced by an object that is projected near the Earth's surface and moves along a curved path under the action of gravity only. The horizontal and vertical motions are independent:
              </p>
              <ul class="list-disc ml-6 mt-2 space-y-1">
                <li><strong>Horizontal:</strong> Uniform motion (constant velocity) since there is no horizontal acceleration (ignoring air resistance).</li>
                <li><strong>Vertical:</strong> Uniformly accelerated motion due to constant gravity (<MathLatex formula="g" />).</li>
              </ul>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">2. Mathematical Equations</h4>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="x = v_0 \cos(\theta) \cdot t" :inline="false" />
                <MathLatex formula="y = v_0 \sin(\theta) \cdot t - \frac{1}{2} g t^2" :inline="false" />
              </div>
              <p>
                The resulting path is a <strong>parabola</strong>. The time of flight and maximum height depend solely on the vertical component of the initial velocity.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">3. Velocity Vector</h4>
              <p>
                The instantaneous velocity (<MathLatex formula="\vec{v}" />) is always tangential to the trajectory. Its magnitude (speed) changes as the vertical component is affected by gravity, while the horizontal component remains constant.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">4. Controls Guide</h4>
              <ul class="list-disc ml-6 space-y-2">
                <li><strong>Initial Speed:</strong> Determines the magnitude of the launch velocity.</li>
                <li><strong>Launch Angle:</strong> Controls the ratio between horizontal and vertical velocity. 45° maximizes range on level ground.</li>
                <li><strong>Gravity:</strong> Adjust the local gravitational pull. Negative values pull the object downward.</li>
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
    <!-- RESIZER BAR -->
    <div
      class="w-2 h-full cursor-col-resize bg-base-300 hover:bg-primary transition-colors flex items-center justify-center z-50 select-none"
      @mousedown="startResize"
    >
      <div class="w-0.5 h-8 bg-base-content/20 rounded"></div>
    </div>

    <!-- RIGHT: Sidebar -->
    <div
      class="flex flex-col h-full bg-base-200 shadow-xl border-l border-base-300 overflow-y-auto"
      :style="{ width: sidebarWidth + 'px', minWidth: '260px' }"
    >
      <!-- Header -->
      <div class="p-4 bg-primary text-primary-content shadow-md z-10 flex justify-between items-center">
        <h1 class="text-lg font-bold flex items-center gap-2">
          <span>🚀</span> Projectile Sim
        </h1>
        <UButton
          icon="i-lucide-book-open"
          color="white"
          variant="ghost"
          size="xs"
          @click.stop="showHelpModal = true"
        />
      </div>

      <div class="flex-grow p-2 space-y-2">
        <!-- PANEL 1: CONTROLS -->
        <div
          class="collapse collapse-arrow bg-base-100 shadow-sm rounded-box border border-base-300"
        >
          <input type="radio" name="sidebar-accordion" checked="checked" />
          <div class="collapse-title text-md font-medium font-bold">
            🎛️ Control Panel
          </div>
          <div class="collapse-content space-y-4 pt-2">
            <!-- Input: Launch Speed -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Initial Speed (m/s)</span>
                <span class="label-text-alt font-mono">{{
                  params.speed.toFixed(1)
                }}</span>
              </label>
              <input
                type="range"
                min="1"
                max="100"
                step="0.5"
                v-model.number="params.speed"
                class="range range-primary range-sm"
                :disabled="isSimulating"
              />
            </div>

            <!-- Input: Launch Angle -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Launch Angle (deg)</span>
                <span class="label-text-alt font-mono"
                  >{{ params.angle.toFixed(0) }}°</span
                >
              </label>
              <input
                type="range"
                min="0"
                max="90"
                step="1"
                v-model.number="params.angle"
                class="range range-secondary range-sm"
                :disabled="isSimulating"
              />
            </div>

            <!-- Input: Gravity -->
            <div class="form-control w-full">
              <label class="label">
                <span class="label-text">Gravity (m/s²)</span>
                <span class="label-text-alt font-mono">{{
                  params.gravity.toFixed(1)
                }}</span>
              </label>
              <input
                type="range"
                min="-20"
                max="-1"
                step="0.5"
                v-model.number="params.gravity"
                class="range range-accent range-sm"
                :disabled="isSimulating"
              />
            </div>

            <!-- Control Buttons -->
            <div class="grid grid-cols-2 gap-2 pt-2">
              <button
                class="btn btn-success btn-sm"
                @click="startRestartSimulation"
              >
                {{ isSimulating || isFinished ? "Restart" : "Start" }}
              </button>

              <button
                class="btn btn-warning btn-sm"
                @click="togglePause"
                :disabled="!isSimulating || isFinished"
              >
                {{ isPaused ? "Resume" : "Pause" }}
              </button>
            </div>
          </div>
        </div>

        <!-- PANEL 2: LIVE DATA -->
        <div
          class="collapse collapse-arrow bg-base-100 shadow-sm rounded-box border border-base-300"
        >
          <input type="radio" name="sidebar-accordion" />
          <div class="collapse-title text-md font-medium font-bold">
            📊 Live Data
          </div>
          <div class="collapse-content pt-2">
            <div class="overflow-x-auto">
              <table class="table table-xs table-zebra w-full font-mono">
                <tbody>
                  <tr>
                    <th>Time</th>
                    <td>{{ liveData.t }} s</td>
                  </tr>
                  <tr>
                    <th>Pos (X,Y,Z)</th>
                    <td>
                      {{ liveData.x }}, {{ liveData.y }}, {{ liveData.z }}
                    </td>
                  </tr>
                  <tr>
                    <th>Vel (X,Y,Z)</th>
                    <td>
                      {{ liveData.vx }}, {{ liveData.vy }}, {{ liveData.vz }}
                    </td>
                  </tr>
                  <tr>
                    <th>Acc (X,Y,Z)</th>
                    <td>
                      {{ liveData.ax }}, {{ liveData.ay }}, {{ liveData.az }}
                    </td>
                  </tr>
                  <tr>
                    <th>Speed</th>
                    <td>{{ liveData.speed }} m/s</td>
                  </tr>
                </tbody>
              </table>
            </div>
            <button
              class="btn btn-outline btn-info btn-xs w-full mt-3"
              @click="exportLogsToCSV"
            >
              📥 Export Data (CSV)
            </button>
          </div>
        </div>

        <!-- PANEL 3: CHART -->
        <div
          class="collapse collapse-arrow bg-base-100 shadow-sm rounded-box border border-base-300"
        >
          <input type="radio" name="sidebar-accordion" />
          <div class="collapse-title text-md font-medium font-bold">
            📈 Analysis Chart
          </div>
          <div class="collapse-content pt-2 flex flex-col gap-2">
            <div class="h-48 w-full bg-white rounded p-1">
              <canvas ref="chartCanvas"></canvas>
            </div>
            <div class="grid grid-cols-2 gap-2">
              <button class="btn btn-primary btn-xs" @click="plotChart">
                Update Plot
              </button>
              <button class="btn btn-outline btn-xs" @click="exportChartCSV">
                Export Plot
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- Footer Info -->
      <div class="p-2 text-center text-xs text-base-content/50 bg-base-300">
        Nuxt 4 • Three.js • Cannon-es
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, reactive, watch } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";

// ==========================================
// 🛠️ STATE & CONFIGURATION
// ==========================================

const isClient = ref(false);
const canvasContainer = ref<HTMLElement | null>(null);
const chartCanvas = ref<HTMLCanvasElement | null>(null);

// Layout State
const sidebarWidth = ref(320);
const minCanvasWidth = 300;

// Simulation State
const isSimulating = ref(false);
const isPaused = ref(false);
const isFinished = ref(false);
const showHelpModal = ref(true); // Auto-show on first load
let animationFrameId: number;

// Physics Parameters
const params = reactive({
  speed: 20,
  angle: 45,
  gravity: -9.81,
  radius: 0.5,
  mass: 1,
});

// Live Data Display
const liveData = reactive({
  t: "0.00",
  x: "0.00",
  y: "0.00",
  z: "0.00",
  vx: "0.00",
  vy: "0.00",
  vz: "0.00",
  ax: "0.00",
  ay: "0.00",
  az: "0.00",
  speed: "0.00",
});

// Data Logging
interface DataPoint {
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
}
let dataLogs: DataPoint[] = [];

// Three.js Globals
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let ballMesh: THREE.Mesh;
let groundMesh: THREE.Mesh;
let trajectoryLine: THREE.Line;
let velocityArrow: THREE.ArrowHelper;
let accelArrow: THREE.ArrowHelper;

// Cannon.js Globals
let world: CANNON.World;
let ballBody: CANNON.Body;
let groundBody: CANNON.Body;

// Chart.js Global
let chartInstance: Chart | null = null;

// ==========================================
// 📐 RESIZABLE SIDEBAR LOGIC
// ==========================================

const startResize = () => {
  window.addEventListener("mousemove", onResize);
  window.addEventListener("mouseup", stopResize);
};

const onResize = (e: MouseEvent) => {
  const maxWidth = window.innerWidth - minCanvasWidth;
  const newWidth = window.innerWidth - e.clientX;
  if (newWidth >= 260 && newWidth <= maxWidth) { sidebarWidth.value = newWidth; } else if (newWidth > maxWidth) { sidebarWidth.value = maxWidth; }
  handleWindowResize();
};

const stopResize = () => {
  window.removeEventListener("mousemove", onResize);
  window.removeEventListener("mouseup", stopResize);
};

// ==========================================
// 🎥 THREE.JS SETUP
// ==========================================

const initThree = () => {
  if (!canvasContainer.value) return;
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);
  scene.fog = new THREE.Fog(0x111111, 20, 100);
  const aspect = canvasContainer.value.clientWidth / canvasContainer.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 1000);
  camera.position.set(10, 10, 20);
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight);
  renderer.shadowMap.enabled = true;
  canvasContainer.value.appendChild(renderer.domElement);
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  const ambientLight = new THREE.AmbientLight(0x404040);
  scene.add(ambientLight);
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(10, 20, 10);
  dirLight.castShadow = true;
  dirLight.shadow.mapSize.width = 2048; dirLight.shadow.mapSize.height = 2048;
  scene.add(dirLight);
  const gridHelper = new THREE.GridHelper(50, 50, 0x444444, 0x222222);
  scene.add(gridHelper);
  const axesHelper = new THREE.AxesHelper(2);
  scene.add(axesHelper);
  const groundGeo = new THREE.PlaneGeometry(200, 200);
  const groundMat = new THREE.MeshStandardMaterial({ color: 0x333333, roughness: 0.8, metalness: 0.2, });
  groundMesh = new THREE.Mesh(groundGeo, groundMat);
  groundMesh.rotation.x = -Math.PI / 2;
  groundMesh.receiveShadow = true;
  scene.add(groundMesh);
  const ballGeo = new THREE.SphereGeometry(params.radius, 32, 32);
  const ballMat = new THREE.MeshStandardMaterial({ color: 0xff0055, roughness: 0.4, });
  ballMesh = new THREE.Mesh(ballGeo, ballMat);
  ballMesh.castShadow = true;
  scene.add(ballMesh);
  const lineMat = new THREE.LineBasicMaterial({ color: 0xffff00 });
  const lineGeo = new THREE.BufferGeometry();
  trajectoryLine = new THREE.Line(lineGeo, lineMat);
  scene.add(trajectoryLine);
  velocityArrow = new THREE.ArrowHelper(new THREE.Vector3(1, 0, 0), new THREE.Vector3(0, 0, 0), 1, 0x22c55e);
  scene.add(velocityArrow);
  accelArrow = new THREE.ArrowHelper(new THREE.Vector3(1, 0, 0), new THREE.Vector3(0, 0, 0), 1, 0xef4444);
  scene.add(accelArrow);
  renderer.render(scene, camera);
};

const initPhysics = () => {
  world = new CANNON.World();
  world.gravity.set(0, params.gravity, 0);
  const concreteMaterial = new CANNON.Material("concrete");
  const plasticMaterial = new CANNON.Material("plastic");
  const contactMaterial = new CANNON.ContactMaterial(concreteMaterial, plasticMaterial, { friction: 0.1, restitution: 0.6, });
  world.addContactMaterial(contactMaterial);
  const groundShape = new CANNON.Plane();
  groundBody = new CANNON.Body({ mass: 0, material: concreteMaterial });
  groundBody.addShape(groundShape);
  groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
  world.addBody(groundBody);
  const ballShape = new CANNON.Sphere(params.radius);
  ballBody = new CANNON.Body({ mass: params.mass, material: plasticMaterial, linearDamping: 0.01, angularDamping: 0.01, });
  ballBody.addShape(ballShape);
  world.addBody(ballBody);
  resetBallPosition();
};

const resetBallPosition = () => {
  ballBody.position.set(0, params.radius, 0);
  ballBody.velocity.set(0, 0, 0);
  ballBody.angularVelocity.set(0, 0, 0);
  ballMesh.position.copy(ballBody.position as any);
  ballMesh.quaternion.copy(ballBody.quaternion as any);
};

const startRestartSimulation = () => {
  world.gravity.set(0, params.gravity, 0);
  resetBallPosition();
  const rad = params.angle * (Math.PI / 180);
  const vx = params.speed * Math.cos(rad);
  const vy = params.speed * Math.sin(rad);
  ballBody.velocity.set(vx, vy, 0);
  dataLogs = []; isFinished.value = false; isPaused.value = false; isSimulating.value = true;
  trajectoryLine.geometry.setFromPoints([]);
  if (chartInstance) { chartInstance.destroy(); chartInstance = null; }
};

const togglePause = () => { isPaused.value = !isPaused.value; };

const stepSimulation = () => {
  if (isPaused.value || isFinished.value) return;
  const timeStep = 1 / 60;
  const vPrev = ballBody.velocity.clone();
  world.step(timeStep);
  const vCurr = ballBody.velocity;
  const accel = vCurr.vsub(vPrev).scale(1 / timeStep);
  ballMesh.position.copy(ballBody.position as any);
  ballMesh.quaternion.copy(ballBody.quaternion as any);
  const t = dataLogs.length * timeStep;
  const pos = ballBody.position;
  const vel = ballBody.velocity;
  const frameData = { t: parseFloat(t.toFixed(3)), x: parseFloat(pos.x.toFixed(3)), y: parseFloat(pos.y.toFixed(3)), z: parseFloat(pos.z.toFixed(3)), vx: parseFloat(vel.x.toFixed(3)), vy: parseFloat(vel.y.toFixed(3)), vz: parseFloat(vel.z.toFixed(3)), ax: parseFloat(accel.x.toFixed(3)), ay: parseFloat(accel.y.toFixed(3)), az: parseFloat(accel.z.toFixed(3)), };
  dataLogs.push(frameData);
  updateLiveData(frameData);
  updateTrajectory();
  updateVectors(vel, accel);
  if (pos.y <= params.radius + 0.01 && vel.y <= 0) { isFinished.value = true; isSimulating.value = false; ballBody.position.y = params.radius; ballBody.velocity.set(0, 0, 0); ballMesh.position.y = params.radius; }
};

const updateLiveData = (d: DataPoint) => {
  liveData.t = d.t.toFixed(2); liveData.x = d.x.toFixed(2); liveData.y = d.y.toFixed(2); liveData.z = d.z.toFixed(2); liveData.vx = d.vx.toFixed(2); liveData.vy = d.vy.toFixed(2); liveData.vz = d.vz.toFixed(2); liveData.ax = d.ax.toFixed(2); liveData.ay = d.ay.toFixed(2); liveData.az = d.az.toFixed(2); liveData.speed = Math.sqrt(d.vx ** 2 + d.vy ** 2 + d.vz ** 2).toFixed(2);
};

const updateVectors = (vel: CANNON.Vec3, accel: CANNON.Vec3) => {
  const vLen = vel.length();
  if (vLen > 0.01) { velocityArrow.visible = true; velocityArrow.setDirection(new THREE.Vector3(vel.x, vel.y, vel.z).normalize()); velocityArrow.setLength(vLen * 0.2); velocityArrow.position.copy(ballMesh.position); } else { velocityArrow.visible = false; }
  const aLen = accel.length();
  if (aLen > 0.01) { accelArrow.visible = true; accelArrow.setDirection(new THREE.Vector3(accel.x, accel.y, accel.z).normalize()); accelArrow.setLength(aLen * 0.2); accelArrow.position.copy(ballMesh.position); } else { accelArrow.visible = false; }
};

const updateTrajectory = () => {
  if (dataLogs.length % 5 !== 0) return;
  const points = dataLogs.map((d) => new THREE.Vector3(d.x, d.y, d.z));
  trajectoryLine.geometry.setFromPoints(points);
};

const renderLoop = () => {
  if (isSimulating.value) stepSimulation();
  controls.update();
  renderer.render(scene, camera);
  animationFrameId = requestAnimationFrame(renderLoop);
};

const plotChart = () => {
  if (!chartCanvas.value || dataLogs.length === 0) return;
  if (chartInstance) chartInstance.destroy();
  const ctx = chartCanvas.value.getContext("2d");
  if (!ctx) return;
  chartInstance = new Chart(ctx, { type: "line", data: { labels: dataLogs.map((d) => d.t), datasets: [{ label: "Height (Y)", data: dataLogs.map((d) => d.y), borderColor: "rgb(255, 99, 132)", tension: 0.1, pointRadius: 0, }, { label: "Velocity Y", data: dataLogs.map((d) => d.vy), borderColor: "rgb(54, 162, 235)", tension: 0.1, pointRadius: 0, },], }, options: { responsive: true, maintainAspectRatio: false, animation: false, interaction: { mode: "index", intersect: false, }, }, });
};

const exportChartCSV = () => { if (!chartInstance) return; exportLogsToCSV(); };

const exportLogsToCSV = () => {
  if (dataLogs.length === 0) return;
  const headers = ["time", "x", "y", "z", "vx", "vy", "vz", "ax", "ay", "az"];
  const rows = dataLogs.map((d) => `${d.t},${d.x},${d.y},${d.z},${d.vx},${d.vy},${d.vz},${d.ax},${d.ay},${d.az}`);
  const csvContent = [headers.join(","), ...rows].join("\n");
  const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a"); link.setAttribute("href", url); link.setAttribute("download", `simulation_data_${new Date().toISOString()}.csv`); link.style.visibility = "hidden"; document.body.appendChild(link); link.click(); document.body.removeChild(link);
};

const handleWindowResize = () => {
  if (!canvasContainer.value || !renderer || !camera) return;
  const w = canvasContainer.value.clientWidth; const h = canvasContainer.value.clientHeight;
  camera.aspect = w / h; camera.updateProjectionMatrix(); renderer.setSize(w, h);
};

onMounted(() => {
  isClient.value = true;
  initThree(); initPhysics();
  window.addEventListener("resize", handleWindowResize);
  const resizeObserver = new ResizeObserver(() => handleWindowResize());
  if (canvasContainer.value) resizeObserver.observe(canvasContainer.value);
  renderLoop();
});

onBeforeUnmount(() => {
  window.removeEventListener("resize", handleWindowResize);
  cancelAnimationFrame(animationFrameId);
  if (chartInstance) chartInstance.destroy();
  if (renderer) renderer.dispose();
});
</script>

<style scoped>
canvas { width: 100% !important; height: 100% !important; }
</style>
