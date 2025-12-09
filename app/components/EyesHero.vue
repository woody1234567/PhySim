<!-- components/EyesHero.vue -->
<template>
  <section
    class="relative min-h-[60vh] lg:min-h-screen flex items-end justify-center overflow-hidden mb-6"
  >
    <canvas ref="canvasEl" class="absolute inset-0 w-full h-full"></canvas>

    <!-- Optional headline overlay -->
    <div class="relative z-10 px-6 text-center">
      <h1 class="text-3xl md:text-6xl font-extrabold tracking-tight">
        Eyes on You 👀
      </h1>
      <p class="mt-3 text-base md:text-lg opacity-80">
        Move your cursor — the eyes will follow.
      </p>
    </div>

    <!-- Subtle vignette -->
    <div
      class="pointer-events-none absolute inset-0 bg-gradient-to-b from-transparent via-transparent to-black/10"
    ></div>
  </section>
</template>

<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from "vue";
import * as THREE from "three";
// @ts-ignore
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
// @ts-ignore
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";

const canvasEl = ref<HTMLCanvasElement | null>(null);

let renderer: THREE.WebGLRenderer;
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let controls: any;
let animationId = 0;

// Eye groups
let leftEye!: THREE.Group;
let rightEye!: THREE.Group;
// Pupils kept for reference if needed, but movement is now rotational
// so we don't necessarily need to move these meshes directly.
let leftPupil!: THREE.Mesh;
let rightPupil!: THREE.Mesh;

// Reusable objects
const raycaster = new THREE.Raycaster();
const mouseNDC = new THREE.Vector2();
const planeZ = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0); // z=0 plane
const hitPoint = new THREE.Vector3();

// Track state
let isPointerActive = false;
let idleTime = 0;

// Eyeball/pupil sizing
const EYE_RADIUS = 0.6;
const PUPIL_RADIUS = 0.18;
const IRIS_RADIUS = 0.28;
// Max pupil offset logic is removed since we rotate the eye now.

function buildEye(): { group: THREE.Group; pupil: THREE.Mesh } {
  const group = new THREE.Group();

  // Basic Sclera (White sphere)
  const scleraGeo = new THREE.SphereGeometry(EYE_RADIUS, 48, 48);
  const scleraMat = new THREE.MeshPhysicalMaterial({
    color: 0xffffff,
    roughness: 0.35,
    metalness: 0.0,
    transmission: 0,
    reflectivity: 0.2,
  });
  const sclera = new THREE.Mesh(scleraGeo, scleraMat);
  group.add(sclera);

  // Geometry constants for caps
  const irisArc = IRIS_RADIUS / EYE_RADIUS;
  const pupilArc = PUPIL_RADIUS / EYE_RADIUS;

  // 1. Iris Cap (Point to Z+ by rotating X -90deg later)
  const irisR = EYE_RADIUS + 0.002;
  const irisGeo = new THREE.SphereGeometry(
    irisR,
    32,
    16,
    0,
    Math.PI * 2,
    0,
    irisArc
  );
  const irisMat = new THREE.MeshStandardMaterial({
    color: 0x3a6ea5,
    roughness: 0.6,
    metalness: 0.0,
  });
  const iris = new THREE.Mesh(irisGeo, irisMat);
  iris.rotation.x = -Math.PI / 2;
  group.add(iris);

  // 2. Pupil Cap
  const pupilR = irisR + 0.002;
  const pupilGeo = new THREE.SphereGeometry(
    pupilR,
    32,
    16,
    0,
    Math.PI * 2,
    0,
    pupilArc
  );
  const pupilMat = new THREE.MeshStandardMaterial({ color: 0x111111 });
  const pupil = new THREE.Mesh(pupilGeo, pupilMat);
  pupil.rotation.x = -Math.PI / 2;
  group.add(pupil);

  // 3. Occlusion Ring (Simulated by a spherical slice)
  const ringR = irisR + 0.001;
  const ringInner = irisArc * 0.9;
  const ringOuter = irisArc * 1.2;
  const ringGeo = new THREE.SphereGeometry(
    ringR,
    32,
    8,
    0,
    Math.PI * 2,
    ringInner,
    ringOuter - ringInner
  );
  const ringMat = new THREE.MeshBasicMaterial({
    color: 0x000000,
    transparent: true,
    opacity: 0.12,
    side: THREE.DoubleSide,
  });
  const ring = new THREE.Mesh(ringGeo, ringMat);
  ring.rotation.x = -Math.PI / 2;
  group.add(ring);

  // 4. Cornea (Glassy bulge)
  const corneaR = EYE_RADIUS + 0.005;
  const corneaGeo = new THREE.SphereGeometry(
    corneaR,
    48,
    48,
    0,
    Math.PI * 2,
    0,
    0.8 // A large cap to cover the front
  );
  const corneaMat = new THREE.MeshPhysicalMaterial({
    color: 0xffffff,
    roughness: 0.05,
    metalness: 0,
    transparent: true,
    opacity: 0.25,
    ior: 1.38,
  });
  const cornea = new THREE.Mesh(corneaGeo, corneaMat);
  cornea.rotation.x = -Math.PI / 2;
  group.add(cornea);

  return { group, pupil };
}

