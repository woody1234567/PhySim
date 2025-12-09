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
// Max pupil offset from center so it never leaves the sclera
const MAX_OFFSET = IRIS_RADIUS - PUPIL_RADIUS - 0.06;

function buildEye(): { group: THREE.Group; pupil: THREE.Mesh } {
  const group = new THREE.Group();

  // Sclera (white)
  const scleraGeo = new THREE.SphereGeometry(EYE_RADIUS, 48, 48);
  const scleraMat = new THREE.MeshPhysicalMaterial({
    color: 0xffffff,
    roughness: 0.35,
    metalness: 0.0,
    transmission: 0, // opaque
    reflectivity: 0.2,
  });
  const sclera = new THREE.Mesh(scleraGeo, scleraMat);
  group.add(sclera);

  // Slight cornea bulge (subtle specular)
  const corneaGeo = new THREE.SphereGeometry(EYE_RADIUS * 1.01, 48, 48);
  const corneaMat = new THREE.MeshPhysicalMaterial({
    color: 0xffffff,
    roughness: 0.05,
    metalness: 0,
    transparent: true,
    opacity: 0.25,
    ior: 1.38,
  });
  const cornea = new THREE.Mesh(corneaGeo, corneaMat);
  group.add(cornea);

  // Iris (disk sitting slightly above sclera)
  const irisGeo = new THREE.CircleGeometry(IRIS_RADIUS, 48);
  const irisMat = new THREE.MeshStandardMaterial({
    color: 0x3a6ea5, // blue iris; feel free to tweak
    roughness: 0.6,
    metalness: 0.0,
  });
  const iris = new THREE.Mesh(irisGeo, irisMat);
  iris.position.z = EYE_RADIUS + 0.01;
  group.add(iris);

  // Pupil (circle on top of iris, we will move this mesh)
  const pupilGeo = new THREE.CircleGeometry(PUPIL_RADIUS, 48);
  const pupilMat = new THREE.MeshStandardMaterial({ color: 0x111111 });
  const pupil = new THREE.Mesh(pupilGeo, pupilMat);
  pupil.position.z = iris.position.z + 0.005;
  group.add(pupil);

  // Soft ambient occlusion ring (fake shadow)
  const ringGeo = new THREE.RingGeometry(
    IRIS_RADIUS * 0.92,
    IRIS_RADIUS * 1.2,
    64
  );
  const ringMat = new THREE.MeshBasicMaterial({
    color: 0x000000,
    transparent: true,
    opacity: 0.12,
    side: THREE.DoubleSide,
  });
  const ring = new THREE.Mesh(ringGeo, ringMat);
  ring.position.z = iris.position.z + 0.001;
  group.add(ring);

  return { group, pupil };
}

function clampOffset(vec: THREE.Vector3, maxLen: number) {
  const len = Math.hypot(vec.x, vec.y);
  if (len > maxLen && len > 0) {
    const s = maxLen / len;
    vec.x *= s;
    vec.y *= s;
  }
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
  camera.position.set(0, 10, 20); // Moved back slightly to see the whole robot

  renderer = new THREE.WebGLRenderer({
    canvas,
    antialias: true,
    powerPreference: "high-performance",
    alpha: true, // Enable transparency for the canvas
  });
  renderer.setClearColor(0x000000, 0); // Transparent background
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
    // Adjust scale and position based on typical model sizes; may need tuning
    // robot.scale.set(1.5, 1.5, 1.5);
    robot.position.y = -2; // Move down to center the face
    robot.rotation.y = 1.57;
    scene.add(robot);

    // Create Eyes and attach to robot or scene
    // Because the robot might have its own transforms, let's add eyes to the scene for now
    // and position them where the robot's eyes likely are.

    ({ group: leftEye, pupil: leftPupil } = buildEye());
    ({ group: rightEye, pupil: rightPupil } = buildEye());

    // Scale eyes down to fit the robot face (assuming robot head is roughly human-sized scaled up)
    const EYE_SCALE = 1.1;
    leftEye.scale.set(EYE_SCALE, EYE_SCALE, EYE_SCALE);
    rightEye.scale.set(EYE_SCALE, EYE_SCALE, EYE_SCALE);

    // Estimated positions for the robot's eye sockets
    leftEye.position.set(-0.7, 4.7, 1.5);
    rightEye.position.set(0.7, 4.7, 1.5);

    scene.add(leftEye, rightEye);
  });

  // Soft floor to ground the scene visually
  const floorGeo = new THREE.CircleGeometry(6, 64);
  const floorMat = new THREE.MeshBasicMaterial({
    color: 0x000000,
    transparent: true,
    opacity: 0.05,
  });
  const floor = new THREE.Mesh(floorGeo, floorMat);
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -2.0; // aligned with robot feet
  floor.position.z = -0.5;
  scene.add(floor);
}

function lookAtPointForEye(
  eye: THREE.Group,
  pupil: THREE.Mesh,
  target: THREE.Vector3
) {
  // Convert target into the eye's local space (so we can compute local x/y offsets)
  const local = eye.worldToLocal(target.clone());

  // Project onto the eye's front plane (z ~ EYE_RADIUS)
  // We want a 2D offset on the iris plane
  const offset = new THREE.Vector3(local.x, local.y, 0);

  // Scale down sensitivity
  offset.multiplyScalar(0.35);
  clampOffset(offset, MAX_OFFSET);

  // Animate toward target (lerp for smoothness)
  pupil.position.x = THREE.MathUtils.lerp(pupil.position.x, offset.x, 0.2);
  pupil.position.y = THREE.MathUtils.lerp(pupil.position.y, offset.y, 0.2);

  // Slight eyeball rotation to sell the effect
  const ROT_MAX = 0.18;
  eye.rotation.y = THREE.MathUtils.clamp(
    THREE.MathUtils.lerp(eye.rotation.y, -offset.x * 0.25, 0.18),
    -ROT_MAX,
    ROT_MAX
  );
  eye.rotation.x = THREE.MathUtils.clamp(
    THREE.MathUtils.lerp(eye.rotation.x, offset.y * 0.25, 0.18),
    -ROT_MAX,
    ROT_MAX
  );
}

function animate() {
  animationId = requestAnimationFrame(animate);

  if (controls) controls.update();

  if (!leftEye || !rightEye) return;

  // Determine a 3D target point on z=0 plane
  let target: THREE.Vector3;
  if (isPointerActive) {
    raycaster.setFromCamera(mouseNDC, camera);
    raycaster.ray.intersectPlane(planeZ, hitPoint);
    target = hitPoint;
    idleTime = 0;
  } else {
    // Idle: slow, subtle wandering so it doesn’t feel static on touch devices
    idleTime += 0.016;
    const r = 0.6;
    const x = Math.cos(idleTime * 0.6) * r;
    const y = Math.sin(idleTime * 0.9) * r * 0.6;
    target = new THREE.Vector3(x, y, 0);
  }

  lookAtPointForEye(leftEye, leftPupil, target);
  lookAtPointForEye(rightEye, rightPupil, target);

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
