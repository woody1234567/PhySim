<template>
  <div
    class="w-full h-full relative flex flex-col overflow-hidden font-sans bg-base-200"
  >
    <!-- DAISYUI DRAWER: LAYOUT -->
    <div
      class="drawer drawer-end h-screen w-full shadow-2xl overflow-hidden"
    >
      <input
        id="sim-drawer"
        type="checkbox"
        class="drawer-toggle"
        v-model="drawerOpen"
      />

      <!-- 3D VIEWPORT (DRAWER CONTENT) -->
      <div class="drawer-content relative">
        <div
          ref="canvasContainer"
          class="w-full h-full bg-gradient-to-b from-sky-300 to-sky-100 relative"
        >
          <!-- Overlay UI -->
          <div class="absolute top-4 left-4 z-10 flex flex-col gap-2">
            <h1 class="text-2xl font-bold text-slate-800 drop-shadow-md">
              ⚡ Physics Lab
            </h1>
            <div class="badge badge-primary badge-lg shadow-sm">
              t: {{ currentTime.toFixed(2) }} s
            </div>
            <div
              v-if="isFinished"
              class="badge badge-error badge-lg animate-pulse shadow-sm"
            >
              Impact Detected
            </div>
          </div>

          <!-- Open Controls Button (Visible when closed) -->
          <label
            for="sim-drawer"
            class="btn btn-circle btn-neutral absolute top-4 right-4 z-10 shadow-lg drawer-button"
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
              stroke-width="1.5"
              stroke="currentColor"
              class="w-6 h-6"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M10.5 6h9.75M10.5 6a1.5 1.5 0 1 1-3 0m3 0a1.5 1.5 0 1 0-3 0M3.75 6H7.5m3 12h9.75m-9.75 0a1.5 1.5 0 0 1-3 0m3 0a1.5 1.5 0 0 0-3 0m-3.75 0H7.5m9-6h3.75m-3.75 0a1.5 1.5 0 0 1-3 0m3 0a1.5 1.5 0 0 0-3 0m-9.75 0h9.75"
              />
            </svg>
          </label>

          <!-- Help Button (Top Left Overlay) -->
          <button
            @click="showHelpModal = true"
            class="absolute top-20 left-4 btn btn-circle btn-ghost btn-md z-10 text-primary-focus bg-base-100/50 backdrop-blur hover:bg-base-200 transition-all"
            title="Physics Explanation"
          >
            <Icon name="heroicons:question-mark-circle" class="text-3xl" />
          </button>

          <!-- Physics Help Modal -->
          <dialog class="modal" :class="{ 'modal-open': showHelpModal }">
            <div class="modal-box max-w-2xl bg-base-100 border border-base-300 text-base-content">
              <h3 class="font-bold text-2xl mb-4 flex items-center gap-2">
                <Icon name="heroicons:academic-cap" class="text-primary text-3xl" />
                Physics Concepts: Kinetics & Trajectory
              </h3>

              <div class="space-y-4 text-sm leading-relaxed overflow-y-auto max-h-[70vh] pr-2">
                <section>
                  <h4 class="font-bold text-lg text-secondary">1. Projectile Motion</h4>
                  <p>
                    The trajectory of the ball is determined by decomposing its initial velocity (<MathLatex formula="v_0" />) into horizontal and vertical components:
                  </p>
                  <ul class="list-disc ml-6 mt-2 space-y-1">
                    <li><strong>Horizontal (<MathLatex formula="v_x" />):</strong> <MathLatex formula="v_0 \cdot \cos(\theta)" /> - Constant (ignoring air resistance).</li>
                    <li><strong>Vertical (<MathLatex formula="v_y" />):</strong> <MathLatex formula="v_0 \cdot \sin(\theta) - g \cdot t" /> - Subject to gravitational acceleration.</li>
                  </ul>
                </section>

                <section>
                  <h4 class="font-bold text-lg text-secondary">2. Position Equations</h4>
                  <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                    <MathLatex formula="x(t) = v_{0x} \cdot t" :inline="false" />
                    <MathLatex formula="y(t) = h_0 + v_{0y} \cdot t - \frac{1}{2} g t^2" :inline="false" />
                  </div>
                  <p>
                    where <MathLatex formula="h_0" /> is the initial height and <MathLatex formula="g" /> is the gravitational acceleration.
                  </p>
                </section>

                <section>
                  <h4 class="font-bold text-lg text-secondary">3. Slope Interaction</h4>
                  <p>
                    The ground can be tilted by an angle (<MathLatex formula="\alpha" />). This changes the normal vector of the impact surface, affecting how the ball bounces and rolls:
                  </p>
                  <ul class="list-disc ml-6 space-y-1">
                    <li><strong>Friction:</strong> Calculated based on the normal force component and the friction coefficient (<MathLatex formula="\mu" />).</li>
                    <li><strong>Impact:</strong> The collision handling is performed by the Cannon-es physics engine, simulating elastic restitution.</li>
                  </ul>
                </section>

                <section>
                  <h4 class="font-bold text-lg text-secondary">4. Parameters Guide</h4>
                  <ul class="list-disc ml-6 space-y-2">
                    <li><strong>Launch Angle:</strong> Controls the direction of initial velocity. 45° typically yields maximum range on flat ground.</li>
                    <li><strong>Initial Height:</strong> Increases the time of flight and potential energy.</li>
                    <li><strong>Slope Angle:</strong> Simulates "uphill" (positive) or "downhill" (negative) conditions.</li>
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

          <!-- Canvas Target -->
          <canvas
            ref="canvasRef"
            class="w-full h-full block touch-none"
          ></canvas>
        </div>
      </div>

      <!-- SIDEBAR CONTROLS (DRAWER SIDE) -->
      <div class="drawer-side z-20">
        <label for="sim-drawer" class="drawer-overlay"></label>
        <div
          class="menu p-4 w-96 min-h-full bg-base-100 text-base-content flex flex-col gap-4 shadow-xl overflow-y-auto"
        >
          <div class="flex justify-between items-center mb-2">
            <h2 class="text-xl font-bold flex items-center gap-2">
              <span>⚙️ Configuration</span>
            </h2>
            <UButton
              icon="i-lucide-book-open"
              color="neutral"
              variant="ghost"
              size="xs"
              @click.stop="showHelpModal = true"
            />
          </div>

          <!-- PANEL 1: CONTROLS -->
          <div
            class="collapse collapse-arrow bg-base-200 border border-base-300"
          >
            <input type="radio" name="accordion-1" checked />
            <div class="collapse-title text-lg font-medium">
              Parameters & Control
            </div>
            <div class="collapse-content space-y-3">
              <!-- Input Grid -->
              <div class="grid grid-cols-2 gap-2">
                <div class="form-control w-full">
                  <label class="label"
                    ><span class="label-text-alt">Mass (kg)</span></label
                  >
                  <input
                    type="number"
                    v-model.number="params.mass"
                    class="input input-bordered input-sm"
                    step="0.1"
                  />
                </div>
                <div class="form-control w-full">
                  <label class="label"
                    ><span class="label-text-alt">Gravity (m/s²)</span></label
                  >
                  <input
                    type="number"
                    v-model.number="params.gravity"
                    class="input input-bordered input-sm"
                    step="0.1"
                  />
                </div>
                <div class="form-control w-full">
                  <label class="label"
                    ><span class="label-text-alt">Init Speed (m/s)</span></label
                  >
                  <input
                    type="number"
                    v-model.number="params.initialSpeed"
                    class="input input-bordered input-sm"
                  />
                </div>
                <div class="form-control w-full">
                  <label class="label"
                    ><span class="label-text-alt"
                      >Launch Angle (deg)</span
                    ></label
                  >
                  <input
                    type="number"
                    v-model.number="params.launchAngle"
                    class="input input-bordered input-sm"
                  />
                </div>
                <div class="form-control w-full">
                  <label class="label"
                    ><span class="label-text-alt">Init Height (m)</span></label
                  >
                  <input
                    type="number"
                    v-model.number="params.initialHeight"
                    class="input input-bordered input-sm"
                  />
                </div>
                <div class="form-control w-full">
                  <label class="label"
                    ><span class="label-text-alt"
                      >Slope Angle (deg)</span
                    ></label
                  >
                  <input
                    type="number"
                    v-model.number="params.slopeAngle"
                    class="input input-bordered input-sm"
                  />
                </div>
                <div class="form-control w-full col-span-2">
                  <label class="label"
                    ><span class="label-text-alt"
                      >Friction Coeff (0-1)</span
                    ></label
                  >
                  <input
                    type="range"
                    min="0"
                    max="1"
                    step="0.05"
                    v-model.number="params.friction"
                    class="range range-xs range-primary"
                  />
                </div>
              </div>

              <!-- Action Buttons -->
              <div class="divider my-1"></div>
              <div class="flex flex-col gap-2">
                <button
                  v-if="!isRunning"
                  @click="startSimulation"
                  class="btn btn-primary w-full"
                >
                  {{ currentTime > 0 ? "Resume" : "Start Simulation" }}
                </button>
                <button
                  v-else
                  @click="pauseSimulation"
                  class="btn btn-warning w-full"
                >
                  Pause
                </button>
                <button
                  @click="resetSimulation"
                  class="btn btn-outline btn-error w-full"
                >
                  Reset / Reload
                </button>
              </div>
            </div>
          </div>

          <!-- PANEL 2: LIVE DATA -->
          <div
            class="collapse collapse-arrow bg-base-200 border border-base-300"
          >
            <input type="radio" name="accordion-1" />
            <div class="collapse-title text-lg font-medium">Live Telemetry</div>
            <div class="collapse-content">
              <div class="overflow-x-auto">
                <table class="table table-xs table-zebra w-full font-mono">
                  <tbody>
                    <tr>
                      <th>Time</th>
                      <td>{{ liveData.t }} s</td>
                    </tr>
                    <tr>
                      <th>Pos X</th>
                      <td>{{ liveData.x }} m</td>
                    </tr>
                    <tr>
                      <th>Pos Y</th>
                      <td>{{ liveData.y }} m</td>
                    </tr>
                    <tr>
                      <th>Pos Z</th>
                      <td>{{ liveData.z }} m</td>
                    </tr>
                    <tr>
                      <th>Vel |V|</th>
                      <td>{{ liveData.v }} m/s</td>
                    </tr>
                    <tr>
                      <th>Acc |A|</th>
                      <td>{{ liveData.a }} m/s²</td>
                    </tr>
                  </tbody>
                </table>
              </div>
              <button
                @click="downloadCSV"
                class="btn btn-xs btn-ghost w-full mt-2 border border-base-content/20"
              >
                📥 Export CSV Log
              </button>
            </div>
          </div>

          <!-- PANEL 3: CHARTS -->
          <div
            class="collapse collapse-arrow bg-base-200 border border-base-300"
          >
            <input type="radio" name="accordion-1" />
            <div class="collapse-title text-lg font-medium">
              Analysis & Charts
            </div>
            <div class="collapse-content">
              <div class="w-full h-48 mb-2 bg-white rounded p-2">
                <canvas ref="chartCanvas"></canvas>
              </div>
              <div class="flex gap-2">
                <button @click="updateChart" class="btn btn-sm btn-info flex-1">
                  Plot Chart
                </button>
              </div>
              <div class="form-control mt-2">
                <label class="label cursor-pointer">
                  <span class="label-text text-xs">Auto-plot on finish</span>
                  <input
                    type="checkbox"
                    v-model="autoPlot"
                    class="checkbox checkbox-xs"
                  />
                </label>
              </div>
            </div>
          </div>

          <div class="text-xs text-center text-base-content/50 mt-auto pt-4">
            Three.js • Cannon-es • Chart.js
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
/**
 * PhysicsProjectile.vue
 * A self-contained physics simulation component for Nuxt 4 / Vue 3.
 *
 * Dependencies (ensure these are installed in package.json):
 * - three
 * - cannon-es
 * - chart.js
 */

