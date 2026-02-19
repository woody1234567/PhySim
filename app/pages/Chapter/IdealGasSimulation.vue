<template>
  <div
    class="flex h-screen w-full overflow-hidden bg-base-100 text-base-content"
    @mouseup="stopResize"
    @mousemove="onResize"
  >
    <!-- 3D Simulation Canvas -->
    <div
      ref="canvasContainer"
      class="relative flex-grow h-full overflow-hidden bg-black"
      :style="{ flexBasis: `calc(100% - ${sidebarWidth}px)` }"
    >
      <canvas ref="canvas" class="block w-full h-full outline-none"></canvas>

      <!-- Overlay Info -->
      <div
        class="absolute top-4 left-4 bg-black/50 text-white p-2 rounded pointer-events-none select-none"
      >
        <h2 class="font-bold">Ideal Gas Simulation</h2>
        <p class="text-xs">Particles: {{ particleCount }}</p>
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
        <div class="modal-box max-w-2xl bg-base-100 border border-base-300 text-base-content">
          <h3 class="font-bold text-2xl mb-4 flex items-center gap-2">
            <Icon name="heroicons:academic-cap" class="text-primary text-3xl" />
            Physics Concepts: Ideal Gas Law
          </h3>

          <div class="space-y-4 text-sm leading-relaxed overflow-y-auto max-h-[70vh] pr-2">
            <section>
              <h4 class="font-bold text-lg text-secondary">1. The State Equation</h4>
              <p>
                The relationship between pressure, volume, and temperature for an ideal gas is given by:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="P \cdot V = n \cdot R \cdot T" :inline="false" />
              </div>
              <p>
                In this microscopic simulation, we use the equivalent form:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="P \cdot V = N \cdot k \cdot T" :inline="false" />
              </div>
              <p>
                Where <MathLatex formula="N" /> is the number of particles and <MathLatex formula="k" /> is the Boltzmann constant.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">2. Kinetic Theory of Gases</h4>
              <p>
                Temperature is a measure of the average kinetic energy of the particles:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="\bar{K} = \frac{3}{2} k T" :inline="false" />
              </div>
              <p>
                The Root-Mean-Square (RMS) velocity of a particle is:
              </p>
              <div class="bg-base-200 p-3 rounded-lg font-mono text-center my-2 italic">
                <MathLatex formula="v_{rms} = \sqrt{\frac{3 k T}{m}}" :inline="false" />
              </div>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">3. Pressure Origins</h4>
              <p>
                Pressure results from the continuous collisions of particles with the container walls. Each collision transfers momentum (<MathLatex formula="\Delta p" />), exerting a force on the area.
              </p>
            </section>

            <section>
              <h4 class="font-bold text-lg text-secondary">4. Controls & Observations</h4>
              <ul class="list-disc ml-6 space-y-2">
                <li><strong>Temperature (T):</strong> Increases particle speed and collision frequency/force.</li>
                <li><strong>Volume (V):</strong> Controlled by Container Size (L). Decreasing volume increases collision frequency.</li>
                <li><strong>Particle Count (N):</strong> More particles lead to higher pressure due to more frequent wall impacts.</li>
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
      class="w-1 bg-base-300 hover:bg-primary cursor-col-resize flex-none z-50 transition-colors"
      @mousedown="startResize"
    ></div>

    <!-- Sidebar -->
    <div
      class="flex flex-col h-full bg-base-200 overflow-y-auto overflow-x-hidden border-l border-base-300 shadow-xl"
      :style="{
        width: `${sidebarWidth}px`,
        minWidth: '260px',
        maxWidth: '600px',
      }"
    >
      <!-- Panel 1: Controls -->
      <div
        class="collapse collapse-arrow border-b border-base-300 bg-base-100 rounded-none"
      >
        <input type="checkbox" checked />
        <div class="collapse-title text-lg font-medium flex justify-between items-center pr-12">
          <span>Control Panel</span>
          <UButton
            icon="i-lucide-book-open"
            color="neutral"
            variant="ghost"
            size="xs"
            @click.stop="showHelpModal = true"
          />
        </div>
        <div class="collapse-content space-y-4">
          <!-- Simulation Control -->
          <div class="flex gap-2">
            <button
              class="btn btn-primary flex-1 btn-sm"
              @click="resetSimulation"
            >
              Restart
            </button>
            <button
              class="btn btn-secondary flex-1 btn-sm"
              @click="togglePause"
            >
              {{ isPaused ? "Resume" : "Pause" }}
            </button>
          </div>

          <div class="divider my-0"></div>

          <!-- Parameters -->
          <!-- Temperature (T) -->
          <div class="form-control">
            <label class="label">
              <span class="label-text">Temperature (T)</span>
              <span class="label-text-alt">{{ temperature.toFixed(1) }} K</span>
            </label>
            <input
              type="range"
              min="50"
              max="1000"
              step="10"
              class="range range-xs range-error"
              v-model.number="temperature"
              @input="onTemperatureChange"
            />
          </div>

          <!-- Volume (V) - Controlled by Box Size L -->
          <div class="form-control">
            <label class="label">
              <span class="label-text">Container Size (L)</span>
              <span class="label-text-alt">{{ boxSize.toFixed(1) }} m</span>
            </label>
            <input
              type="range"
              min="5"
              max="25"
              step="0.5"
              class="range range-xs range-info"
              v-model.number="boxSize"
              @input="onVolumeChange"
            />
            <div class="text-xs text-center mt-1 text-base-content/60">
              Volume ≈ {{ Math.pow(boxSize, 3).toFixed(0) }} m³
            </div>
          </div>

          <!-- Particle Count (N) -->
          <div class="form-control">
            <label class="label">
              <span class="label-text">Particles (N)</span>
              <span class="label-text-alt">{{ particleCount }}</span>
            </label>
            <input
              type="range"
              min="10"
              max="500"
              step="10"
              class="range range-xs range-success"
              v-model.number="particleCount"
              @change="resetSimulation"
            />
          </div>
        </div>
      </div>

      <!-- Panel 2: Live Data -->
      <div
        class="collapse collapse-arrow border-b border-base-300 bg-base-100 rounded-none"
      >
        <input type="checkbox" checked />
        <div class="collapse-title text-lg font-medium">Live Data</div>
        <div class="collapse-content">
          <div class="overflow-x-auto">
            <table class="table table-xs w-full">
              <tbody>
                <tr>
                  <td>Pressure (P)</td>
                  <td class="font-mono font-bold text-right">
                    {{ currentPressure.toFixed(2) }} Pa
                  </td>
                </tr>
                <tr>
                  <td>Temperature (T)</td>
                  <td class="font-mono text-right">
                    {{ currentTemperature.toFixed(2) }} K
                  </td>
                </tr>
                <tr>
                  <td>Volume (V)</td>
                  <td class="font-mono text-right">
                    {{ Math.pow(boxSize, 3).toFixed(1) }} m³
                  </td>
                </tr>
                <tr>
                  <td>Particles (N)</td>
                  <td class="font-mono text-right">{{ particles.length }}</td>
                </tr>
                <tr>
                  <td>Kinetic Energy</td>
                  <td class="font-mono text-right">
                    {{ totalKineticEnergy.toFixed(1) }} J
                  </td>
                </tr>
                <tr>
                  <td>PV / nRT</td>
                  <td
                    class="font-mono text-right"
                    :class="{
                      'text-success': Math.abs(pv_nrt - 1) < 0.1,
                      'text-error': Math.abs(pv_nrt - 1) >= 0.1,
                    }"
                  >
                    {{ pv_nrt.toFixed(3) }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
          <button
            class="btn btn-outline btn-xs w-full mt-4"
            @click="exportLogsToCSV"
          >
            Export Simulation Data (CSV)
          </button>
        </div>
      </div>

      <!-- Panel 3: Charts -->
      <div
        class="collapse collapse-arrow border-b border-base-300 bg-base-100 rounded-none"
      >
        <input type="checkbox" />
        <div class="collapse-title text-lg font-medium">Analysis Charts</div>
        <div class="collapse-content space-y-4">
          <div class="form-control">
            <label class="label"
              ><span class="label-text">Chart Type</span></label
            >
            <select
              class="select select-bordered select-xs w-full"
              v-model="selectedChartType"
            >
              <option value="PT">Pressure vs Temperature</option>
              <option value="PV">Pressure vs Volume</option>
              <option value="VT">Volume vs Temperature</option>
            </select>
          </div>

          <div class="h-64 w-full bg-white rounded p-2">
            <canvas ref="chartCanvas"></canvas>
          </div>

          <div class="flex gap-2">
            <button class="btn btn-primary btn-xs flex-1" @click="plotChart">
              Plot Chart
            </button>
            <button
              class="btn btn-ghost btn-xs flex-1"
              @click="exportChartData"
            >
              Export CSV
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import * as CANNON from "cannon-es";
import Chart from "chart.js/auto";
import { ref, onMounted, onUnmounted, watch, computed } from "vue";

// --- Types & Constants ---

// Physics Constants
// To make numbers readable, we'll use a "Simulation Boltzmann" constant
const SIM_K = 1.0;
const PARTICLE_MASS = 1.0;
const PARTICLE_RADIUS = 0.2;
const SUBSTEPS = 5;
const TIME_STEP = 1 / 60 / SUBSTEPS;

// Materials
const wallMaterial = new CANNON.Material("wall");
const particleMaterial = new CANNON.Material("particle");

const particleWallContact = new CANNON.ContactMaterial(
  particleMaterial,
  wallMaterial,
  {
    friction: 0.0,
    restitution: 1.0, // Perfectly elastic collisions
  },
);

// --- UI State ---
const sidebarWidth = ref(320);
const isResizing = ref(false);
const canvasContainer = ref<HTMLElement | null>(null);
const canvas = ref<HTMLCanvasElement | null>(null);
const chartCanvas = ref<HTMLCanvasElement | null>(null);

// --- Simulation Parameters ---
const temperature = ref(300); // Kelvin (simulated)
const boxSize = ref(10); // Length of side of cube
const particleCount = ref(100);

const isPaused = ref(false);
const showHelpModal = ref(true); // Auto-show on first load

// --- Simulation Metrics ---
const currentPressure = ref(0);
const currentTemperature = ref(0);
const totalKineticEnergy = ref(0);

const pv_nrt = computed(() => {
  const V = Math.pow(boxSize.value, 3);
  const nkT =
    particleCount.value * SIM_K * Math.max(currentTemperature.value, 1);
  if (nkT === 0) return 0;
  return (currentPressure.value * V) / nkT;
});

// --- Three.js & Cannon Objects ---
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let world: CANNON.World;
let particles: CANNON.Body[] = [];
let particleMesh: THREE.InstancedMesh;
const dummy = new THREE.Object3D();

// Walls
const wallBodies: CANNON.Body[] = []; // 6 walls
let wallMesh: THREE.LineSegments; // Wireframe box

// Chart
let chartInstance: Chart | null = null;
const selectedChartType = ref<"PT" | "PV" | "VT">("PT");

// Data Logging
interface LogData {
  time: number;
  pressure: number;
  temperature: number;
  volume: number;
  particleCount: number;
  kineticEnergy: number;
}
const logs = ref<LogData[]>([]);
let simTime = 0;

// --- Sidebar Resizing ---
const startResize = () => {
  isResizing.value = true;
  document.body.style.cursor = "col-resize";
};
const stopResize = () => {
  isResizing.value = false;
  document.body.style.cursor = "";
};
const onResize = (e: MouseEvent) => {
  if (!isResizing.value) return;
  const newWidth = Math.min(Math.max(window.innerWidth - e.clientX, 260), 600);
  if (window.innerWidth - newWidth < 300) return; // Min canvas width
  sidebarWidth.value = newWidth;
  if (camera && renderer) handleResize();
};

// --- Initialization ---

onMounted(() => {
  initThree();
  initPhysics();
  initOrbitControls();
  initChart();
  resetSimulation();
  animate();
  window.addEventListener("resize", handleResize);
});

onUnmounted(() => {
  window.removeEventListener("resize", handleResize);
  if (renderer) renderer.dispose();
  if (chartInstance) chartInstance.destroy();
});

const initThree = () => {
  if (!canvas.value || !canvasContainer.value) return;

  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);

  camera = new THREE.PerspectiveCamera(
    50,
    canvasContainer.value.clientWidth / canvasContainer.value.clientHeight,
    0.1,
    1000,
  );
  camera.position.set(25, 25, 25);
  camera.lookAt(0, 0, 0);

  renderer = new THREE.WebGLRenderer({ canvas: canvas.value, antialias: true });
  renderer.setSize(
    canvasContainer.value.clientWidth,
    canvasContainer.value.clientHeight,
  );
  renderer.setPixelRatio(window.devicePixelRatio);

  // Lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);
  const pointLight = new THREE.PointLight(0xffffff, 1);
  pointLight.position.set(20, 20, 20);
  scene.add(pointLight);

  // Wireframe Box Helper
  const geometry = new THREE.BoxGeometry(1, 1, 1);
  const edges = new THREE.EdgesGeometry(geometry);
  wallMesh = new THREE.LineSegments(
    edges,
    new THREE.LineBasicMaterial({ color: 0x44aa88 }),
  );
  scene.add(wallMesh);
};

