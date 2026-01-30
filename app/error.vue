<template>
  <div class="flex justify-center items-center min-h-screen bg-base-200">
    <ErrorCard :error="error" />
  </div>
</template>

<script setup>
import ErrorCard from "~/components/ErrorCard.vue";

const props = defineProps({
  error: {
    type: Object,
    default: () => ({ statusCode: 500, message: "Something went wrong" }),
  },
});
</script>

<style scoped></style>