import { ref, reactive, onMounted, onUnmounted, shallowRef, watch } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import * as CANNON from "cannon-es";
import { Chart, registerables } from "chart.js";

// Register Chart.js components
Chart.register(...registerables);

// --- 1. STATE & CONFIGURATION ---

const drawerOpen = ref(true);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const canvasContainer = ref<HTMLElement | null>(null);
const chartCanvas = ref<HTMLCanvasElement | null>(null);

// Simulation Control Flags
const isRunning = ref(false);
const isFinished = ref(false);
const autoPlot = ref(true);
const currentTime = ref(0);
const showHelpModal = ref(true); // Auto-show on first load

// Physics Parameters
const params = reactive({
  mass: 1.0, // kg
  gravity: 9.81, // m/s^2
  initialSpeed: 15, // m/s
  launchAngle: 45, // degrees (trajectory)
  initialHeight: 2, // meters
  slopeAngle: 0, // degrees (ground incline)
  friction: 0.3, // 0 to 1
  timeStep: 1 / 60,
});

// Live Data for UI
const liveData = reactive({
  t: "0.00",
  x: "0.00",
  y: "0.00",
  z: "0.00",
  v: "0.00",
  a: "0.00",
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
let dataLog: DataPoint[] = [];

// --- 2. ENGINE REFERENCES (Non-reactive for performance) ---

// Three.js
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let ballMesh: THREE.Mesh;
let groundMesh: THREE.Mesh;
let slopeContainer: THREE.Group;
let arrowHelper: THREE.ArrowHelper;

// Cannon.js
let world: CANNON.World;
let ballBody: CANNON.Body;
let groundBody: CANNON.Body;
let physicsMaterial: CANNON.Material;
let physicsContactMaterial: CANNON.ContactMaterial;

// Chart.js
let chartInstance: Chart | null = null;

// Loop ID
let animationFrameId: number | null = null;
let lastCallTime = 0;

// --- 3. INITIALIZATION ---

onMounted(() => {
  if (!process.client) return; // Nuxt safety

  initThree();
  initPhysics();
  initChart();

  // Initial render without running physics
  renderer.render(scene, camera);
  updateVisuals();
});

onUnmounted(() => {
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  if (renderer) renderer.dispose();
  if (chartInstance) chartInstance.destroy();
});

// --- 4. THREE.JS SETUP ---

function initThree() {
  if (!canvasRef.value || !canvasContainer.value) return;

  // Scene
  scene = new THREE.Scene();
  scene.background = null; // Let CSS gradient show

  // Camera
  const width = canvasContainer.value.clientWidth;
  const height = canvasContainer.value.clientHeight;
  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000);
  camera.position.set(20, 20, 30);
  camera.lookAt(0, 2, 0);

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
    alpha: true,
  });
  renderer.setSize(width, height);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;

  // Lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(10, 20, 10);
  dirLight.castShadow = true;
  dirLight.shadow.mapSize.width = 2048;
  dirLight.shadow.mapSize.height = 2048;
  dirLight.shadow.camera.near = 0.5;
  dirLight.shadow.camera.far = 50;
  dirLight.shadow.camera.left = -20;
  dirLight.shadow.camera.right = 20;
  dirLight.shadow.camera.top = 20;
  dirLight.shadow.camera.bottom = -20;
  scene.add(dirLight);

  // Container for Rotating Slope
  slopeContainer = new THREE.Group();
  scene.add(slopeContainer);

  // Ground (Visual) - Grid + Plane
  const planeGeo = new THREE.PlaneGeometry(50, 50);
  const planeMat = new THREE.MeshStandardMaterial({
    color: 0xf0f0f0,
    roughness: 0.8,
    side: THREE.DoubleSide,
  });
  groundMesh = new THREE.Mesh(planeGeo, planeMat);
  groundMesh.rotation.x = -Math.PI / 2;
  groundMesh.receiveShadow = true;
  slopeContainer.add(groundMesh);

  const gridHelper = new THREE.GridHelper(50, 50, 0x888888, 0xdddddd);
  gridHelper.position.y = 0.01; // slight offset to prevent z-fighting
  slopeContainer.add(gridHelper);

  // Coordinate Axes
  const axesHelper = new THREE.AxesHelper(5);
  slopeContainer.add(axesHelper);

  // Ball (Visual)
  const sphereGeo = new THREE.SphereGeometry(0.5, 32, 32);
  const sphereMat = new THREE.MeshStandardMaterial({
    color: 0xff3e00,
    roughness: 0.4,
  });
  ballMesh = new THREE.Mesh(sphereGeo, sphereMat);
  ballMesh.castShadow = true;
  scene.add(ballMesh); // Ball is in world space, not slope space

  // Velocity Arrow
  arrowHelper = new THREE.ArrowHelper(
    new THREE.Vector3(1, 0, 0),
    new THREE.Vector3(0, 0, 0),
    2,
    0x0000ff
  );
  scene.add(arrowHelper);

  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  // Resize Handler
  window.addEventListener("resize", onWindowResize);
}

