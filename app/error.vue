<template>
  <div class="flex justify-center items-center">
    <error-card :error="error" />
  </div>
</template>

<script setup>
import ErrorCard from "~/components/ErrorCard.vue";
defineProps(["error"]);
</script>

<style scoped></style>
