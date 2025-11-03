<template>
  <Navbar />
  <div>
    <slot />
  </div>
  <Footer />
</template>

<script setup>
import Navbar from "~/components/Navbar.vue";
import Footer from "~/components/footer.vue";
</script>

<style scoped></style>