function onWindowResize() {
  if (!canvasContainer.value) return;
  const w = canvasContainer.value.clientWidth;
  const h = canvasContainer.value.clientHeight;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
}

// --- 5. PHYSICS SETUP (CANNON) ---

function initPhysics() {
  world = new CANNON.World();
  world.gravity.set(0, -params.gravity, 0);

  // Materials
  physicsMaterial = new CANNON.Material("physics");
  physicsContactMaterial = new CANNON.ContactMaterial(
    physicsMaterial,
    physicsMaterial,
    {
      friction: params.friction,
      restitution: 0.6, // Bounciness
    }
  );
  world.addContactMaterial(physicsContactMaterial);

  // Ground Body (Infinite Plane)
  const groundShape = new CANNON.Plane();
  groundBody = new CANNON.Body({ mass: 0, material: physicsMaterial }); // Static
  groundBody.addShape(groundShape);
  groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
  world.addBody(groundBody);

  // Ball Body
  const sphereShape = new CANNON.Sphere(0.5);
  ballBody = new CANNON.Body({
    mass: params.mass,
    material: physicsMaterial,
    shape: sphereShape,
    linearDamping: 0.01,
    angularDamping: 0.01,
  });
  world.addBody(ballBody);

  resetSimulation();
}

// --- 6. SIMULATION CONTROL ---

