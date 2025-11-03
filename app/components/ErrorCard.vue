<template>
  <div>
    <div class="card bg-base-100 w-96 shadow-sm">
      <div class="text-8xl font-bold text-center mt-10">
        {{ error.statusCode }}
      </div>
      <div class="card-body items-center text-center">
        <div class="card-title text-2xl my-1">{{ error.message }}</div>
        <div class="card-actions">
          <nuxt-link class="btn btn-primary" to="/">Back Home</nuxt-link>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
const { error } = defineProps(["error"]);
</script>

<style scoped></style>
