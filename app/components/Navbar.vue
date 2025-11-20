<template>
  <div class="navbar bg-base-100 shadow-sm sticky top-0 z-30">
    <div class="navbar-start">
      <nuxt-link class="btn btn-ghost text-xl normal-case" to="/"
        >PhysSim</nuxt-link
      >
    </div>
    <div class="navbar-end hidden lg:flex">
      <ul class="menu menu-horizontal px-5 font-bold">
        <li><nuxt-link to="/about">About</nuxt-link></li>
        <li>
          <details>
            <summary>Chapter</summary>
            <ul class="p-2">
              <li>
                <nuxt-link class="btn" to="/Chapter/Incline">Incline</nuxt-link>
              </li>
              <li>
                <nuxt-link class="btn" to="/Chapter/Projectile"
                  >Projectile</nuxt-link
                >
              </li>
              <li>
                <nuxt-link class="btn" to="/Chapter/Pendulum"
                  >Pendulum</nuxt-link
                >
              </li>
              <li>
                <nuxt-link class="btn" to="/Chapter/ThreeBodySimulation"
                  >Three Body Simulation</nuxt-link
                >
              </li>
            </ul>
          </details>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup></script>

<style scoped></style>