function resetSimulation() {
  isRunning.value = false;
  isFinished.value = false;
  currentTime.value = 0;
  dataLog = [];

  // 1. Configure World Parameters
  world.gravity.set(0, -params.gravity, 0);
  physicsContactMaterial.friction = params.friction;

  // 2. Configure Slope
  const radSlope = THREE.MathUtils.degToRad(params.slopeAngle);
  const groundQuat = new CANNON.Quaternion();
  groundQuat.setFromEuler(-Math.PI / 2 + radSlope, 0, 0);
  groundBody.quaternion.copy(groundQuat);
  slopeContainer.rotation.x = radSlope;

  // 3. Configure Ball Initial State
  ballBody.position.set(0, params.initialHeight, 0);
  ballBody.velocity.set(0, 0, 0);
  ballBody.angularVelocity.set(0, 0, 0);

  // 4. Calculate Launch Velocity Vector
  const radLaunch = THREE.MathUtils.degToRad(params.launchAngle);
  const vx = params.initialSpeed * Math.cos(radLaunch);
  const vy = params.initialSpeed * Math.sin(radLaunch);
  ballBody.velocity.set(vx, vy, 0);

  updateVisuals();
  updateLiveData();

  if (chartInstance) {
    chartInstance.data.labels = [];
    chartInstance.data.datasets[0].data = [];
    chartInstance.data.datasets[1].data = [];
    chartInstance.update();
  }
}

