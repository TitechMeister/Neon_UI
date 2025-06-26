<script setup lang="ts">
import { ref,watch, onMounted } from 'vue'
import Altimeter from './devices/Altimeter.vue'
import GPS from './devices/GPS.vue';
import Velosity from './devices/Velosity.vue';
import Angular from './devices/Angular.vue';
</script>

<template>
  <h1>Neon</h1>
    <p>Welcome to the Neon app!</p>
    <p>Explore the features and enjoy your stay.</p>
    <div class="wrapper">
    <div><Altimeter /></div>
    <div><GPS /></div>
    <div><Velosity /></div>
    <div><Angular /></div>
    </div>
  <!-- <AltimeterChart /> -->
</template>

<style>
.wrapper {
  display: grid;
  grid-template-columns: 1fr 1fr;
  grid-auto-rows: 1fr 1fr;
  column-gap: 100px;
}
</style>
