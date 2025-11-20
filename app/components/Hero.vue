<template>
  <div>
    <div class="hero bg-base-200 min-h-[60vh]">
      <div class="hero-content text-center">
        <div class="max-w-md">
          <h1 class="text-5xl font-bold">Welcome to PhysSim</h1>
          <p class="py-6">
            Explore the fascinating world of physics through our interactive
            simulations. Whether you're a student, teacher, or just curious,
            dive in and start experimenting!
          </p>
          <button class="btn btn-primary" @click="$emit('scrollToSims')">
            Explore Simulations
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup></script>

<style scoped></style>