function startSimulation() {
  if (isFinished.value) {
    resetSimulation();
  }
  isRunning.value = true;
  lastCallTime = performance.now();
  animate();
}

function pauseSimulation() {
  isRunning.value = false;
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
}

function animate() {
  if (!isRunning.value) return;

  animationFrameId = requestAnimationFrame(animate);

  const now = performance.now();
  const dt = (now - lastCallTime) / 1000;
  lastCallTime = now;

  world.step(params.timeStep, dt, 3);
  currentTime.value += params.timeStep;

  updateVisuals();
  logData();
  updateLiveData();
  checkCollisions();

  controls.update();
  renderer.render(scene, camera);
}

// --- 7. LOGIC & UTILS ---

function updateVisuals() {
  ballMesh.position.copy(ballBody.position as any);
  ballMesh.quaternion.copy(ballBody.quaternion as any);

  const vel = ballBody.velocity;
  const speed = vel.length();
  if (speed > 0.1) {
    arrowHelper.position.copy(ballMesh.position);
    arrowHelper.setDirection(
      new THREE.Vector3(vel.x, vel.y, vel.z).normalize()
    );
    arrowHelper.setLength(Math.min(speed * 0.5, 5));
    arrowHelper.visible = true;
  } else {
    arrowHelper.visible = false;
  }
}

