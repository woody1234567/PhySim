<template>
  <div>
    <hero></hero>
  </div>
  <div class="flex justify-center mt-10">
    <div class="grid grid-cols-3 gap-5">
      <sim-card />
      <sim-card />
      <sim-card />
    </div>
  </div>
</template>

<script setup>
import SimCard from "~/components/SimCard.vue";
import Hero from "~/components/Hero.vue";
</script>

<style scoped></style>
