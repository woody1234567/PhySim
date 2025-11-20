<template>
  <div class="navbar bg-base-100 shadow-sm">
    <div class="navbar-start">
      <nuxt-link class="btn btn-ghost text-xl normal-case" to="/"
        >PhysSim</nuxt-link
      >
    </div>
    <div class="navbar-end hidden lg:flex">
      <ul class="menu menu-horizontal px-5 font-bold">
        <li><nuxt-link to="/About">About</nuxt-link></li>
        <li>
          <details>
            <summary>Chapter</summary>
            <ul class="p-2">
              <li>
                <nuxt-link class="btn" to="/Chapter/Kinetics"
                  >Kinetics</nuxt-link
                >
              </li>
              <li>
                <nuxt-link class="btn" to="/Chapter/Dynamics"
                  >Dynamics</nuxt-link
                >
              </li>
              <li>
                <nuxt-link class="btn" to="/Chapter/Incline">Incline</nuxt-link>
              </li>
              <li>
                <nuxt-link class="btn" to="/Chapter/Incline_revised"
                  >Incline_revised</nuxt-link
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
