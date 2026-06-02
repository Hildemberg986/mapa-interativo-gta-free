<script setup>
import mapaGta from './assets/mapa-gta.webp'
</script>

<template>
  <main class="home">
    <img
      class="home-image"
      :src="mapaGta"
      alt="Mapa interativo GTA"
    />
  </main>
</template>

<style scoped>
.home {
  min-height: 100vh;
  display: grid;
  place-items: center;
  margin: 0;
  padding: 1rem;
}

.home-image {
  width: 100%;
  max-width: 900px;
  height: auto;
  display: block;
}
</style>
