<template>
  <div
    class="card bg-base-100 w-96 shadow-xl hover:shadow-2xl transition-shadow duration-300"
  >
    <figure class="px-10 pt-10">
      <img
        :src="image"
        :alt="title"
        class="rounded-xl h-48 w-full object-cover"
      />
    </figure>
    <div class="card-body items-center text-center">
      <h2 class="card-title">{{ title }}</h2>
      <p>{{ description }}</p>
      <div class="card-actions mt-4">
        <NuxtLink :to="to" class="btn btn-primary">Try Simulation</NuxtLink>
      </div>
    </div>
  </div>
</template>

<script setup>
defineProps({
  title: {
    type: String,
    required: true,
  },
  description: {
    type: String,
    required: true,
  },
  image: {
    type: String,
    default:
      "https://img.daisyui.com/images/stock/photo-1606107557195-0e29a4b5b4aa.webp",
  },
  to: {
    type: String,
    required: true,
  },
});
</script>

<style scoped></style>