const initPhysics = () => {
  world = new CANNON.World();
  world.gravity.set(0, 0, 0);
  world.addContactMaterial(particleWallContact);

  // Create 6 planes for walls (updated dynamically)
  for (let i = 0; i < 6; i++) {
    const wallBody = new CANNON.Body({
      mass: 0, // Static
      material: wallMaterial,
    });
    wallBody.addShape(new CANNON.Plane());
    world.addBody(wallBody);
    wallBodies.push(wallBody);
  }
};

const initOrbitControls = () => {
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
};

const updateWallPositions = () => {
  const halfSize = boxSize.value / 2;

  // Visual Mesh
  wallMesh.scale.set(boxSize.value, boxSize.value, boxSize.value);

  // Cannon Planes
  wallBodies[0].position.set(halfSize, 0, 0);
  wallBodies[0].quaternion.setFromEuler(0, -Math.PI / 2, 0);

  wallBodies[1].position.set(-halfSize, 0, 0);
  wallBodies[1].quaternion.setFromEuler(0, Math.PI / 2, 0);

  wallBodies[2].position.set(0, halfSize, 0);
  wallBodies[2].quaternion.setFromEuler(Math.PI / 2, 0, 0);

  wallBodies[3].position.set(0, -halfSize, 0);
  wallBodies[3].quaternion.setFromEuler(-Math.PI / 2, 0, 0);

  wallBodies[4].position.set(0, 0, halfSize);
  wallBodies[4].quaternion.setFromEuler(0, Math.PI, 0);

  wallBodies[5].position.set(0, 0, -halfSize);
  wallBodies[5].quaternion.setFromEuler(0, 0, 0);
};

