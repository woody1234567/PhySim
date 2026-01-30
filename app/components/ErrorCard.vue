<template>
  <div class="p-4">
    <div class="card bg-base-100 w-96 shadow-xl border border-base-300">
      <div
        class="text-8xl font-black text-center mt-10 text-primary opacity-20"
      >
        {{ error?.statusCode || "500" }}
      </div>
      <div class="card-body items-center text-center -mt-8">
        <h2 class="card-title text-2xl font-bold">
          {{ error?.message || "Unexpected Error" }}
        </h2>
        <p class="text-base-content/60 text-sm mb-4">
          An error occurred while trying to process your request.
        </p>
        <div class="card-actions">
          <nuxt-link class="btn btn-primary" to="/">Back to Home</nuxt-link>
          <button class="btn btn-ghost" @click="handleClearError">
            Clear Error
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
const props = defineProps({
  error: {
    type: Object,
    default: () => ({}),
  },
});

const handleClearError = () => clearError({ redirect: "/" });
</script>

<style scoped></style>