function updateLiveData() {
  liveData.t = currentTime.value.toFixed(2);
  liveData.x = ballBody.position.x.toFixed(2);
  liveData.y = ballBody.position.y.toFixed(2);
  liveData.z = ballBody.position.z.toFixed(2);

  const v = ballBody.velocity.length();
  liveData.v = v.toFixed(2);
  const f = ballBody.force.length();
  liveData.a = (f / params.mass).toFixed(2);
}

function logData() {
  dataLog.push({
    t: currentTime.value,
    x: ballBody.position.x,
    y: ballBody.position.y,
    z: ballBody.position.z,
    vx: ballBody.velocity.x,
    vy: ballBody.velocity.y,
    vz: ballBody.velocity.z,
    ax: ballBody.force.x / params.mass,
    ay: ballBody.force.y / params.mass,
    az: ballBody.force.z / params.mass,
  });
}

function checkCollisions() {
  if (ballBody.position.y < -20) {
    pauseSimulation();
    isFinished.value = true;
    if (autoPlot.value) updateChart();
  }
}

// --- 8. CHARTING ---

function initChart() {
  if (!chartCanvas.value) return;

  chartInstance = new Chart(chartCanvas.value, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        {
          label: "Height (Y)",
          data: [],
          borderColor: "rgb(255, 99, 132)",
          backgroundColor: "rgba(255, 99, 132, 0.5)",
          yAxisID: "y",
          tension: 0.1,
        },
        {
          label: "Velocity (m/s)",
          data: [],
          borderColor: "rgb(53, 162, 235)",
          backgroundColor: "rgba(53, 162, 235, 0.5)",
          yAxisID: "y1",
          tension: 0.1,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      interaction: { mode: "index", intersect: false },
      scales: {
        x: { title: { display: true, text: "Time (s)" } },
        y: {
          type: "linear",
          display: true,
          position: "left",
          title: { display: true, text: "Position (m)" },
        },
        y1: {
          type: "linear",
          display: true,
          position: "right",
          grid: { drawOnChartArea: false },
          title: { display: true, text: "Velocity (m/s)" },
        },
      },
    },
  });
}

function updateChart() {
  if (!chartInstance) return;
  const step = 5;
  const labels = [];
  const dataY = [];
  const dataV = [];
  for (let i = 0; i < dataLog.length; i += step) {
    const p = dataLog[i];
    labels.push(p.t.toFixed(2));
    dataY.push(p.y);
    dataV.push(Math.sqrt(p.vx * p.vx + p.vy * p.vy + p.vz * p.vz));
  }
  chartInstance.data.labels = labels;
  chartInstance.data.datasets[0].data = dataY;
  chartInstance.data.datasets[1].data = dataV;
  chartInstance.update();
}

// --- 9. CSV EXPORT ---

function downloadCSV() {
  if (dataLog.length === 0) return;
  let csv = "Time,PosX,PosY,PosZ,VelX,VelY,VelZ,AccX,AccY,AccZ\n";
  dataLog.forEach((row) => {
    csv += `${row.t.toFixed(3)},${row.x.toFixed(3)},${row.y.toFixed(3)},${row.z.toFixed(3)},` +
      `${row.vx.toFixed(3)},${row.vy.toFixed(3)},${row.vz.toFixed(3)},` +
      `${row.ax.toFixed(3)},${row.ay.toFixed(3)},${row.az.toFixed(3)}\n`;
  });
  const blob = new Blob([csv], { type: "text/csv" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `physics-sim-${new Date().toISOString()}.csv`;
  a.click();
  URL.revokeObjectURL(url);
}

watch(params, () => { if (!isRunning.value) resetSimulation(); });
</script>

<style scoped>
canvas { display: block; }
.drawer-side { scroll-behavior: smooth; }
</style>