// --- Simulation Logic ---

const resetSimulation = () => {
  // Clear particles
  particles.forEach((p) => world.removeBody(p));
  particles = [];
  logs.value = [];
  simTime = 0;

  // Re-create particles
  const geom = new THREE.SphereGeometry(PARTICLE_RADIUS, 8, 8);
  const mat = new THREE.MeshStandardMaterial({ color: 0xffaa00 });
  if (particleMesh) scene.remove(particleMesh);
  particleMesh = new THREE.InstancedMesh(geom, mat, particleCount.value);
  scene.add(particleMesh);

  // Init velocities based on Temperature
  const v_rms = Math.sqrt((3 * SIM_K * temperature.value) / PARTICLE_MASS);
  const halfL = boxSize.value / 2 - PARTICLE_RADIUS;

  for (let i = 0; i < particleCount.value; i++) {
    const body = new CANNON.Body({
      mass: PARTICLE_MASS,
      shape: new CANNON.Sphere(PARTICLE_RADIUS),
      material: particleMaterial,
      linearDamping: 0,
      angularDamping: 0,
    });

    body.position.set(
      (Math.random() - 0.5) * 2 * halfL,
      (Math.random() - 0.5) * 2 * halfL,
      (Math.random() - 0.5) * 2 * halfL,
    );

    const theta = Math.random() * Math.PI * 2;
    const phi = Math.acos(2 * Math.random() - 1);
    const vx = v_rms * Math.sin(phi) * Math.cos(theta);
    const vy = v_rms * Math.sin(phi) * Math.sin(theta);
    const vz = v_rms * Math.cos(phi);

    body.velocity.set(vx, vy, vz);

    world.addBody(body);
    particles.push(body);
  }

  updateWallPositions();
  updateMeasuredStats();
};

