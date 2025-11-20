<template>
  <div
    class="w-full h-full relative flex flex-col overflow-hidden font-sans bg-base-200"
  >
    <!-- DAISYUI DRAWER: LAYOUT -->
    <div
      class="drawer drawer-end h-[800px] w-full rounded-xl shadow-2xl overflow-hidden"
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
          <h2 class="text-xl font-bold mb-2 flex items-center gap-2">
            <span>⚙️ Configuration</span>
          </h2>

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
  // Note: Cannon planes are infinite and face +Z by default. We must rotate.
  const groundShape = new CANNON.Plane();
  groundBody = new CANNON.Body({ mass: 0, material: physicsMaterial }); // Static
  groundBody.addShape(groundShape);

  // Initial rotation to make it flat (facing up Y)
  groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);

  world.addBody(groundBody);

  // Ball Body
  const sphereShape = new CANNON.Sphere(0.5); // Radius matches visual
  ballBody = new CANNON.Body({
    mass: params.mass,
    material: physicsMaterial,
    shape: sphereShape,
    linearDamping: 0.01,
    angularDamping: 0.01,
  });

  world.addBody(ballBody);

  resetSimulation(); // Set initial positions
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

  // 2. Configure Slope (Rotate Ground Body)
  // Slope Angle alpha. Rotation around Z axis or X axis?
  // Let's rotate around X axis to tip it towards/away, or Z to tip left/right.
  // Let's rotate around Z so it looks like a ramp from side view.
  // Actually, user expects to shoot "forward". A slope usually means "uphill" or "downhill".
  // Let's rotate around X axis. Positive angle = Uphill.
  const radSlope = THREE.MathUtils.degToRad(params.slopeAngle);

  // Reset Ground Quaternion to flat first (-90 deg around X) then add slope
  const groundQuat = new CANNON.Quaternion();
  groundQuat.setFromEuler(-Math.PI / 2 + radSlope, 0, 0);
  groundBody.quaternion.copy(groundQuat);

  // Update Visual Slope Container
  slopeContainer.rotation.x = radSlope;

  // 3. Configure Ball Initial State
  // Position: Local to world, but often relative to slope surface in problems.
  // We will set Position (0, h, 0) in World Space.
  // If slope is active, we might want to spawn it *on* the slope or above 0.
  // Let's keep it simple: Spawn at x=0, z=0, y=initialHeight
  ballBody.position.set(0, params.initialHeight, 0);
  ballBody.velocity.set(0, 0, 0);
  ballBody.angularVelocity.set(0, 0, 0);

  // 4. Calculate Launch Velocity Vector
  // Speed v0, Angle theta.
  // Usually assume launching in +Z direction or +X direction? Let's use +X as "Forward".
  const radLaunch = THREE.MathUtils.degToRad(params.launchAngle);

  // vx = v * cos(theta)
  // vy = v * sin(theta)
  // vz = 0
  const vx = params.initialSpeed * Math.cos(radLaunch);
  const vy = params.initialSpeed * Math.sin(radLaunch);

  ballBody.velocity.set(vx, vy, 0);

  // Sync visuals immediately
  updateVisuals();
  updateLiveData();

  // Clear chart
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

  // Physics Step (Fixed time step for stability)
  // We use a fixed step for consistency, but call it based on frame time
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
  // Sync Ball
  ballMesh.position.copy(ballBody.position as any);
  ballMesh.quaternion.copy(ballBody.quaternion as any);

  // Sync Arrow Helper (Velocity)
  const vel = ballBody.velocity;
  const speed = vel.length();
  if (speed > 0.1) {
    arrowHelper.position.copy(ballMesh.position);
    arrowHelper.setDirection(
      new THREE.Vector3(vel.x, vel.y, vel.z).normalize()
    );
    arrowHelper.setLength(Math.min(speed * 0.5, 5)); // Scale arrow
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

  // Approximate acceleration (Force / Mass)
  // Cannon applies forces then updates velocity.
  // A simple way: gravity is constant, but contact forces vary.
  // We'll show the net force magnitude / mass
  const f = ballBody.force.length();
  liveData.a = (f / params.mass).toFixed(2); // Usually just gravity unless hitting something
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
  // Determine ground height at current X/Z based on slope
  // Or simpler: If y < radius (on flat slope) or collision event.
  // Cannon handles physical collision. We just want to stop logging/simulating if user wants.
  // Let's define "Finished" if it hits the ground and stays there (speed ~ 0)
  // OR just if y < -50 (fell off world)

  if (ballBody.position.y < -20) {
    pauseSimulation();
    isFinished.value = true;
    if (autoPlot.value) updateChart();
  }

  // Stop if rolling very slowly on ground
  if (
    currentTime.value > 1.0 &&
    ballBody.velocity.length() < 0.05 &&
    ballBody.position.y < 2
  ) {
    // Check if contact with ground
    // Simple heuristic: y is close to ground plane height at this x
    // For this demo, we just let it run until manually stopped or falls off
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
      interaction: {
        mode: "index",
        intersect: false,
      },
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

  // Downsample data for chart performance (every 5th frame)
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
    csv +=
      `${row.t.toFixed(3)},${row.x.toFixed(3)},${row.y.toFixed(
        3
      )},${row.z.toFixed(3)},` +
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

// Watchers
watch(params, () => {
  if (!isRunning.value) resetSimulation();
});
</script>

<style scoped>
/* Custom styling to ensure chart fits nicely */
canvas {
  display: block;
}
.drawer-side {
  scroll-behavior: smooth;
}
</style>
