<template>
  <div class="relative w-full h-[80vh] bg-black overflow-hidden">
    <canvas
      ref="canvasRef"
      class="absolute top-0 left-0 flex h-full block outline-none"
    ></canvas>
    <div
      class="absolute inset-0 flex items-center justify-end pointer-events-none transition-opacity duration-1000"
      :class="{ 'opacity-90': showText, 'opacity-0': !showText }"
    >
      <div
        class="text-center text-white px-4 max-w-3xl bg-black/40 backdrop-blur-sm rounded-xl p-8 border border-white/10 shadow-2xl"
      >
        <h1
          class="text-4xl md:text-6xl font-bold mb-6 drop-shadow-[0_0_15px_rgba(255,255,255,0.5)] leading-tight"
        >
          Learning physics <br />
          <span class="text-primary">with world in your hand</span>
        </h1>
        <p
          class="text-xl md:text-3xl font-light drop-shadow-md text-gray-200 mb-8"
        >
          PhysSim, your learning companion.
        </p>
        <button
          class="btn btn-primary btn-lg pointer-events-auto hover:scale-105 transition-transform shadow-[0_0_20px_rgba(var(--p),0.5)]"
          @click="$emit('scrollToSims')"
        >
          Start Exploring
        </button>
      </div>
    </div>

    <div
      v-if="loading"
      class="absolute inset-0 flex items-center justify-center bg-black z-10"
    >
      <span
        class="loading loading-infinity loading-lg text-primary scale-150"
      ></span>
    </div>
  </div>
</template>

<script setup>
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

const canvasRef = ref(null);
const showText = ref(false);
const loading = ref(true);

let scene, camera, renderer, model;
let animationId;

const initThree = () => {
  // Scene setup
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x050505); // Very dark gray, almost black
  scene.fog = new THREE.FogExp2(0x050505, 0.002); // Add fog for depth

  // // Add grid helper
  // const grid = new THREE.GridHelper(30, 30, 0x334155, 0x1e293b);
  // grid.position.y = -10;
  // scene.add(grid);

  // Camera setup
  camera = new THREE.PerspectiveCamera(
    45,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  );
  // Initial camera position (far away)
  camera.position.set(0, 5, 15);

  // Renderer setup
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
    alpha: true,
  });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.2;

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 2);
  dirLight.position.set(5, 10, 7);
  dirLight.castShadow = true;
  dirLight.shadow.mapSize.width = 1024;
  dirLight.shadow.mapSize.height = 1024;
  scene.add(dirLight);

  // Add a blue rim light for sci-fi feel
  const spotLight = new THREE.SpotLight(0x00ffff, 5);
  spotLight.position.set(-5, 5, -5);
  spotLight.lookAt(0, 0, 0);
  scene.add(spotLight);

  // Load Model
  const loader = new GLTFLoader();
  loader.load(
    "/models/need_some_space/scene.gltf",
    (gltf) => {
      model = gltf.scene;

      // Center the model
      const box = new THREE.Box3().setFromObject(model);
      const center = box.getCenter(new THREE.Vector3());
      model.position.sub(center); // Center at 0,0,0

      // Scale if necessary (adjust based on model size)
      // model.scale.set(1, 1, 1);

      model.traverse((child) => {
        if (child.isMesh) {
          child.castShadow = true;
          child.receiveShadow = true;
        }
      });

      scene.add(model);
      loading.value = false;

      // Start animation loop
      animate();
    },
    undefined,
    (error) => {
      console.error("An error occurred loading the model:", error);
      loading.value = false;
    }
  );

  // Handle resize
  window.addEventListener("resize", onWindowResize);
};

const targetCameraPosition = new THREE.Vector3(0, 1, 3); // Target close-up position
const lookAtPosition = new THREE.Vector3(0, 0, 0);

const animate = () => {
  animationId = requestAnimationFrame(animate);

  if (model) {
    // Rotate model slowly
    model.rotation.y += 0.001;

    // let the model stop at rotation 0.7
    if (model.rotation.y > 0.7) {
      model.rotation.y -= 0.001;
    }
  }

  // Smooth camera movement (Zoom in)
  camera.position.lerp(targetCameraPosition, 0.02);
  camera.lookAt(lookAtPosition);
  // Check if camera is close enough to show text
  if (camera.position.distanceTo(targetCameraPosition) < 0.5) {
    showText.value = true;
  }

  renderer.render(scene, camera);
};

const onWindowResize = () => {
  if (camera && renderer) {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }
};

onMounted(() => {
  initThree();
});

onUnmounted(() => {
  window.removeEventListener("resize", onWindowResize);
  if (animationId) cancelAnimationFrame(animationId);
  if (renderer) renderer.dispose();
});
</script>

<style scoped>
/* Ensure the canvas is behind everything else if needed, though absolute positioning handles it */
</style>