const thermalize = (targetT: number) => {
  let totalKE = 0;
  particles.forEach((p) => {
    totalKE += 0.5 * p.mass * p.velocity.lengthSquared();
  });
  const avgKE = totalKE / particles.length;
  const currentT = (2 * avgKE) / (3 * SIM_K);

  if (currentT <= 0.001) return;

  const scale = Math.sqrt(targetT / currentT);
  particles.forEach((p) => {
    p.velocity.scale(scale, p.velocity);
  });
};

const constrainParticles = () => {
  const halfL = boxSize.value / 2 - PARTICLE_RADIUS;
  particles.forEach((p) => {
    if (p.position.x > halfL) { p.position.x = halfL; p.velocity.x = -Math.abs(p.velocity.x); }
    else if (p.position.x < -halfL) { p.position.x = -halfL; p.velocity.x = Math.abs(p.velocity.x); }
    if (p.position.y > halfL) { p.position.y = halfL; p.velocity.y = -Math.abs(p.velocity.y); }
    else if (p.position.y < -halfL) { p.position.y = -halfL; p.velocity.y = Math.abs(p.velocity.y); }
    if (p.position.z > halfL) { p.position.z = halfL; p.velocity.z = -Math.abs(p.velocity.z); }
    else if (p.position.z < -halfL) { p.position.z = -halfL; p.velocity.z = Math.abs(p.velocity.z); }
  });
};

