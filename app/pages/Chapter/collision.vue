<template>
  <div>
    <PhysicsProjectile />
  </div>
</template>

<script setup lang="ts">
import PhysicsProjectile from "~/components/PhysicsProjectile.vue";

definePageMeta({
  title: "Collision Simulation",
  layout: "default",
});
</script>
