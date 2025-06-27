<script setup lang="ts">
import { ref,watch, onMounted } from 'vue'
import Altimeter from './dashboard/Altimeter.vue'
import GPS from './devices/GPS.vue';
import Velosity from './devices/Velosity.vue';
import Angular from './devices/Angular.vue';
</script>

<template>
  <header>
    <div id="header">
      <h1>Neon App</h1>
    </div>
  </header>
  <!-- <AltimeterChart /> -->
<div class ="grid">
  <div>One</div>
  <div>Two</div>
  <Altimeter />
  <div>Four</div>
  <div>Five</div>
  <div>Six</div>
</div>
</template>

<style>

#header {
  background-color: #434447;
  color: white;
  text-align: center;
}

.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-auto-rows: 1fr 1fr;
}


</style>