const stepSimulation = () => {
  if (isPaused.value) return;

  thermalize(temperature.value);

  for (let i = 0; i < SUBSTEPS; i++) {
    world.step(TIME_STEP);
    constrainParticles();
  }

  simTime += TIME_STEP * SUBSTEPS;
  updateMeasuredStats();

  if (Math.floor(simTime / (TIME_STEP * SUBSTEPS)) % 10 === 0) {
    logData();
  }
};

const updateMeasuredStats = () => {
  let ke = 0;
  particles.forEach((p) => {
    ke += 0.5 * p.mass * p.velocity.lengthSquared();
  });
  totalKineticEnergy.value = ke;

  const N = Math.max(1, particles.length);
  currentTemperature.value = (2 * (ke / N)) / (3 * SIM_K);

  const V = Math.pow(boxSize.value, 3);
  if (V > 0) {
    currentPressure.value = (2 * totalKineticEnergy.value) / (3 * V);
  } else {
    currentPressure.value = 0;
  }
};

const logData = () => {
  logs.value.push({
    time: Number(simTime.toFixed(2)),
    pressure: Number(currentPressure.value.toFixed(2)),
    temperature: Number(currentTemperature.value.toFixed(2)),
    volume: Number(Math.pow(boxSize.value, 3).toFixed(2)),
    particleCount: particleCount.value,
    kineticEnergy: Number(totalKineticEnergy.value.toFixed(2)),
  });

  if (logs.value.length > 2000) logs.value.shift();
};

const animate = () => {
  requestAnimationFrame(animate);
  stepSimulation();

  for (let i = 0; i < particles.length; i++) {
    dummy.position.copy(particles[i].position as any);
    dummy.updateMatrix();
    particleMesh.setMatrixAt(i, dummy.matrix);
  }
  particleMesh.instanceMatrix.needsUpdate = true;

  controls.update();
  renderer.render(scene, camera);
};

