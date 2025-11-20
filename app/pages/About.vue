<template>
  <div class="min-h-screen bg-base-200 p-4 md:p-8">
    <div class="max-w-4xl mx-auto">
      <!-- Hero Section -->
      <div class="hero bg-base-100 rounded-box shadow-xl mb-8">
        <div class="hero-content text-center py-12">
          <div class="max-w-md">
            <h1 class="text-5xl font-bold text-primary">About PhysSim</h1>
            <p class="py-6 text-lg">
              Welcome to PhysSim, an interactive physics simulation platform
              designed to visualize and demonstrate complex physical concepts
              with ease.
            </p>
            <NuxtLink to="/" class="btn btn-primary">Start Simulation</NuxtLink>
          </div>
        </div>
      </div>

      <!-- Project Details -->
      <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
        <!-- Card 1: Purpose -->
        <div class="card bg-base-100 shadow-xl">
          <div class="card-body">
            <h2 class="card-title text-secondary">
              <i class="fa fa-bullseye"></i> Our Purpose
            </h2>
            <p>
              This project aims to bridge the gap between theoretical physics
              and visual understanding. By using real-time rendering and
              accurate physics engines, we bring equations to life.
            </p>
          </div>
        </div>

        <!-- Card 2: Tech Stack -->
        <div class="card bg-base-100 shadow-xl">
          <div class="card-body">
            <h2 class="card-title text-accent">
              <i class="fa fa-code"></i> Technology
            </h2>
            <p>
              Built with modern web technologies for performance and
              scalability:
            </p>
            <ul class="list-disc list-inside mt-2 opacity-80">
              <li>Nuxt 4 & Vue 3</li>
              <li>Three.js for 3D Rendering</li>
              <li>Cannon-es for Physics</li>
              <li>Daisy UI & Tailwind CSS</li>
            </ul>
          </div>
        </div>
      </div>

      <!-- Footer / Contact info -->
      <div class="mt-8 text-center opacity-50 text-sm">
        <p>Created as a demo project. &copy; 2025</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
useHead({
  title: "About - PhysSim",
  meta: [{ name: "description", content: "About the PhysSim project" }],
});
</script>