function lookAtPointForEye(eye: THREE.Group, target: THREE.Vector3) {
  // Simple LookAt with constraints
  // Smoothly interpolate current rotation to target rotation
  const startQ = eye.quaternion.clone();
  eye.lookAt(target);
  const targetQ = eye.quaternion.clone();
  eye.quaternion.copy(startQ).slerp(targetQ, 0.1);
}

function onPointerMove(e: PointerEvent) {
  if (!renderer || !camera) return;
  isPointerActive = true;
  const rect = renderer.domElement.getBoundingClientRect();
  mouseNDC.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
  mouseNDC.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
}

function onPointerLeave() {
  isPointerActive = false;
}

function onResize() {
  if (!renderer || !camera) return;
  const el = renderer.domElement;
  const w = el.clientWidth;
  const h = el.clientHeight;
  const dpr = Math.min(window.devicePixelRatio || 1, 2);
  renderer.setPixelRatio(dpr);
  renderer.setSize(w, h, false);
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
}

function createScene(canvas: HTMLCanvasElement) {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0xf6f7fb);

  camera = new THREE.PerspectiveCamera(35, 1, 0.1, 100);
  camera.position.set(0, 10, 20);

  renderer = new THREE.WebGLRenderer({
    canvas,
    antialias: true,
    powerPreference: "high-performance",
    alpha: true,
  });
  renderer.setClearColor(0x000000, 0);
  onResize();

  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;

  // Lights
  const hemi = new THREE.HemisphereLight(0xffffff, 0xcccccc, 0.8);
  scene.add(hemi);
  const key = new THREE.DirectionalLight(0xffffff, 0.9);
  key.position.set(3, 4, 5);
  scene.add(key);
  const rim = new THREE.DirectionalLight(0xffffff, 0.4);
  rim.position.set(-4, 2, -2);
  scene.add(rim);

  // Load Robot Model
  const loader = new GLTFLoader();
  loader.load("/models/robot/scene.gltf", (gltf: any) => {
    const robot = gltf.scene;
    robot.position.y = -2;
    robot.rotation.y = 1.57;
    scene.add(robot);

    // Create Eyes
    ({ group: leftEye, pupil: leftPupil } = buildEye());
    ({ group: rightEye, pupil: rightPupil } = buildEye());

    const EYE_SCALE = 1.1;
    leftEye.scale.set(EYE_SCALE, EYE_SCALE, EYE_SCALE);
    rightEye.scale.set(EYE_SCALE, EYE_SCALE, EYE_SCALE);

    // Positions adjusted by user
    leftEye.position.set(-0.7, 4.7, 1.5);
    rightEye.position.set(0.7, 4.7, 1.5);

    scene.add(leftEye, rightEye);
  });

  // Basic floor
  const floorGeo = new THREE.CircleGeometry(6, 64);
  const floorMat = new THREE.MeshBasicMaterial({
    color: 0x000000,
    transparent: true,
    opacity: 0.05,
  });
  const floor = new THREE.Mesh(floorGeo, floorMat);
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -2.0;
  floor.position.z = -0.5;
  scene.add(floor);
}

function animate() {
  animationId = requestAnimationFrame(animate);

  if (controls) controls.update();

  if (!leftEye || !rightEye) return;

  // Determine target
  let target: THREE.Vector3;
  if (isPointerActive) {
    raycaster.setFromCamera(mouseNDC, camera);
    raycaster.ray.intersectPlane(planeZ, hitPoint);
    target = hitPoint;
    idleTime = 0;
  } else {
    // Idle wandering
    idleTime += 0.016;
    const r = 0.6;
    const x = Math.cos(idleTime * 0.6) * r;
    const y = Math.sin(idleTime * 0.9) * r * 0.6;
    target = new THREE.Vector3(x, y, 0);
  }

  lookAtPointForEye(leftEye, target);
  lookAtPointForEye(rightEye, target);

  renderer.render(scene, camera);
}

function onVisibilityChange() {
  if (document.hidden) {
    cancelAnimationFrame(animationId);
  } else {
    animate();
  }
}

onMounted(() => {
  if (!canvasEl.value) return;
  createScene(canvasEl.value);

  // Events
  window.addEventListener("resize", onResize, { passive: true });
  renderer.domElement.addEventListener("pointermove", onPointerMove);
  renderer.domElement.addEventListener("pointerenter", onPointerMove);
  renderer.domElement.addEventListener("pointerleave", onPointerLeave);
  document.addEventListener("visibilitychange", onVisibilityChange);

  animate();
});

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener("resize", onResize);
  if (renderer) {
    const el = renderer.domElement;
    el.removeEventListener("pointermove", onPointerMove);
    el.removeEventListener("pointerenter", onPointerMove);
    el.removeEventListener("pointerleave", onPointerLeave);
  }
  document.removeEventListener("visibilitychange", onVisibilityChange);

  // Dispose
  scene?.traverse((obj) => {
    if ((obj as THREE.Mesh).geometry) (obj as THREE.Mesh).geometry.dispose?.();
    const mat = (obj as THREE.Mesh).material;
    if (Array.isArray(mat)) mat.forEach((m) => m.dispose?.());
    else (mat as THREE.Material)?.dispose?.();
  });
  renderer?.dispose();
});
</script>

<style scoped>
section {
  /* Smoothens the canvas edges on some displays */
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
canvas {
  display: block;
}
</style>