// --- UI Handlers ---

const togglePause = () => {
  isPaused.value = !isPaused.value;
};

const onTemperatureChange = () => {};
const onVolumeChange = () => { updateWallPositions(); };

const handleResize = () => {
  if (!canvasContainer.value || !renderer || !camera) return;
  const width = canvasContainer.value.clientWidth;
  const height = canvasContainer.value.clientHeight;
  renderer.setSize(width, height);
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
};

// --- Charting ---

const initChart = () => {
  if (!chartCanvas.value) return;
  chartInstance = new Chart(chartCanvas.value, {
    type: "scatter",
    data: { datasets: [] },
    options: {
      responsive: true,
      plugins: { legend: { display: false } },
      scales: {
        x: { title: { display: true, text: "X Axis" } },
        y: { title: { display: true, text: "Y Axis" } },
      },
    },
  });
};

const plotChart = () => {
  if (!chartInstance) return;

  const data = logs.value.map((l) => {
    let x = 0, y = 0;
    if (selectedChartType.value === "PT") { x = l.temperature; y = l.pressure; }
    else if (selectedChartType.value === "PV") { x = l.volume; y = l.pressure; }
    else if (selectedChartType.value === "VT") { x = l.temperature; y = l.volume; }
    return { x, y };
  });

  let xLabel = "", yLabel = "";
  if (selectedChartType.value === "PT") { xLabel = "Temperature (K)"; yLabel = "Pressure (Pa)"; }
  else if (selectedChartType.value === "PV") { xLabel = "Volume (m³)"; yLabel = "Pressure (Pa)"; }
  else if (selectedChartType.value === "VT") { xLabel = "Temperature (K)"; yLabel = "Volume (m³)"; }

  chartInstance.data = {
    datasets: [{
      label: selectedChartType.value,
      data: data,
      backgroundColor: "rgba(75, 192, 192, 1)",
    }],
  };

  if (chartInstance.options.scales?.x?.title) chartInstance.options.scales.x.title.text = xLabel;
  if (chartInstance.options.scales?.y?.title) chartInstance.options.scales.y.title.text = yLabel;

  chartInstance.update();
};

// --- Export ---
const exportLogsToCSV = () => {
  const headers = ["Time", "Pressure", "Temperature", "Volume", "ParticleCount", "KineticEnergy"];
  const rows = logs.value.map((l) =>
    [l.time, l.pressure, l.temperature, l.volume, l.particleCount, l.kineticEnergy].join(",")
  );
  downloadCSV("simulation_data.csv", [headers.join(","), ...rows].join("\n"));
};

const exportChartData = () => {
  if (!chartInstance) return;
  const dataset = chartInstance.data.datasets[0];
  if (!dataset) return;

  const data = dataset.data as { x: number; y: number }[];
  let headers = ["X", "Y"];
  if (selectedChartType.value === "PT") headers = ["Temperature", "Pressure"];
  if (selectedChartType.value === "PV") headers = ["Volume", "Pressure"];
  if (selectedChartType.value === "VT") headers = ["Temperature", "Volume"];

  const rows = data.map((pt) => `${pt.x},${pt.y}`);
  downloadCSV(`chart_data_${selectedChartType.value}.csv`, [headers.join(","), ...rows].join("\n"));
};

const downloadCSV = (filename: string, content: string) => {
  const blob = new Blob([content], { type: "text/csv;charset=utf-8;" });
  const link = document.createElement("a");
  if (link.download !== undefined) {
    const url = URL.createObjectURL(blob);
    link.setAttribute("href", url);
    link.setAttribute("download", filename);
    link.style.visibility = "hidden";
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  }
};
</script>

<style scoped>
::-webkit-scrollbar { width: 8px; height: 8px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: #888; border-radius: 4px; }
::-webkit-scrollbar-thumb:hover { background: #555; }
</style>
