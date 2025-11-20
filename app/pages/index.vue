<template>
  <div>
    <Hero @scrollToSims="scrollToSims" />
  </div>
  <div ref="simsSection" class="flex justify-center py-16 bg-base-100">
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8 px-4">
      <SimCard
        v-for="sim in simulations"
        :key="sim.title"
        :title="sim.title"
        :description="sim.description"
        :image="sim.image"
        :to="sim.to"
      />
    </div>
  </div>
</template>

<script setup>
const simsSection = ref(null);

const scrollToSims = () => {
  simsSection.value?.scrollIntoView({ behavior: "smooth" });
};

const simulations = [
  {
    title: "Pendulum",
    description:
      "Explore the physics of a simple pendulum. Adjust length and gravity to see how the period changes.",
    image: "https://picsum.photos/seed/pendulum/400/250",
    to: "/Chapter/Pendulum",
  },
  {
    title: "Projectile Motion",
    description:
      "Analyze projectile motion and trajectories. Experiment with launch angle and initial velocity.",
    image: "https://picsum.photos/seed/projectile/400/250",
    to: "/Chapter/Projectile",
  },
  {
    title: "Incline Plane",
    description:
      "Study forces on an inclined plane. Observe how friction and angle affect the motion of a block.",
    image: "https://picsum.photos/seed/incline/400/250",
    to: "/Chapter/Incline",
  },
];
</script>

<style scoped></style>
